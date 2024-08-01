Let's start by creating the Spring Batch application without using JobBuilderFactory and StepBuilderFactory. We'll configure the job and steps directly and ensure it meets the requirements.

Project Structure
Ensure your project structure is as follows:

lua
Copy code
src
|-- main
    |-- java
        |-- com
            |-- example
                |-- batch
                    |-- config
                        |-- BatchConfig.java
                        |-- JobConfig.java
                    |-- job
                        |-- KycIndicatorItemProcessor.java
                        |-- KycIndicatorItemReader.java
                        |-- KycIndicatorItemWriter.java
                        |-- JobScheduler.java
                    |-- model
                        |-- KycIndicator.java
    |-- resources
        |-- application.properties
        |-- kyc_indicators.csv
1. Batch Configuration
Configure the batch setup with the necessary beans:

java
Copy code
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.repository.support.JobRepositoryFactoryBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean
    public JobRepository jobRepository(PlatformTransactionManager transactionManager) throws Exception {
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(dataSource);
        factory.setTransactionManager(transactionManager);
        factory.setIsolationLevelForCreate("ISOLATION_SERIALIZABLE");
        factory.setDatabaseType("ORACLE");
        factory.afterPropertiesSet();
        return factory.getObject();
    }
}
2. Job Configuration
Define the job and steps directly:

java
Copy code
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.core.step.tasklet.TaskletStep;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
import org.springframework.batch.item.file.mapping.DefaultLineMapper;
import org.springframework.batch.item.file.transform.DelimitedLineTokenizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.scheduling.annotation.Scheduled;

@Configuration
@EnableBatchProcessing
public class JobConfig {

    @Bean
    public Job importKycIndicatorsJob(JobRepository jobRepository, Step step1) {
        return new JobBuilder("importKycIndicatorsJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .start(step1)
            .build();
    }

    @Bean
    public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager, 
                      ItemReader<KycIndicator> reader, ItemProcessor<KycIndicator, KycIndicator> processor,
                      ItemWriter<KycIndicator> writer) {
        return new StepBuilder("step1", jobRepository)
            .<KycIndicator, KycIndicator>chunk(10, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
3. Reader, Processor, and Writer Configuration
Reader
java
Copy code
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
import org.springframework.batch.item.file.mapping.DefaultLineMapper;
import org.springframework.batch.item.file.transform.DelimitedLineTokenizer;
import org.springframework.context.annotation.Bean;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Component;

@Component
public class KycIndicatorItemReader {

    @Bean
    public FlatFileItemReader<KycIndicator> reader() {
        FlatFileItemReader<KycIndicator> reader = new FlatFileItemReader<>();
        reader.setResource(new ClassPathResource("kyc_indicators.csv"));
        reader.setLineMapper(new DefaultLineMapper<KycIndicator>() {{
            setLineTokenizer(new DelimitedLineTokenizer() {{
                setNames("relId", "kycStatus");
            }});
            setFieldSetMapper(new BeanWrapperFieldSetMapper<KycIndicator>() {{
                setTargetType(KycIndicator.class);
            }});
        }});
        return reader;
    }
}
Processor
java
Copy code
import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;

import java.util.Date;

@Component
public class KycIndicatorItemProcessor implements ItemProcessor<KycIndicator, KycIndicator> {

    @Override
    public KycIndicator process(KycIndicator kycIndicator) throws Exception {
        kycIndicator.setDCreat(new Date());
        kycIndicator.setDUpd(new Date());
        kycIndicator.setNKycStatusFileId(1L);  // Static value for dev purposes
        kycIndicator.setXCreat("system");      // Static value for dev purposes
        kycIndicator.setXUpd("system");        // Static value for dev purposes
        return kycIndicator;
    }
}
Writer
java
Copy code
import org.springframework.batch.item.database.BeanPropertyItemSqlParameterSourceProvider;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;

@Component
public class KycIndicatorItemWriter {

    @Bean
    public JdbcBatchItemWriter<KycIndicator> writer(DataSource dataSource) {
        JdbcBatchItemWriter<KycIndicator> writer = new JdbcBatchItemWriter<>();
        writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
        writer.setSql("MERGE INTO COPS_CUST_INDICATORS (REL_ID, F_KYC_STATUS, N_KYC_STATUS_FILE_ID, D_CREAT, D_UPD, X_CREAT, X_UPD) " +
                      "KEY (REL_ID) VALUES (:relId, :fKycStatus, :nKycStatusFileId, :dCreat, :dUpd, :xCreat, :xUpd)");
        writer.setDataSource(dataSource);
        return writer;
    }
}
4. Scheduling the Job
Configure the scheduler to run the job every 10 minutes:

java
Copy code
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class JobScheduler {

    private final JobLauncher jobLauncher;
    private final Job importKycIndicatorsJob;

    public JobScheduler(JobLauncher jobLauncher, Job importKycIndicatorsJob) {
        this.jobLauncher = jobLauncher;
        this.importKycIndicatorsJob = importKycIndicatorsJob;
    }

    @Scheduled(cron = "0 */10 * * * ?")
    public void runJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addLong("time", System.currentTimeMillis())
            .toJobParameters();
        jobLauncher.run(importKycIndicatorsJob, params);
    }
}
5. CSV File Model
Define the KycIndicator class to map the CSV data:

java
Copy code
import java.util.Date;

public class KycIndicator {

    private String relId;
    private String kycStatus;
    private Long nKycStatusFileId;
    private Date dCreat;
    private Date dUpd;
    private String xCreat;
    private String xUpd;

    // Getters and setters
}
Final Project Structure
Ensure your project structure looks like this:

lua
Copy code
src
|-- main
    |-- java
        |-- com
            |-- example
                |-- batch
                    |-- config
                        |-- BatchConfig.java
                        |-- JobConfig.java
                    |-- job
                        |-- KycIndicatorItemProcessor.java
                        |-- KycIndicatorItemReader.java
                        |-- KycIndicatorItemWriter.java
                        |-- JobScheduler.java
                    |-- model
                        |-- KycIndicator.java
    |-- resources
        |-- application.properties
        |-- kyc_indicators.csv
application.properties Configuration
Configure your database connection:

properties
Copy code
spring.datasource.url=jdbc:oracle:thin:@//your-db-host:1521/your-db-service
spring.datasource.username=your-db-username
spring.datasource.password=your-db-password
spring.batch.initialize-schema=always
With these changes, you should have a fully functional Spring Batch application that loads CSV files into the COPS_CUST_INDICATORS table, handling both insert and update operations, and running every 10 minutes.








