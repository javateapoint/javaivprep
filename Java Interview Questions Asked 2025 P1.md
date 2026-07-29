# Java & Spring Boot Interview Questions Asked (2025 - Part 1)

A structured collection of real-world Java senior backend interview questions asked in 2025, complete with detailed answers, code snippets, SQL queries, and architectural explanations.

---

## 1. Difference between `@Component`, `@Service`, and `@Repository`

In Spring Framework, `@Component`, `@Service`, and `@Repository` are stereotype annotations used to mark classes as Spring-managed beans.

| Annotation | Primary Purpose | Key Features & Exceptions |
| :--- | :--- | :--- |
| **`@Component`** | Generic stereotype for any Spring-managed component. | Utility classes, custom helpers, or classes without explicit domain roles. |
| **`@Service`** | Specialized stereotype for Service Layer (Business Logic). | Holds business calculations, transaction boundaries (`@Transactional`), and workflow logic. |
| **`@Repository`** | Specialized stereotype for Data Access Object (DAO) / Persistence Layer. | Enables automatic exception translation: translates vendor-specific SQL exceptions into Spring’s `DataAccessException` hierarchy. |

---

## 2. Difference between `HashMap` and `ConcurrentHashMap`

| Feature | `HashMap` | `ConcurrentHashMap` |
| :--- | :--- | :--- |
| **Thread Safety** | Not thread-safe. Concurrent modifications lead to corruption or infinite loops (in legacy implementations). | Fully thread-safe for concurrent read and write operations. |
| **Locking Mechanism** | No locking. | **Lock-free reads** (using `volatile` reads) and **bucket-level granularity locking** (using `synchronized` or CAS on individual node heads). |
| **Null Keys / Values** | Allows 1 `null` key and multiple `null` values. | **Does NOT allow `null` keys or values** (throws `NullPointerException`). |
| **Performance** | High for single-threaded usage; zero synchronization overhead. | High concurrent throughput; multiple threads can update different buckets simultaneously. |

---

## 3. High-Level Design: Scalable Banking & Account Management System

### System Requirements
- Multi-client access (Web, Mobile Apps, ATM, Partner APIs).
- Concurrent account operations with strict minimum balance validation.
- Idempotency, high availability, and financial auditability.

### Core Architecture Components
1. **API Gateway & Rate Limiter**: Authenticates requests (OAuth2/JWT), limits rate per user/IP.
2. **Account Microservice**: Handles balance queries, account settings, minimum balance checks.
3. **Transaction Microservice**: Handles transfers, credits, debits. Uses distributed locking (Redisson) or optimistic locking on database rows (`@Version`).
4. **Audit & Logging**: Asynchronous audit logger backed by Kafka event stream.

```java
// Spring Boot Service with Optimistic Locking & Minimum Balance Guard
@Service
public class BankAccountService {

    private static final BigDecimal MIN_BALANCE = new BigDecimal("500.00");
    private final AccountRepository accountRepository;

    public BankAccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional
    public void withdraw(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId)
                .orElseThrow(() -> new AccountNotFoundException(accountId));

        BigDecimal newBalance = account.getBalance().subtract(amount);
        if (newBalance.compareTo(MIN_BALANCE) < 0) {
            throw new InsufficientBalanceException("Transaction violates minimum balance requirement of " + MIN_BALANCE);
        }

        account.setBalance(newBalance);
        accountRepository.save(account); // JPA automatically includes @Version check
    }
}
```

---

## 4. SQL Query: Department-Wise Average Salary

Given two tables: `employee` (`id`, `name`, `salary`, `department_id`) and `department` (`id`, `name`):

```sql
SELECT 
    d.name AS department_name,
    ROUND(AVG(e.salary), 2) AS avg_salary,
    COUNT(e.id) AS total_employees
FROM department d
JOIN employee e ON d.id = e.department_id
GROUP BY d.id, d.name
HAVING COUNT(e.id) > 0
ORDER BY avg_salary DESC;
```

---

## 5. System Architecture: Justifying a "60% Application Performance Improvement" During Monolith-to-Microservices Migration

### Key Steps Taken
1. **Bottleneck Identification**: Profiled monolithic JVM using async-profiler, VisualVM, and database slow-query logs. Identified DB connection pool contention and blocking synchronous external calls.
2. **Decomposition via Strangler Pattern**: Extracted non-critical read-heavy modules (e.g., Reporting, Product Search) into independent microservices.
3. **Caching & Async Offloading**: Introduced Redis cache for read-heavy endpoints and offloaded non-blocking events (notifications, audit logs) to Kafka.
4. **Database Decoupling**: Replaced single monolithic database shared locks with domain-isolated database schemas per microservice.

### How the 60% Metric Was Quantified
- Measured **p99 response latency** before migration (monolith p99 = 450ms) vs after migration (microservices p99 = 180ms): `((450 - 180) / 450) * 100 = 60% reduction in latency`.
- Measured **system throughput**: Increased overall HTTP requests per second (RPS) from 1,200 RPS to 3,000 RPS on identical infrastructure budget.

---

## 6. Apache Kafka: Consumer Groups & Consumer Rebalancing

### What is a Consumer Group?
A Consumer Group consists of multiple consumer instances sharing a single `group.id`. Kafka automatically assigns partitions of a topic to consumers in the group so that each partition is consumed by exactly one consumer within the group.

### What Happens When a Consumer Leaves/Crashes Mid-Message?
1. **Rebalance Trigger**: The Kafka Broker (Group Coordinator) detects missing heartbeats (exceeding `session.timeout.ms`) and triggers a **Consumer Group Rebalance**.
2. **Uncommitted Message Handling**:
   - If manual commit is enabled (`enable.auto.commit=false`) and the crashed consumer had fetched a batch but **not yet committed the offset**, the newly assigned consumer re-reads the uncommitted messages from the last committed offset.
   - **Result**: Potential duplicate processing (requires idempotent consumers).
   - If auto-commit was enabled (`enable.auto.commit=true`), offset might have auto-committed, causing message loss if processing crashed prior to completion. Best practice: use manual `ACK` mode.

---

## 7. Stream API Interview Coding Exercises

### Exercise 7.1: Count Word Occurrences
Given array: `{"paper", "pen", "paper", "pen", "pen"}`

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class WordCountExample {
    public static void main(String[] args) {
        String[] words = {"paper", "pen", "paper", "pen", "pen"};

        Map<String, Long> wordCounts = Arrays.stream(words)
                .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

        System.out.println(wordCounts); 
        // Output: {paper=2, pen=3}
    }
}
```

### Exercise 7.2: Count Character Occurrences from String
Given string: `"I love india"`

```java
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public class CharacterCountExample {
    public static void main(String[] args) {
        String input = "I love india";

        Map<Character, Long> charCounts = input.chars()
                .mapToObj(c -> (char) c)
                .filter(c -> !Character.isWhitespace(c)) // Exclude spaces
                .collect(Collectors.groupingBy(
                        Character::toLowerCase, // Case-insensitive
                        Collectors.counting()
                ));

        System.out.println(charCounts);
        // Output: {i=3, l=1, o=1, v=1, e=1, n=1, d=1, a=1}
    }
}
```

---

## 8. HTTP Methods: POST vs PUT vs PATCH

| HTTP Method | Semantics | Idempotent | Request Body |
| :--- | :--- | :--- | :--- |
| **`POST`** | Creates a new resource under a target collection or triggers processing. | **No** (Multiple identical calls create multiple resources). | Required |
| **`PUT`** | Completely **replaces** the target resource with the provided payload. | **Yes** (Calling 1 time or 10 times leaves resource in identical state). | Required |
| **`PATCH`** | Applies **partial updates** to an existing resource. | **No** (Can be idempotent if formulated as absolute field updates, but RFC 5789 permits non-idempotent operations like array appends). | Required |

---

## 9. How Spring Boot Auto-Configuration Works

1. **`@SpringBootApplication`**: Combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`.
2. **`AutoConfigurationImportSelector`**: Spring Boot scans imports from configuration index files:
   - Spring Boot 3+: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
   - Spring Boot 2.x (Legacy): `META-INF/spring.factories`
3. **Conditional Evaluation**: Candidate `@Configuration` classes are evaluated against conditional annotations:
   - `@ConditionalOnClass`: Enables bean creation only if specific class is on classpath.
   - `@ConditionalOnMissingBean`: Registers default bean only if user hasn't defined custom bean.
   - `@ConditionalOnProperty`: Activates configuration based on application properties.
