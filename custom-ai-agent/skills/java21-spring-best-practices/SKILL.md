---
name: java21-spring-best-practices
description: >
  Java 21, Spring Boot 3.x, Spring Batch 5.x, and Microservices best practices reference skill.
  Ensures modern Java patterns, design principles, and token-optimized guidance.
---

# Java 21 & Spring Boot 3 Best Practices Skill

## Purpose
Provides concise technical guidelines for senior Java backend developers working on modern Spring Boot microservices and batch processing pipelines.

---

## 1. Java 21 Core Idioms

### A. Virtual Threads (Project Loom)
- Use for I/O-intensive microservice tasks and concurrent Spring Batch steps.
- Avoid pinning virtual threads: do not use `synchronized` blocks around blocking I/O; prefer `ReentrantLock`.
- Configure task executor:
  ```java
  @Bean
  public AsyncTaskExecutor applicationTaskExecutor() {
      return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
  }
  ```

### B. Records & Data Modeling
- Use `record` for DTOs, API requests/responses, and Spring Batch item transfer objects.
- Combine with record components validation (`Objects.requireNonNull`).

### C. Pattern Matching & Sealed Interfaces
- Model domain events and state machines using `sealed interface` and pattern matching `switch`:
  ```java
  public sealed interface PaymentStatus permits Pending, Completed, Failed {}

  public String process(PaymentStatus status) {
      return switch (status) {
          case Pending p   -> "Processing transaction " + p.txnId();
          case Completed c -> "Settled at " + c.timestamp();
          case Failed f    -> "Error: " + f.reason();
      };
  }
  ```

### D. Sequenced Collections
- Use `collection.getFirst()` and `collection.getLast()` over legacy index operations (`list.get(0)`).

---

## 2. Spring Boot 3.x & Spring Batch 5.x Standards

### A. Spring Batch 5.x Configuration
- **JobRepository Injection**: Always pass explicit `JobRepository` to `JobBuilder` and `StepBuilder`.
- **Chunk Step Idiom**:
  ```java
  @Bean
  public Step chunkStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
                        ItemReader<OrderRecord> reader, ItemProcessor<OrderRecord, ProcessedOrder> processor,
                        ItemWriter<ProcessedOrder> writer) {
      return new StepBuilder("orderProcessingStep", jobRepository)
          .<OrderRecord, ProcessedOrder>chunk(100, transactionManager)
          .reader(reader)
          .processor(processor)
          .writer(writer)
          .faultTolerant()
          .retryLimit(3)
          .retry(TransientDataAccessException.class)
          .build();
  }
  ```

### B. Microservices & Resilience Patterns
- **Resilience4j**: Wrap external RPC / REST calls with `@CircuitBreaker` and `@Retry`.
- **Observability**: Use Micrometer Tracing with W3C Trace Context headers (`traceparent`).
- **Error Handling**: Use `ProblemDetail` (RFC 7807) via `@RestControllerAdvice`.

---

## 3. Design Principles (SOLID & Clean Architecture)
1. **Single Responsibility**: Decouple controllers, business services, repository adapters, and batch processors.
2. **Interface Segregation**: Keep batch readers/writers narrow. Avoid fat service interfaces.
3. **Immutability First**: Default to unmodifiable collections (`List.of()`) and immutable records.
