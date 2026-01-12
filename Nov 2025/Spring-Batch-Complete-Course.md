# Spring Batch Complete End-to-End Deep Dive Course

> **Author:** Principal Java Architect  
> **Level:** Beginner → Advanced  
> **Goal:** Master Spring Batch from fundamentals to production-ready implementations with crystal-clear explanations and practical coding examples.

---

## Table of Contents

1. [Introduction & Why Spring Batch?](#1-introduction--why-spring-batch)
2. [Core Concepts Explained Simply](#2-core-concepts-explained-simply)
3. [Setting Up Your First Spring Batch Project](#3-setting-up-your-first-spring-batch-project)
4. [The Three Heroes: Reader, Processor, Writer](#4-the-three-heroes-reader-processor-writer)
5. [Step & Job: Building Blocks](#5-step--job-building-blocks)
6. [Real-World Example 1: CSV to Database](#6-real-world-example-1-csv-to-database)
7. [Error Handling & Resilience](#7-error-handling--resilience)
8. [Listeners: Monitoring Your Jobs](#8-listeners-monitoring-your-jobs)
9. [Parallel Processing & Performance](#9-parallel-processing--performance)
10. [Real-World Example 2: Inventory Sync](#10-real-world-example-2-inventory-sync)
11. [Partitioning: Processing at Scale](#11-partitioning-processing-at-scale)
12. [JobRepository & Job Metadata](#12-jobrepository--job-metadata)
13. [Advanced Patterns](#13-advanced-patterns)
14. [Interview Questions & Answers](#14-interview-questions--answers)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. Introduction & Why Spring Batch?

### What is Spring Batch?

Imagine you need to process **1 million customer records** every night to:
- Send promotional emails
- Calculate monthly statements
- Update inventory
- Generate reports

You **can't** do this interactively (user-by-user). You need **batch processing**: large volumes of data, processed offline, in a structured way.

**Spring Batch** is a framework that makes this easy and reliable.

### Real-World Use Cases

| Company | Use Case | Scale |
| :--- | :--- | :--- |
| **Banks** | Daily transaction reconciliation | 10M+ transactions/day |
| **E-commerce (Amazon)** | Inventory sync across warehouses | 500M+ SKUs hourly |
| **Fintech (Stripe)** | Payment settlement & reporting | 100M+ transactions/day |
| **SaaS (Salesforce)** | Data export, cleanup, analytics | Scheduled batch jobs |
| **Telecom** | CDR (Call Detail Record) processing | Billions of records |

### Why Not Just Loop & Process?

```java
// ❌ BAD: No restartability, no monitoring, manual retry
for (Record record : getAllRecords()) {
    try {
        process(record);
        save(record);
    } catch (Exception e) {
        // Manual retry? Log where we stopped? Nightmare.
    }
}

// ✅ GOOD: Spring Batch handles all this automatically
```

### Key Benefits of Spring Batch

1. **Restartability:** If job fails at record 500K, restart from 500K (not from 1).
2. **Monitoring:** Built-in metrics (read count, write count, failure tracking).
3. **Error Handling:** Skip bad records, retry on transient failures, Dead Letter Queue.
4. **Scalability:** Process 1M records in parallel using partitioning.
5. **Transaction Management:** Chunk-based commits (10K records = 1 commit).

---

## 2. Core Concepts Explained Simply

### Concept 1: Job vs. Step vs. Chunk

Think of a **Restaurant Order**:

```text
JOB = Customer's entire order
    │
    ├─ STEP 1 = Prepare appetizers (Read CSV → Process → Write to DB)
    │   └─ CHUNK = 100 appetizers at a time (process 100, commit, repeat)
    │
    ├─ STEP 2 = Prepare main course
    │   └─ CHUNK = 100 main courses at a time
    │
    └─ STEP 3 = Pack order
        └─ CHUNK = 50 packages at a time
```

### Concept 2: The Three Components of a Step

Every **Step** has three parts (like an assembly line):

```text
INPUT (Reader) → PROCESSING (Processor) → OUTPUT (Writer)
    ↓                  ↓                      ↓
  CSV File       Transform Data         Database Insert
  Database       Validate               Elasticsearch
  API            Enrich                 S3 Upload
  Message Q      Filter                 HTTP API Call
```

### Concept 3: Chunk-Based Processing (Critical!)

Instead of loading all 1M records into memory:

```java
// Chunk-based = Process in batches
Chunk 1: Records 1-1000 → Process → Commit to DB
Chunk 2: Records 1001-2000 → Process → Commit to DB
Chunk 3: Records 2001-3000 → Process → Commit to DB
...
Total: 1000 chunks (commits) instead of 1 huge transaction

// Benefits:
// 1. Memory: Only 1000 records in RAM at a time (not 1M)
// 2. Restartability: If chunk 500 fails, retry from chunk 500
// 3. Performance: Database handles 1000-record inserts efficiently
```

### Concept 4: JobRepository (The Brain)

Spring Batch tracks every job run using **JobRepository**:

```text
Job: "DailyInvoiceProcess"
├─ Execution 1 (Jan 1, 2024): SUCCESS - 100K records
├─ Execution 2 (Jan 2, 2024): FAILED at record 50K
│   └─ Restart: Continue from record 50K
└─ Execution 3 (Jan 2, 2024 retry): SUCCESS - 100K records
```

Stored in database tables:
- `BATCH_JOB_INSTANCE`: Unique job definitions.
- `BATCH_JOB_EXECUTION`: Each run of a job.
- `BATCH_STEP_EXECUTION`: Each step within a run.

---

## 3. Setting Up Your First Spring Batch Project

### Step 1: Create a Spring Boot Project

Use **Spring Initializr** (start.spring.io):

**Dependencies to add:**
- Spring Batch
- Spring Data JPA
- MySQL Driver (or your DB)
- Lombok (optional, but recommended)

### Step 2: Project Structure

```text
spring-batch-course/
├── src/main/java/com/example/batch/
│   ├── config/
│   │   └── BatchConfiguration.java
│   ├── model/
│   │   └── Employee.java
│   ├── reader/
│   │   └── EmployeeReader.java
│   ├── processor/
│   │   └── EmployeeProcessor.java
│   ├── writer/
│   │   └── EmployeeWriter.java
│   ├── listener/
│   │   └── JobCompletionListener.java
│   └── SpringBatchCourseApplication.java
├── src/main/resources/
│   ├── application.properties
│   └── employees.csv
└── pom.xml
```

### Step 3: application.properties

```properties
# Database Configuration
spring.datasource.url=jdbc:mysql://localhost:3306/batch_db
spring.datasource.username=root
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Spring Batch Configuration
spring.batch.job.enabled=false  # Don't auto-run jobs at startup
spring.batch.jdbc.initialize-database=always  # Initialize batch tables

# Logging
logging.level.org.springframework.batch=DEBUG
```

### Step 4: Enable Spring Batch

```java
package com.example.batch;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class SpringBatchCourseApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBatchCourseApplication.class, args);
    }
}
```

---

## 4. The Three Heroes: Reader, Processor, Writer

### Hero 1: ItemReader (The Input Source)

**Purpose:** Read data one record at a time from any source.

#### Example 1: Read from CSV File

```java
package com.example.batch.reader;

import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.core.io.ClassPathResource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import com.example.batch.model.Employee;

@Configuration
public class EmployeeReaderConfig {

    @Bean
    public FlatFileItemReader<Employee> employeeReader() {
        return new FlatFileItemReaderBuilder<Employee>()
            .name("employeeReader")  // Unique name for this reader
            .resource(new ClassPathResource("employees.csv"))  // File path
            .delimited()  // CSV format
            .names("id", "firstName", "lastName", "email")  // Column names
            .targetType(Employee.class)  // Map to this class
            .build();
    }
}
```

**CSV File Example (employees.csv):**
```csv
id,firstName,lastName,email
1,John,Doe,john@example.com
2,Jane,Smith,jane@example.com
3,Bob,Johnson,bob@example.com
```

#### Example 2: Read from Database

```java
@Bean
public JdbcCursorItemReader<Employee> employeeDatabaseReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Employee>()
        .name("employeeDatabaseReader")
        .dataSource(dataSource)
        .sql("SELECT id, firstName, lastName, email FROM employees WHERE status = 'ACTIVE'")
        .rowMapper(new BeanPropertyRowMapper<>(Employee.class))  // Map SQL results to Employee
        .build();
}
```

#### Example 3: Read from JPA Query

```java
@Bean
public RepositoryItemReader<Employee> employeeRepositoryReader(EmployeeRepository repository) {
    return new RepositoryItemReaderBuilder<Employee>()
        .name("employeeRepositoryReader")
        .repository(repository)
        .methodName("findByStatusOrderByIdAsc")  // Method in repository
        .pageSize(100)  // How many per page
        .sorts(Collections.singletonMap("id", Sort.Direction.ASC))
        .build();
}
```

### Hero 2: ItemProcessor (The Transformer)

**Purpose:** Transform, validate, or enrich each record.

#### Simple Processor: Validation & Transformation

```java
package com.example.batch.processor;

import org.springframework.batch.item.ItemProcessor;
import org.springframework.stereotype.Component;
import com.example.batch.model.Employee;

@Component
public class EmployeeProcessor implements ItemProcessor<Employee, Employee> {

    @Override
    public Employee process(Employee employee) throws Exception {
        // Step 1: Validate
        if (employee.getFirstName() == null || employee.getFirstName().trim().isEmpty()) {
            System.out.println("❌ SKIP: Missing first name for ID " + employee.getId());
            return null;  // Return null = skip this record
        }

        // Step 2: Transform (business logic)
        String fullName = employee.getFirstName() + " " + employee.getLastName();
        employee.setFullName(fullName);
        employee.setEmail(employee.getEmail().toLowerCase());

        // Step 3: Enrich
        employee.setProcessedAt(System.currentTimeMillis());

        System.out.println("✅ PROCESSED: " + employee.getFullName());
        return employee;  // Return processed record
    }
}
```

**Important:** If you return `null`, the record is **skipped** (not written).

#### Complex Processor: External API Call

```java
@Component
public class EmployeeEnricherProcessor implements ItemProcessor<Employee, Employee> {

    @Autowired
    private ExternalAPIService apiService;

    @Override
    public Employee process(Employee employee) throws Exception {
        try {
            // Call external API to get department info
            Department dept = apiService.getDepartment(employee.getId());
            employee.setDepartment(dept);
            employee.setEnrichmentStatus("SUCCESS");
        } catch (TemporaryException e) {
            // Transient error - let Spring Batch retry this
            throw e;
        } catch (PermanentException e) {
            // Skip bad records
            employee.setEnrichmentStatus("FAILED");
            return null;
        }
        return employee;
    }
}
```

### Hero 3: ItemWriter (The Output Destination)

**Purpose:** Write processed records to a destination.

#### Example 1: Write to Database

```java
package com.example.batch.writer;

import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.JdbcBatchItemWriter;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.stereotype.Component;
import javax.sql.DataSource;
import com.example.batch.model.Employee;

@Component
public class EmployeeWriter {

    @Bean
    public JdbcBatchItemWriter<Employee> employeeWriter(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<Employee>()
            .name("employeeWriter")
            .dataSource(dataSource)
            .sql("INSERT INTO employee_processed (id, firstName, lastName, email, processedAt) " +
                 "VALUES (:id, :firstName, :lastName, :email, :processedAt)")
            .beanMapped()  // Use Employee's @Column-annotated fields
            .build();
    }
}
```

#### Example 2: Write to File

```java
@Bean
public FlatFileItemWriter<Employee> flatFileItemWriter() {
    return new FlatFileItemWriterBuilder<Employee>()
        .name("employeeFileWriter")
        .resource(new FileSystemResource("processed_employees.csv"))
        .delimited()
        .names("id", "firstName", "lastName", "email", "processedAt")
        .headerCallback(writer -> writer.write("ID,FirstName,LastName,Email,ProcessedAt"))
        .build();
}
```

#### Example 3: Write to Multiple Destinations (Composite Writer)

```java
@Bean
public CompositeItemWriter<Employee> compositeWriter(
        JdbcBatchItemWriter<Employee> dbWriter,
        FlatFileItemWriter<Employee> fileWriter) {

    List<ItemWriter<? super Employee>> writers = Arrays.asList(dbWriter, fileWriter);
    CompositeItemWriter<Employee> compositeWriter = new CompositeItemWriter<>();
    compositeWriter.setDelegates(writers);
    return compositeWriter;
}
```

---

## 5. Step & Job: Building Blocks

### Creating a Step (Assembling Reader → Processor → Writer)

```java
@Configuration
public class StepConfiguration {

    @Bean
    public Step employeeProcessingStep(
            StepBuilderFactory stepBuilderFactory,
            ItemReader<Employee> reader,
            ItemProcessor<Employee, Employee> processor,
            ItemWriter<Employee> writer) {

        return stepBuilderFactory.get("employeeProcessingStep")
            .<Employee, Employee>chunk(100)  // Process 100 records per chunk
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
```

**What happens:**
1. Reader reads first 100 records.
2. Processor processes each of the 100.
3. Writer writes all 100 to destination.
4. **Commit**: One database transaction (all 100 or none).
5. Repeat for next 100 records.

### Creating a Job (Orchestrating Multiple Steps)

```java
@Configuration
@EnableBatchProcessing
public class JobConfiguration {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Bean
    public Job employeeProcessingJob(
            Step employeeProcessingStep,
            JobCompletionListener jobCompletionListener) {

        return jobBuilderFactory.get("employeeProcessingJob")
            .incrementer(new RunIdIncrementer())  // Each run gets unique ID
            .listener(jobCompletionListener)  // Listen to job events
            .start(employeeProcessingStep)
            .build();
    }
}
```

### Job with Multiple Steps (Sequential Flow)

```java
@Bean
public Job multiStepJob(
        Step readDataStep,
        Step processDataStep,
        Step aggregateStep) {

    return jobBuilderFactory.get("multiStepJob")
        .incrementer(new RunIdIncrementer())
        .start(readDataStep)
        .next(processDataStep)
        .next(aggregateStep)
        .build();
}
```

---

## 6. Real-World Example 1: CSV to Database

### Scenario
Process an employee CSV file, validate records, and save to database.

### Step 1: Create Entity (Employee.java)

```java
package com.example.batch.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import javax.persistence.*;

@Entity
@Table(name = "employee_processed")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String firstName;
    private String lastName;
    private String email;
    private String fullName;
    private Long processedAt;
    private String status = "PENDING";
}
```

### Step 2: Create Reader (CSV Configuration)

```java
@Bean
public FlatFileItemReader<Employee> csvReader() {
    return new FlatFileItemReaderBuilder<Employee>()
        .name("csvReader")
        .resource(new ClassPathResource("employees.csv"))
        .delimited()
        .names("id", "firstName", "lastName", "email")
        .targetType(Employee.class)
        .linesToSkip(1)  // Skip header row
        .build();
}
```

### Step 3: Create Processor (Validate & Transform)

```java
@Component
public class EmployeeValidationProcessor implements ItemProcessor<Employee, Employee> {

    @Override
    public Employee process(Employee employee) throws Exception {
        // Validation Rule 1: FirstName is required
        if (employee.getFirstName() == null || employee.getFirstName().isEmpty()) {
            System.out.println("❌ REJECTED: Missing firstName for ID " + employee.getId());
            return null;  // Skip this record
        }

        // Validation Rule 2: Email format
        if (!employee.getEmail().contains("@")) {
            System.out.println("❌ REJECTED: Invalid email for ID " + employee.getId());
            return null;
        }

        // Transformation: Create full name
        employee.setFullName(employee.getFirstName() + " " + employee.getLastName());
        employee.setEmail(employee.getEmail().toLowerCase());
        employee.setProcessedAt(System.currentTimeMillis());
        employee.setStatus("PROCESSED");

        System.out.println("✅ ACCEPTED: " + employee.getFullName());
        return employee;
    }
}
```

### Step 4: Create Writer (Database)

```java
@Bean
public JdbcBatchItemWriter<Employee> databaseWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Employee>()
        .name("databaseWriter")
        .dataSource(dataSource)
        .sql("INSERT INTO employee_processed (firstName, lastName, email, fullName, processedAt, status) " +
             "VALUES (:firstName, :lastName, :email, :fullName, :processedAt, :status)")
        .beanMapped()
        .build();
}
```

### Step 5: Assemble Step & Job

```java
@Configuration
public class CsvToDatabaseJobConfig {

    @Bean
    public Step csvProcessingStep(
            StepBuilderFactory stepBuilderFactory,
            FlatFileItemReader<Employee> csvReader,
            EmployeeValidationProcessor processor,
            JdbcBatchItemWriter<Employee> writer) {

        return stepBuilderFactory.get("csvProcessingStep")
            .<Employee, Employee>chunk(100)  // Process 100 at a time
            .reader(csvReader)
            .processor(processor)
            .writer(writer)
            .build();
    }

    @Bean
    public Job csvToDatabaseJob(
            JobBuilderFactory jobBuilderFactory,
            Step csvProcessingStep) {

        return jobBuilderFactory.get("csvToDatabaseJob")
            .incrementer(new RunIdIncrementer())
            .start(csvProcessingStep)
            .build();
    }
}
```

### Step 6: Run the Job

```java
@Autowired
private JobLauncher jobLauncher;

@Autowired
private Job csvToDatabaseJob;

@GetMapping("/api/batch/run")
public ResponseEntity<String> runBatchJob() throws Exception {
    JobParameters jobParams = new JobParametersBuilder()
        .addLong("timestamp", System.currentTimeMillis())
        .toJobParameters();

    JobExecution jobExecution = jobLauncher.run(csvToDatabaseJob, jobParams);
    
    return ResponseEntity.ok("Job started: " + jobExecution.getJobInstance().getJobName() +
                            " Status: " + jobExecution.getStatus());
}
```

---

## 7. Error Handling & Resilience

### Error Scenario 1: Skip Bad Records

When a validation fails, skip the record and continue.

```java
@Bean
public Step stepWithSkip(
        StepBuilderFactory stepBuilderFactory,
        ItemReader<Employee> reader,
        ItemProcessor<Employee, Employee> processor,
        ItemWriter<Employee> writer) {

    return stepBuilderFactory.get("stepWithSkip")
        .<Employee, Employee>chunk(100)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .faultTolerant()  // Enable fault tolerance
        .skip(ValidationException.class)  // Skip if this exception occurs
        .skipLimit(10)  // But fail job if > 10 skips
        .listener(new SkipListener<Employee, Employee>() {
            @Override
            public void onSkipInProcess(Employee item, Exception exception) {
                System.out.println("⚠️ SKIPPED PROCESS: " + item.getId() + " Reason: " + exception.getMessage());
            }

            @Override
            public void onSkipInWrite(Employee item, Exception exception) {
                System.out.println("⚠️ SKIPPED WRITE: " + item.getId() + " Reason: " + exception.getMessage());
            }
        })
        .build();
}
```

### Error Scenario 2: Retry on Transient Failures

If an API call times out, retry automatically.

```java
@Bean
public Step stepWithRetry(
        StepBuilderFactory stepBuilderFactory,
        ItemReader<Employee> reader,
        ItemProcessor<Employee, Employee> processor,
        ItemWriter<Employee> writer) {

    return stepBuilderFactory.get("stepWithRetry")
        .<Employee, Employee>chunk(100)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .faultTolerant()
        .retry(TemporaryApiException.class)  // Retry on this exception
        .retryLimit(3)  // Max 3 retries
        .build();
}
```

**How Retry Works:**
```text
Record 1: Read → Process → FAIL (API timeout) → RETRY 1 → RETRY 2 → RETRY 3 → SKIP
Record 2: Read → Process → SUCCESS → Write
Record 3: Read → Process → SUCCESS → Write
...

If job fails: Restart from Record 1 (Spring Batch remembers progress)
```

### Error Scenario 3: Dead Letter Queue (DLQ)

Send failed records to a separate table for manual review.

```java
@Component
public class DeadLetterProcessor implements ItemProcessor<Employee, Employee> {

    @Autowired
    private DeadLetterRepository dlqRepository;

    @Override
    public Employee process(Employee employee) throws Exception {
        try {
            // Process normally
            validateAndProcess(employee);
            return employee;
        } catch (Exception e) {
            // Send to DLQ
            DeadLetterRecord dlRecord = new DeadLetterRecord(
                employee.getId(),
                "VALIDATION_FAILED",
                e.getMessage(),
                System.currentTimeMillis()
            );
            dlqRepository.save(dlRecord);
            return null;  // Skip from normal processing
        }
    }

    private void validateAndProcess(Employee employee) throws Exception {
        // Validation logic
    }
}
```

---

## 8. Listeners: Monitoring Your Jobs

### Job Listener (Before & After Job)

```java
@Component
public class JobCompletionListener extends JobExecutionListenerSupport {

    private static final Logger logger = LoggerFactory.getLogger(JobCompletionListener.class);

    @Override
    public void beforeJob(JobExecution jobExecution) {
        logger.info("🚀 JOB STARTED: {} at {}", 
            jobExecution.getJobInstance().getJobName(),
            jobExecution.getStartTime());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            logger.info("✅ JOB COMPLETED SUCCESSFULLY in {}ms",
                jobExecution.getEndTime().getTime() - jobExecution.getStartTime().getTime());

            // Log statistics
            jobExecution.getStepExecutions().forEach(stepExecution -> {
                logger.info("  Step: {} | Read: {} | Write: {} | Skip: {}",
                    stepExecution.getStepName(),
                    stepExecution.getReadCount(),
                    stepExecution.getWriteCount(),
                    stepExecution.getSkipCount());
            });

            // Do something after successful job (e.g., send email)
            notifyJobCompletion(jobExecution);
        } else if (jobExecution.getStatus() == BatchStatus.FAILED) {
            logger.error("❌ JOB FAILED");
            jobExecution.getAllFailureExceptions().forEach(ex ->
                logger.error("Exception: ", ex)
            );
        }
    }

    private void notifyJobCompletion(JobExecution jobExecution) {
        // Send email, update metrics, etc.
    }
}
```

### Step Listener (Before & After Step)

```java
@Component
public class StepCompletionListener extends StepExecutionListenerSupport {

    private static final Logger logger = LoggerFactory.getLogger(StepCompletionListener.class);

    @Override
    public void beforeStep(StepExecution stepExecution) {
        logger.info("📍 STEP STARTED: {}", stepExecution.getStepName());
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        logger.info("Step {} completed. Read: {}, Write: {}, Skip: {}",
            stepExecution.getStepName(),
            stepExecution.getReadCount(),
            stepExecution.getWriteCount(),
            stepExecution.getSkipCount());

        return stepExecution.getExitStatus();
    }
}
```

### Attaching Listeners to Steps

```java
@Bean
public Step employeeProcessingStep(
        StepBuilderFactory stepBuilderFactory,
        ItemReader<Employee> reader,
        ItemProcessor<Employee, Employee> processor,
        ItemWriter<Employee> writer,
        StepCompletionListener stepListener) {

    return stepBuilderFactory.get("employeeProcessingStep")
        .<Employee, Employee>chunk(100)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .listener(stepListener)  // Attach listener here
        .build();
}
```

---

## 9. Parallel Processing & Performance

### Why Parallel Processing?

**Single-threaded:** Process 1M records sequentially = 1 hour.  
**Multi-threaded (8 threads):** Process 1M records in parallel = ~7.5 minutes.

### Basic Multi-Threading with TaskExecutor

```java
@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);      // 8 threads
    executor.setMaxPoolSize(16);       // Max 16 threads
    executor.setQueueCapacity(100);    // Queue up to 100 tasks
    executor.setThreadNamePrefix("batch-");
    executor.initialize();
    return executor;
}

@Bean
public Step parallelStep(
        StepBuilderFactory stepBuilderFactory,
        ItemReader<Employee> reader,
        ItemProcessor<Employee, Employee> processor,
        ItemWriter<Employee> writer,
        TaskExecutor taskExecutor) {

    return stepBuilderFactory.get("parallelStep")
        .<Employee, Employee>chunk(100)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .taskExecutor(taskExecutor)  // Use thread pool
        .build();
}
```

**Caution:** Thread-safety! If your processor accesses shared resources, use synchronized blocks or thread-safe classes.

---

## 10. Real-World Example 2: Inventory Sync

### Scenario
Sync 5M SKUs from warehouse DB to cache every 30 mins. Apply pricing rules, enrich with stock levels.

### Step 1: Model Classes

```java
@Entity
@Data
public class Sku {
    @Id
    private String skuId;
    private String name;
    private Double baseCost;
    private String categoryId;
}

@Data
public class SkuInventory {
    private String skuId;
    private String name;
    private Integer availableStock;
    private Double retailPrice;
    private Long lastSyncTime;
}
```

### Step 2: Multi-Step Job

```java
@Configuration
public class InventorySyncJobConfig {

    @Bean
    public Step readSkuStep(
            StepBuilderFactory stepBuilderFactory,
            RepositoryItemReader<Sku> skuReader) {
        return stepBuilderFactory.get("readSkuStep")
            .tasklet((contribution, chunkContext) -> {
                // Just read, don't process yet
                return RepeatStatus.FINISHED;
            })
            .build();
    }

    @Bean
    public Step enrichAndPricingStep(
            StepBuilderFactory stepBuilderFactory,
            RepositoryItemReader<Sku> skuReader,
            SkuEnrichmentProcessor processor,
            CacheAndSearchWriter writer,
            TaskExecutor taskExecutor) {

        return stepBuilderFactory.get("enrichAndPricingStep")
            .<Sku, SkuInventory>chunk(5000)  // Process 5K SKUs per chunk
            .reader(skuReader)
            .processor(processor)
            .writer(writer)
            .taskExecutor(taskExecutor)  // Parallel
            .build();
    }

    @Bean
    public Job inventorySyncJob(
            JobBuilderFactory jobBuilderFactory,
            Step enrichAndPricingStep,
            JobCompletionListener jobListener) {

        return jobBuilderFactory.get("inventorySyncJob")
            .incrementer(new RunIdIncrementer())
            .listener(jobListener)
            .start(enrichAndPricingStep)
            .build();
    }

    // Schedule to run every 30 mins
    @Scheduled(fixedRate = 30 * 60 * 1000)
    public void scheduleInventorySyncJob() throws Exception {
        JobParameters jobParams = new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        jobLauncher.run(inventorySyncJob, jobParams);
    }
}
```

### Step 3: Processor with External Service Calls

```java
@Component
public class SkuEnrichmentProcessor implements ItemProcessor<Sku, SkuInventory> {

    @Autowired
    private PricingService pricingService;

    @Autowired
    private StockService stockService;

    @Override
    public SkuInventory process(Sku sku) throws Exception {
        try {
            // Get stock level
            Integer stock = stockService.getAvailableStock(sku.getSkuId());

            // Apply pricing rule
            PriceRule rule = pricingService.getPricingRule(sku.getCategoryId());
            Double retailPrice = rule.calculate(sku.getBaseCost());

            // Build enriched object
            SkuInventory inventory = new SkuInventory();
            inventory.setSkuId(sku.getSkuId());
            inventory.setName(sku.getName());
            inventory.setAvailableStock(stock);
            inventory.setRetailPrice(retailPrice);
            inventory.setLastSyncTime(System.currentTimeMillis());

            return inventory;
        } catch (TemporaryServiceException e) {
            throw e;  // Retry
        } catch (Exception e) {
            logger.error("Failed to enrich SKU {}", sku.getSkuId(), e);
            return null;  // Skip
        }
    }
}
```

### Step 4: Writer (Dual-Write: Cache + Search)

```java
@Component
public class CacheAndSearchWriter implements ItemWriter<SkuInventory> {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;

    @Override
    public void write(List<? extends SkuInventory> items) throws Exception {
        // Batch write to Redis (using pipeline for performance)
        RedisSerializer<String> serializer = redisTemplate.getStringSerializer();
        
        items.forEach(inventory -> {
            String key = "sku:" + inventory.getSkuId();
            String value = toJson(inventory);
            redisTemplate.opsForValue().set(key, value, Duration.ofHours(2));
        });

        // Bulk index in Elasticsearch
        elasticsearchTemplate.save(new ArrayList<>(items));
    }

    private String toJson(SkuInventory inventory) {
        // Use Jackson ObjectMapper
        return new ObjectMapper().writeValueAsString(inventory);
    }
}
```

---

## 11. Partitioning: Processing at Scale

### When to Use Partitioning?

- **Single-threaded:** ~1M records/hour
- **Multi-threaded (8 threads):** ~8M records/hour
- **Partitioned (8 partitions × 8 threads):** ~64M records/hour

### How Partitioning Works

```text
MASTER STEP (Partitioner)
    │
    ├─ WORKER STEP 1: Process SKUs 1-625K
    ├─ WORKER STEP 2: Process SKUs 625K-1.25M
    ├─ WORKER STEP 3: Process SKUs 1.25M-1.875M
    └─ ... (8 workers processing in parallel)

When all workers complete → Job completes
```

### Implementation: Range Partitioner

```java
@Component
public class SkuRangePartitioner implements Partitioner {

    @Autowired
    private SkuRepository skuRepository;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Map<String, ExecutionContext> partitions = new HashMap<>();

        // Get total count
        long totalRecords = skuRepository.count();
        long recordsPerPartition = (totalRecords / gridSize) + 1;

        // Create partitions
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext context = new ExecutionContext();
            long minId = i * recordsPerPartition;
            long maxId = (i + 1) * recordsPerPartition;

            context.putLong("minId", minId);
            context.putLong("maxId", maxId);

            partitions.put("partition-" + i, context);
        }

        return partitions;
    }
}
```

### Master-Worker Configuration

```java
@Configuration
public class PartitionedJobConfig {

    @Bean
    public Step masterStep(
            StepBuilderFactory stepBuilderFactory,
            Step workerStep,
            SkuRangePartitioner partitioner,
            TaskExecutor taskExecutor) {

        return stepBuilderFactory.get("masterStep")
            .partitioner("workerStep", partitioner)  // Partition the work
            .step(workerStep)  // Worker step
            .gridSize(8)  // 8 partitions
            .taskExecutor(taskExecutor)  // Use thread pool
            .build();
    }

    @Bean
    public Step workerStep(
            StepBuilderFactory stepBuilderFactory,
            ItemReader<Sku> partitionedReader,
            SkuEnrichmentProcessor processor,
            CacheAndSearchWriter writer) {

        return stepBuilderFactory.get("workerStep")
            .<Sku, SkuInventory>chunk(5000)
            .reader(partitionedReader)
            .processor(processor)
            .writer(writer)
            .build();
    }

    @Bean
    public Job partitionedInventorySyncJob(
            JobBuilderFactory jobBuilderFactory,
            Step masterStep) {

        return jobBuilderFactory.get("partitionedInventorySyncJob")
            .incrementer(new RunIdIncrementer())
            .start(masterStep)
            .build();
    }
}
```

### Partitioned Reader (Using ExecutionContext)

```java
@Bean
@StepScope  // Important: creates new instance per step
public RepositoryItemReader<Sku> partitionedSkuReader(
        SkuRepository repository,
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {

    return new RepositoryItemReaderBuilder<Sku>()
        .name("partitionedSkuReader")
        .repository(repository)
        .methodName("findBySkuIdBetween")  // Custom method
        .arguments(Arrays.asList(minId, maxId))  // Pass min/max
        .pageSize(5000)
        .sorts(Collections.singletonMap("skuId", Sort.Direction.ASC))
        .build();
}
```

---

## 12. JobRepository & Job Metadata

### Understanding JobRepository

Spring Batch automatically creates tables to track job execution:

```sql
-- Job metadata tables (auto-created)
BATCH_JOB_INSTANCE        -- Unique job names & parameters
BATCH_JOB_EXECUTION       -- Each run (success/failed)
BATCH_STEP_EXECUTION      -- Each step within a run
BATCH_JOB_EXECUTION_CONTEXT  -- Job state (for restart)
BATCH_STEP_EXECUTION_CONTEXT -- Step state (for restart)
```

### Querying Job History

```java
@RestController
public class BatchMonitoringController {

    @Autowired
    private JobExplorer jobExplorer;

    @GetMapping("/api/batch/jobs")
    public List<JobInstance> getAllJobs() {
        return jobExplorer.getJobInstances("inventorySyncJob", 0, 10);
    }

    @GetMapping("/api/batch/job/{jobInstanceId}")
    public JobInstance getJobDetails(@PathVariable Long jobInstanceId) {
        return jobExplorer.getJobInstance(jobInstanceId);
    }

    @GetMapping("/api/batch/execution/{jobExecutionId}")
    public JobExecution getExecutionDetails(@PathVariable Long jobExecutionId) {
        return jobExplorer.getJobExecution(jobExecutionId);
    }
}
```

---

## 13. Advanced Patterns

### Pattern 1: Conditional Flow (If-Else in Jobs)

```java
@Bean
public Job conditionalJob(
        JobBuilderFactory jobBuilderFactory,
        Step step1,
        Step step2,
        Step step3) {

    return jobBuilderFactory.get("conditionalJob")
        .incrementer(new RunIdIncrementer())
        .start(step1)
        .on("COMPLETED")  // If step1 succeeds
        .to(step2)  // Go to step2
        .from(step1)
        .on("FAILED")  // If step1 fails
        .to(step3)  // Go to step3
        .end()
        .build();
}
```

### Pattern 2: Job Chaining (Dependent Jobs)

```java
@Component
public class JobChainController {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job readDataJob;

    @Autowired
    private Job processDataJob;

    @Autowired
    private Job reportJob;

    @GetMapping("/api/batch/run-pipeline")
    public ResponseEntity<String> runJobPipeline() throws Exception {
        // Run Job 1
        JobExecution exec1 = jobLauncher.run(readDataJob, newJobParameters());
        if (exec1.getStatus() != BatchStatus.COMPLETED) {
            return ResponseEntity.status(500).body("Job 1 failed");
        }

        // Run Job 2
        JobExecution exec2 = jobLauncher.run(processDataJob, newJobParameters());
        if (exec2.getStatus() != BatchStatus.COMPLETED) {
            return ResponseEntity.status(500).body("Job 2 failed");
        }

        // Run Job 3
        JobExecution exec3 = jobLauncher.run(reportJob, newJobParameters());

        return ResponseEntity.ok("Pipeline complete");
    }

    private JobParameters newJobParameters() {
        return new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
    }
}
```

---

## 14. Interview Questions & Answers

### Q1: What happens if a step fails mid-way? How does Spring Batch handle restart?

**Answer:**
Spring Batch stores execution state in `JobRepository`. When you restart:
1. It checks the `BATCH_STEP_EXECUTION_CONTEXT`.
2. Finds the last committed chunk (e.g., records 1-5000).
3. Restarts from chunk 5001 (not from beginning).
4. This is automatic, no code needed (use `chunk()`-based steps).

```
First run:  Chunk 1-5K (OK) → Chunk 5K-10K (OK) → Chunk 10K-15K (FAIL at 12K)
Restart:    Chunk 10K-15K (RETRY) → Chunk 15K-20K (OK) → ...
```

### Q2: When should I use `@StepScope` vs `@JobScope`?

**Answer:**
- **@StepScope:** Create new instance for each step execution. Use for:
  - Partitioned readers/writers (need partition-specific parameters).
  - Step-specific state.
  
- **@JobScope:** Create new instance for each job execution. Use for:
  - Job-level parameters (date, environment).

```java
@Bean
@StepScope  // New instance per step execution
public ItemReader<Data> stepScopedReader(
        @Value("#{stepExecutionContext['minId']}") Long minId) {
    // minId varies per partition
}

@Bean
@JobScope  // New instance per job execution
public ItemReader<Data> jobScopedReader(
        @Value("#{jobParameters['date']}") String date) {
    // date is same for all steps in this job
}
```

### Q3: How do you handle idempotency in Spring Batch?

**Answer:**
If a job restarts, writes might occur twice. Handle with:
1. **Upsert (Insert or Update):** Database-level.
2. **Deduplication Cache:** Check Redis before writing.
3. **Idempotency Key:** Use unique constraint.

```java
// Option 1: Database Upsert
String sql = "INSERT INTO employees (id, name) VALUES (?, ?) " +
             "ON DUPLICATE KEY UPDATE name = VALUES(name)";

// Option 2: Redis deduplication
if (!redisCache.exists("processed:" + employee.getId())) {
    write(employee);
    redisCache.set("processed:" + employee.getId(), "1");
}
```

### Q4: What's the difference between `chunk` and `tasklet` steps?

**Answer:**

| Aspect | Chunk | Tasklet |
| :--- | :--- | :--- |
| **Purpose** | Bulk data processing (Reader → Processor → Writer) | Single operation (custom logic) |
| **Restartability** | Automatic (from last committed chunk) | Manual (you handle state) |
| **Use Case** | CSV import, data migration | Cleanup, report generation |
| **Example** | Process 1M records | Delete old files |

```java
// Chunk-based (recommended for bulk)
.<Employee, Employee>chunk(1000)
    .reader(reader)
    .processor(processor)
    .writer(writer)

// Tasklet-based (for simple operations)
.tasklet((contribution, chunkContext) -> {
    System.out.println("Doing cleanup...");
    // Your logic here
    return RepeatStatus.FINISHED;
})
```

---

## 15. Cheat Sheet

### Quick Reference

| Need | Solution |
| :--- | :--- |
| **Read CSV** | `FlatFileItemReaderBuilder` + `delimited()` |
| **Read DB** | `JdbcCursorItemReaderBuilder` or `RepositoryItemReader` |
| **Transform Data** | Implement `ItemProcessor` |
| **Write to DB** | `JdbcBatchItemWriter` or `RepositoryItemWriter` |
| **Write to File** | `FlatFileItemWriter` |
| **Skip Bad Records** | `.faultTolerant().skip(...).skipLimit(...)` |
| **Retry on Error** | `.faultTolerant().retry(...).retryLimit(...)` |
| **Process in Parallel** | Add `.taskExecutor(taskExecutor())` to step |
| **Process at Scale** | Use `.partitioner()` + Master-Worker pattern |
| **Monitor Job** | Implement `JobExecutionListener` & `StepExecutionListener` |
| **Schedule Job** | `@Scheduled(fixedRate = ...)` + `jobLauncher.run()` |

### Common Mistakes (Avoid These!)

```java
// ❌ WRONG: Processing all records in memory
List<Employee> allRecords = repository.findAll();  // Loads everything!
for (Employee emp : allRecords) { process(emp); }

// ✅ RIGHT: Spring Batch handles chunking automatically
.<Employee, Employee>chunk(1000)
    .reader(reader)  // Reader handles pagination
    .processor(processor)
    .writer(writer)

// ❌ WRONG: No error handling
.reader(reader).processor(processor).writer(writer)

// ✅ RIGHT: Handle errors gracefully
.faultTolerant()
    .skip(ValidationException.class)
    .skipLimit(100)
    .retry(TemporaryException.class)
    .retryLimit(3)

// ❌ WRONG: Shared mutable state in parallel processing
AtomicInteger counter = new AtomicInteger();
.taskExecutor(executor)  // Multi-threaded, but counter = race condition

// ✅ RIGHT: Thread-safe or no shared state
.taskExecutor(executor)  // Only use if processor/writer is thread-safe
```

---

## Summary: The Complete Flow

```text
1. CREATE JOB
   ├─ Define Reader (CSV/DB/API)
   ├─ Define Processor (validate/transform)
   ├─ Define Writer (DB/File/Cache)
   └─ Assemble into Step(s)

2. CONFIGURE STEP
   ├─ Set chunk size (100-5000)
   ├─ Add error handling (skip/retry)
   ├─ Add listeners (monitoring)
   └─ Optional: Add TaskExecutor (parallel)

3. ASSEMBLE JOB
   ├─ Combine steps (sequential or conditional)
   ├─ Add job listener
   └─ Set incrementer (unique job runs)

4. LAUNCH JOB
   ├─ Create JobParameters
   ├─ Call jobLauncher.run()
   └─ Check JobExecution status

5. MONITOR
   ├─ Check JobRepository tables
   ├─ View execution logs
   ├─ Handle failures → Restart
   └─ Analyze metrics
```

---

**Document Version:** 1.0  
**Difficulty:** Beginner → Advanced  
**Time to Master:** 2-3 weeks of consistent practice  
**Next Steps:** Build your own batch job following this guide!