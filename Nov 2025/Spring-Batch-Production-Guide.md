# Spring Batch Master Guide: Production-Ready Implementation & Real-World Use Cases

> **Role:** Principal Java Architect with expertise in batch processing at scale  
> **Goal:** Provide a complete Spring Batch guide with real-world production examples, best practices, and practical implementation patterns used by companies today.

---

## 1. Introduction to Spring Batch

### What is Spring Batch?
Spring Batch is a lightweight, comprehensive framework designed for processing large volumes of records in batch mode (non-interactive, bulk data processing).

### Why Spring Batch Matters
*   **Production Reality:** Companies process millions of records daily:
    *   Banks: Daily transaction reconciliation.
    *   E-commerce: Inventory sync across warehouses.
    *   Fintech: End-of-day settlement and reporting.
    *   SaaS: Scheduled data cleanup, user analytics export.
*   **Built-in Resilience:** Retry, skip, restart capabilities out of the box.
*   **Scalability:** Handles 100M+ records efficiently with partitioning & parallel processing.

---

## 2. Spring Batch Core Concepts

### Key Components

| Component | Purpose | Real-World Parallel |
| :--- | :--- | :--- |
| **Job** | Top-level container for batch execution. | Scheduled task (e.g., "Daily Reconciliation"). |
| **Step** | Logical unit within a Job. | Read CSV, Process, Write to DB. |
| **ItemReader** | Reads data from source (CSV, DB, API). | Spring's `FlatFileItemReader`, `JdbcCursorItemReader`. |
| **ItemProcessor** | Transforms/validates each item. | Business logic (calculate totals, validate rules). |
| **ItemWriter** | Writes processed data to destination. | Database insert, file write, API call. |
| **JobRepository** | Metadata storage (job runs, step execution). | Spring Batch's internal H2 DB (or external DB). |
| **JobLauncher** | Starts job execution. | REST API endpoint or `@Scheduled` method. |
| **Partitioner** | Divides data into partitions for parallel execution. | Range-based, custom logic. |
| **TaskExecutor** | Thread pool for parallel processing. | `ThreadPoolTaskExecutor`. |

---

## 3. Spring Batch Interview Framework

### The "RASECAP" Decision Framework

Use this framework when designing batch jobs:

1.  **R - Requirements:** What data? What transformations?
2.  **A - Architecture:** Single-threaded vs. Multi-threaded vs. Partitioned?
3.  **S - Scaling:** How many records? Can we partition?
4.  **E - Error Handling:** Retry? Skip? Dead Letter Queue?
5.  **C - Chunking:** Chunk size? (10, 100, 1000, 10K records)?
6.  **A - Automation:** Scheduled? Triggered? REST endpoint?
7.  **P - Performance:** Bottleneck? Database indexing? Network calls?

---

## 4. Real-World Production Use Cases

### Use Case 1: Daily Invoice Processing (Banks/Fintech)

**Scenario:** Process 10M daily invoices. Validate, enrich with exchange rates, calculate fees, persist to data warehouse.

**Architecture:**
*   **Source:** SFTP CSV files (1GB+ daily).
*   **Processing:** Validate format, enrich with API lookups (rate service), calculate fees.
*   **Destination:** Data warehouse (PostgreSQL).
*   **Error Handling:** Reject invalid invoices, send notification.
*   **Scale:** 500K invoices/hour = ~139 QPS.

**Key Decisions:**
*   **Chunking:** 10K records per chunk (balance memory vs. DB round trips).
*   **Partitioning:** 8 partitions (for 8-core processor).
*   **Error Handling:** Skip 0.1% bad records, log to DLQ.
*   **Scheduling:** Daily at 1 AM (off-peak).

---

### Use Case 2: E-Commerce Inventory Sync (Retail Companies)

**Scenario:** Sync inventory from warehouse DB to public-facing API every 30 mins. 5M SKUs.

**Architecture:**
*   **Source:** MySQL warehouse DB (read-only replica).
*   **Processing:** Calculate available stock (on-hand - reserved), apply pricing rules.
*   **Destination:** Redis cache (for API) + Elasticsearch (for search).
*   **Trigger:** Every 30 mins via `@Scheduled`.

**Code Example (Spring Batch Config):**

```java
@Configuration
@EnableBatchProcessing
public class InventorySyncJobConfig {

    @Autowired
    private JobRepository jobRepository;

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Bean
    public Job inventorySyncJob() {
        return new JobBuilder("inventorySyncJob", jobRepository)
            .start(inventorySyncStep())
            .listener(new JobCompletionNotificationListener())
            .build();
    }

    @Bean
    public Step inventorySyncStep() {
        return new StepBuilder("inventorySyncStep", jobRepository)
            .<Sku, SkuInventory>chunk(5000)  // Process 5K SKUs per chunk
            .reader(skuReader())
            .processor(inventoryProcessor())
            .writer(cacheWriter())
            .faultTolerant()
            .skip(SkuProcessingException.class)  // Skip bad SKUs
            .skipLimit(100)  // Fail job if > 100 skip
            .retry(TemporaryApiException.class)  // Retry API calls
            .retryLimit(3)
            .taskExecutor(taskExecutor())  // Multi-threaded
            .build();
    }

    @Bean
    public ItemReader<Sku> skuReader() {
        return new JdbcCursorItemReaderBuilder<Sku>()
            .name("skuReader")
            .dataSource(dataSource)
            .sql("SELECT * FROM sku WHERE active = 1")
            .rowMapper(new SkuRowMapper())
            .fetchSize(5000)  // Read 5K at a time
            .build();
    }

    @Bean
    public ItemProcessor<Sku, SkuInventory> inventoryProcessor() {
        return sku -> {
            // Get inventory from cache (optimistic)
            Inventory inv = inventoryService.getInventory(sku.getId());
            
            // Apply pricing rules
            PriceRule rule = priceRuleService.get(sku.getCategoryId());
            double price = rule.apply(inv.getBaseCost());
            
            return new SkuInventory(sku.getId(), inv.getAvailable(), price);
        };
    }

    @Bean
    public ItemWriter<SkuInventory> cacheWriter() {
        return items -> {
            // Batch update Redis (pipeline for performance)
            items.forEach(item -> {
                redisTemplate.opsForValue()
                    .set("sku:" + item.getId(), item.toJson(), Duration.ofHours(2));
            });
            // Also index in Elasticsearch for search
            elasticsearchService.bulkIndex(items);
        };
    }

    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("batch-");
        executor.initialize();
        return executor;
    }

    @Scheduled(fixedRate = 30 * 60 * 1000)  // Every 30 mins
    public void runInventorySyncJob() throws Exception {
        JobParameters jobParams = new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        jobLauncher.run(inventorySyncJob(), jobParams);
    }
}
```

---

### Use Case 3: User Data Export (SaaS Platforms)

**Scenario:** Export 100K user profiles to CSV for analytics team every week. Sensitive data encryption.

**Architecture:**
*   **Source:** PostgreSQL user table.
*   **Processing:** Filter active users, encrypt PII, format for analytics.
*   **Destination:** S3 bucket (with server-side encryption).
*   **Notification:** Email when done.

**Code Example (Writer with Encryption):**

```java
@Bean
public ItemWriter<UserProfile> csvEncryptedWriter() {
    return new FlatFileItemWriterBuilder<UserProfile>()
        .name("userCsvWriter")
        .resource(new FileSystemResource("users_export.csv"))
        .lineAggregator(new DelimitedLineAggregator<UserProfile>() {
            {
                setDelimiter(",");
                setFieldExtractor(new BeanWrapperFieldExtractor<UserProfile>() {
                    {
                        setNames(new String[]{"id", "name", "email", "createdAt"});
                    }
                });
            }
        })
        .afterWrite(items -> {
            // Encrypt file before uploading to S3
            File file = new File("users_export.csv");
            encryptionService.encryptFile(file);
            s3Client.putObject("data-exports", "users_" + System.currentTimeMillis() + ".csv.enc");
        })
        .build();
}
```

---

## 5. Error Handling & Resilience Patterns (Production-Critical)

### Pattern 1: Retry with Exponential Backoff

Used for transient errors (network timeouts, temporary service unavailability).

```java
@Bean
public Step apiEnrichmentStep() {
    return new StepBuilder("apiEnrichmentStep", jobRepository)
        .<Record, EnrichedRecord>chunk(100)
        .reader(recordReader())
        .processor(new ItemProcessor<Record, EnrichedRecord>() {
            @Override
            public EnrichedRecord process(Record item) throws Exception {
                // Might fail due to network timeout
                return externalApiService.enrich(item);
            }
        })
        .writer(enrichedWriter())
        .faultTolerant()
        .retry(TimeoutException.class)
        .retryLimit(3)
        .listener(new StepExecutionListener() {
            @Override
            public void afterStep(StepExecution stepExecution) {
                // Log retry metrics
                if (stepExecution.getReadCount() > 0) {
                    logger.info("Processed: {}, Failed: {}", 
                        stepExecution.getReadCount(),
                        stepExecution.getFailureExceptions().size());
                }
            }
        })
        .build();
}
```

### Pattern 2: Skip with Dead Letter Queue (DLQ)

Invalid records are skipped and sent to a queue for manual investigation.

```java
@Bean
public Step validationStep() {
    return new StepBuilder("validationStep", jobRepository)
        .<RawData, ValidatedData>chunk(1000)
        .reader(csvReader())
        .processor(validationProcessor())
        .writer(validatedDataWriter())
        .faultTolerant()
        .skip(DataValidationException.class)
        .skipLimit(1000)  // Stop job if > 1000 records invalid
        .listener(new SkipListener<RawData, ValidatedData>() {
            @Override
            public void onSkipInWrite(ValidatedData item, Exception exception) {
                // Send to DLQ for manual review
                dlqService.sendToQueue("validation-dlq", item);
                metrics.incrementSkippedCount();
            }
        })
        .build();
}
```

### Pattern 3: Idempotency (Critical for Restartability)

Jobs can restart from failure point. Ensure "duplicate writes" don't cause issues.

```java
@Bean
public ItemWriter<Order> idempotentOrderWriter() {
    return items -> {
        items.forEach(order -> {
            // Use "upsert" instead of "insert" for idempotency
            String sql = "INSERT INTO orders (id, amount, status) VALUES (?, ?, ?) " +
                        "ON CONFLICT(id) DO UPDATE SET amount = EXCLUDED.amount, status = EXCLUDED.status";
            jdbcTemplate.update(sql, order.getId(), order.getAmount(), order.getStatus());
        });
    };
}
```

---

## 6. Performance Optimization Techniques

### Optimization 1: Chunk Size Tuning

*   **Too Small (100):** More DB transactions, overhead.
*   **Too Large (10,000):** More memory, potential OOM errors.
*   **Sweet Spot:** 1,000 - 5,000 for typical workloads.

```java
// For 10M records with 2000 chunk size:
// 10M / 2000 = 5000 chunks (commits)
// If each chunk takes 1 sec, total time = 5000 sec = ~1.4 hours

@Bean
public Step optimizedStep() {
    return new StepBuilder("optimizedStep", jobRepository)
        .<Item, Item>chunk(2000)  // Tuned chunk size
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .taskExecutor(taskExecutor())  // Parallel processing
        .build();
}
```

### Optimization 2: Partitioning for Parallel Processing

Divide work into independent partitions, process in parallel.

```java
@Bean
public Job partitionedJob() {
    return new JobBuilder("partitionedJob", jobRepository)
        .start(masterStep())
        .build();
}

@Bean
public Step masterStep() {
    return new StepBuilder("masterStep", jobRepository)
        .partitioner("workerStep", rangePartitioner())
        .step(workerStep())
        .gridSize(8)  // 8 parallel partitions
        .taskExecutor(taskExecutor())
        .build();
}

@Bean
public Partitioner rangePartitioner() {
    RangePartitioner partitioner = new RangePartitioner();
    partitioner.setGridSize(8);
    return partitioner;
}

@Bean
public Step workerStep() {
    return new StepBuilder("workerStep", jobRepository)
        .<Item, Item>chunk(1000)
        .reader(partitionedReader(null))
        .processor(processor())
        .writer(writer())
        .build();
}
```

### Optimization 3: Database Indexing

Batch reads are typically heavy. Ensure proper DB indexing.

```sql
-- Good: Indexed query for batch reader
CREATE INDEX idx_status_created ON orders(status, created_at DESC);
SELECT * FROM orders WHERE status = 'pending' AND created_at > ? ORDER BY created_at DESC;

-- Bad: Full table scan
SELECT * FROM orders WHERE amount > 1000;
```

---

## 7. Real-World Company Examples

### Example 1: Stripe (Payment Processing)

**Job:** Daily transaction settlement.
*   **Scale:** 100M+ transactions daily.
*   **Pattern:** Multi-step job → Validation → Reconciliation → Settlement → Reports.
*   **Error Handling:** Transactional integrity critical. Use DB locks, idempotency keys.

### Example 2: Airbnb (Listing Updates)

**Job:** Sync host availability/pricing every 15 mins.
*   **Scale:** 7M listings, multi-region.
*   **Pattern:** Partitioned by region → Read from cache → Update search index → Sync to API.
*   **Optimization:** Use Redis as intermediate cache to avoid direct DB writes.

### Example 3: Amazon (Inventory Management)

**Job:** Hourly warehouse inventory sync.
*   **Scale:** 500M+ SKUs across 100+ warehouses.
*   **Pattern:** Remote partitioning (run workers on separate JVMs in different regions).
*   **Data Consistency:** Eventual consistency model. Track changes via event log.

---

## 8. Monitoring & Observability (Production Must-Have)

### Metrics to Track

```java
@Component
public class BatchMetricsListener implements JobExecutionListener {
    
    @Autowired
    private MeterRegistry meterRegistry;

    @Override
    public void afterJob(JobExecution jobExecution) {
        long duration = jobExecution.getEndTime().getTime() - jobExecution.getStartTime().getTime();
        long readCount = jobExecution.getStepExecutions().stream()
            .mapToLong(StepExecution::getReadCount)
            .sum();
        long writeCount = jobExecution.getStepExecutions().stream()
            .mapToLong(StepExecution::getWriteCount)
            .sum();

        // Record metrics
        meterRegistry.timer("batch.job.duration", "job", jobExecution.getJobInstance().getJobName())
            .record(duration, TimeUnit.MILLISECONDS);
        meterRegistry.counter("batch.job.records.read", "job", jobExecution.getJobInstance().getJobName())
            .increment(readCount);
        meterRegistry.counter("batch.job.records.written", "job", jobExecution.getJobInstance().getJobName())
            .increment(writeCount);
        
        // Log to monitoring system
        logger.info("Job {} completed in {}ms. Read: {}, Written: {}", 
            jobExecution.getJobInstance().getJobName(), 
            duration, readCount, writeCount);
    }
}
```

---

## 9. Common Pitfalls (Interview Questions)

| Pitfall | Cause | Solution |
| :--- | :--- | :--- |
| Job restarts from beginning | No checkpoint mechanism | Use Spring Batch's built-in state management (JobRepository). |
| "Out of Memory" errors | Chunk size too large | Reduce chunk size or implement streaming reader. |
| Duplicate writes after restart | No idempotency | Use database upserts or external cache deduplication. |
| Slow batch jobs | No partitioning, sequential processing | Use `TaskExecutor` and `Partitioner` for parallel processing. |
| Data inconsistency | Missing error handling | Implement skip/retry policies, DLQ for failed records. |

---

## 10. Spring Batch Best Practices (Production Checklist)

1.  **Use JobRepository for Metadata:** Track job runs, step execution.
2.  **Implement Retry & Skip:** Handle transient and skip-able failures gracefully.
3.  **Partition Large Jobs:** Process data in parallel, not sequentially.
4.  **Monitor & Alert:** Track job duration, record counts, failure rates.
5.  **Idempotency:** Jobs must be safely restartable.
6.  **Transaction Boundaries:** Chunk size should match DB transaction limits.
7.  **Logging:** Log at critical points (start, end, errors).
8.  **Dead Letter Queue:** Invalid data → DLQ for investigation.

---

## 11. Spring Batch vs. Other Batch Frameworks

| Framework | Use Case | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **Spring Batch** | General-purpose batch jobs | Mature, reliable, Spring ecosystem | Verbose configuration |
| **Quartz** | Job scheduling | Simple cron-like scheduling | Not designed for bulk data processing |
| **Apache Spark** | Big data processing (100M+ records) | Distributed, fast | Overkill for simple transformations, requires clusters |
| **Apache Kafka** | Real-time streaming | Low-latency, high-throughput | Not for batch jobs, different paradigm |

---

## 12. Final Spring Batch Cheat Sheet

| Scenario | Solution |
| :--- | :--- |
| **Process 1M CSV rows** | Chunk size 5K, TaskExecutor with 8 threads. |
| **API enrichment with timeouts** | Retry with backoff + Circuit Breaker. |
| **Invalid records** | Skip + DLQ for manual review. |
| **Safe restarts** | Idempotent writes (upsert), JobRepository. |
| **Real-time monitoring** | Metrics listener, log job stats. |
| **Multi-region** | Remote partitioning with separate JVMs. |

---

**Document Version:** 1.0  
**Target Audience:** Senior Java Engineers, Architects, Backend Roles  
**Real-World Examples:** Based on patterns used by Stripe, Amazon, Airbnb