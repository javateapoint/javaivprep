# Caveman Prompt Cheat Sheet for GitHub Copilot in IntelliJ

### Slash Token Costs by 60–80% for Senior Java Engineers

This cheat sheet provides copy-pasteable **Caveman Mode Prompts** for daily Java 21, Spring Boot 3.x, Spring Batch 5.x, and Microservice tasks.

---

## 1. Daily Development Caveman Prompts

| Goal | Standard Verbose Prompt (~35 Tokens) | Caveman Mode Prompt (~8 Tokens) | Token Savings |
|---|---|---|---|
| **Spring Batch Config** | *"How do I configure Spring Batch 5 chunk step with reader and writer in Java configuration?"* | `Spring Batch 5 chunk step config? Code only.` | **~77%** |
| **Virtual Thread Fix** | *"How do I fix virtual thread carrier thread pinning caused by synchronized keyword?"* | `Java 21 virtual thread pinning fix? ReentrantLock code.` | **~75%** |
| **Hibernate N+1 Query** | *"Why is Hibernate executing extra SQL queries for child entities and how do I fix it?"* | `Hibernate N+1 query fix? @EntityGraph snippet.` | **~75%** |
| **Spring Security JWT** | *"How do I set up stateless JWT authentication in Spring Security 6 without deprecated WebSecurityConfigurerAdapter?"* | `Spring Security 6 stateless JWT SecurityFilterChain code.` | **~75%** |
| **Flyway Migration** | *"How do I write a Flyway SQL migration script to add an index on customer_id?"* | `Flyway V2 add index SQL script example.` | **~75%** |
| **Resilience4j Retry** | *"How do I configure Resilience4j retry with exponential backoff in application.yml?"* | `Resilience4j retry exponential backoff application.yml snippet.` | **~70%** |
| **JUnit 5 Mockito** | *"Write a JUnit 5 parameterized test with Mockito for order processing service."* | `JUnit 5 @ParameterizedTest + Mockito code.` | **~75%** |

---

## 2. Universal Caveman Prompting Directives

Add any of these suffix lines to your prompts in Copilot Chat:

### Maximum Token Saver Suffix
> `Mode: Caveman. Zero fluff. Code + 1-line rule only.`

### Refactoring Directives
> `Refactor to Java 21 record. Diff only.`

### Code Review Directives
> `Review diff for N+1 queries & security bugs. Bullet points only.`

---

## 3. Example Caveman Interaction in Copilot Chat

### User Input:
> `@batch-microservice-specialist Spring Batch 5 KafkaItemReader config? Mode: Caveman. Code only.`

### Agent Response:
```java
// KafkaItemReader Spring Batch 5 setup
@Bean
public KafkaItemReader<String, OrderRecord> kafkaReader(ConsumerFactory<String, OrderRecord> factory) {
    Properties props = new Properties();
    props.putAll(factory.getConfigurationProperties());
    return new KafkaItemReaderBuilder<String, OrderRecord>()
        .name("kafkaOrderReader")
        .consumerProperties(props)
        .topic("order-events")
        .partitions(0)
        .build();
}
```
