Let's go through the steps to develop the Spring Batch application to load files into the COPS_CUST_INDICATORS table as per your requirements. We'll configure the batch jobs to handle both insert and update operations and schedule them to run every 10 minutes. Here's a detailed plan for the development:

1. Set Up Spring Batch Dependencies
First, add the necessary dependencies to your pom.xml:

xml
Copy code
<dependencies>
    <!-- Spring Boot and Spring Batch dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.batch</groupId>
        <artifactId>spring-batch-core</artifactId>
    </dependency>
    
    <!-- Oracle JDBC driver -->
    <dependency>
        <groupId>com.oracle.database.jdbc</groupId>
        <artifactId>ojdbc8</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Other necessary dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
2. Configure the Batch Infrastructure
Create a configuration class to set up the job repository, transaction manager, and data source.

java
Copy code
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.repository.support.JobRepositoryFactoryBean;
import org.springframework.batch.core.configuration.annotation.DefaultBatchConfigurer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;

@Configuration
@EnableBatchProcessing
public class BatchConfig extends DefaultBatchConfigurer {

    @Autowired
    private DataSource dataSource;

    @Override
    public PlatformTransactionManager getTransactionManager() {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean
    public JobRepository jobRepository() throws Exception {
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(dataSource);
        factory.setTransactionManager(getTransactionManager());
        factory.setIsolationLevelForCreate("ISOLATION_SERIALIZABLE");
        factory.setDatabaseType("ORACLE");
        factory.afterPropertiesSet();
        return factory.getObject();
    }

    @Bean
    public JobBuilderFactory jobBuilderFactory(JobRepository jobRepository) {
        return new JobBuilderFactory(jobRepository);
    }

    @Bean
    public StepBuilderFactory stepBuilderFactory(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        return new StepBuilderFactory(jobRepository, transactionManager);
    }
}
3. Create the Batch Job
Define the job, steps, and tasklets.

java
Copy code
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JobConfig {

    @Bean
    public Job importKycIndicatorsJob(JobBuilderFactory jobBuilderFactory, Step step1) {
        return jobBuilderFactory.get("importKycIndicatorsJob")
            .incrementer(new RunIdIncrementer())
            .flow(step1)
            .end()
            .build();
    }

    @Bean
    public Step step1(StepBuilderFactory stepBuilderFactory, ItemReader<KycIndicator> reader,
                      ItemProcessor<KycIndicator, KycIndicator> processor, ItemWriter<KycIndicator> writer) {
        return stepBuilderFactory.get("step1")
            .<KycIndicator, KycIndicator>chunk(10)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
4. Develop the Reader, Processor, and Writer
Implement classes to read data from CSV files, process the data, and write it to the database.

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
        kycIndicator.setdCreat(new Date());
        kycIndicator.setdUpd(new Date());
        kycIndicator.setnKycStatusFileId(1L);  // Static value for dev purposes
        kycIndicator.setxCreat("system");      // Static value for dev purposes
        kycIndicator.setxUpd("system");        // Static value for dev purposes
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
5. Handle Insert and Update Operations
The SQL query in the writer uses the MERGE INTO statement to handle both insert and update operations. We will ensure that during the first insert, d_creat and d_upd will contain the date on which the file was first loaded, and during updates, d_upd should change to the current date, while d_creat remains the same.

6. Schedule the Batch Job
Configure a scheduler to run the job every 10 minutes.

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
7. Create CSV File Models
Define models to map the CSV file data.

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
8. Test the Application
Ensure that the application works as expected by running it and checking the database for the loaded data.

Final Code Structure
The final project structure should look like this:

lua
Copy code
src
|-- main
    |-- java
        |-- com
            |-- example
                |-- batch
                    |-- BatchConfig.java
                    |-- JobConfig.java
                    |-- JobScheduler.java
                    |-- KycIndicatorItemProcessor.java
                    |-- KycIndicatorItemReader.java
                    |-- KycIndicatorItemWriter.java
                    |-- model
                        |-- KycIndicator.java
    |-- resources
        |-- application.properties
        |-- kyc_indicators.csv
In application.properties, configure your database connection:

properties
Copy code
spring.datasource.url=jdbc:oracle:thin:@//your-db-host:1521/your-db-service
spring.datasource.username=your-db-username
spring.datasource.password=your-db-password
spring.batch.initialize-schema=always
With these steps, you should have a functional Spring Batch application that loads CSV files into the COPS_CUST_INDICATORS table, handling both insert and update operations and running every 10 minutes.
