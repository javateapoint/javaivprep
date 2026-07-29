# Database Tuning, HikariCP Connection Pooling & JPA Transaction Master Guide

A production-grade reference guide for Senior Java Engineers (7+ YOE) covering **HikariCP connection pool sizing & leak detection**, **Spring `@Transactional` propagation & isolation levels**, **N+1 query elimination**, **indexing strategies**, and reading PostgreSQL/MySQL **`EXPLAIN ANALYZE` execution plans**.

---

## 1. HikariCP Connection Pool Sizing & Leak Detection

### The Mathematical Formula for Connection Pool Sizing
A common misconception in backend engineering is that setting database connection pool size to a high number (e.g. 100 or 200 connections) increases throughput. In reality, excessive connections cause **CPU context switching, disk thrashing, and memory exhaustion**.

The official PostgreSQL / HikariCP benchmark formula for optimal pool size:

$$\text{Maximum Pool Size} = (\text{CPU Cores} \times 2) + \text{Effective Spindles}$$

- **`CPU Cores`**: Number of virtual CPU cores on the database server.
- **`Effective Spindles`**: Number of disk spindles (for SSDs or NVMe drives, `Effective Spindles` is typically treated as 1).

*Example*: For a database server with 8 CPU cores on an NVMe SSD:
$$\text{Optimal Connections} = (8 \times 2) + 1 = 17 \text{ connections}$$

> **Key Rule**: A small connection pool (e.g., 20 to 30 connections) with fast query execution easily handles 10,000+ HTTP requests per second via connection reuse.

### Production `application.yml` HikariCP Configuration

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 10
      idle-timeout: 300000        # 5 minutes
      max-lifetime: 1800000       # 30 minutes (must be < database connection timeout)
      connection-timeout: 30000   # 30 seconds wait for connection before failing
      leak-detection-threshold: 2000 # Log warning if thread holds connection > 2 seconds!
```

- **`leak-detection-threshold`**: Logs a stack trace if a thread holds a connection longer than specified (e.g., 2000ms). Essential for catching unclosed transactions or long-running HTTP calls inside transaction boundaries.

---

## 2. Spring `@Transactional` Propagation & Isolation Levels

### 2.1 Transaction Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
| :--- | :--- | :--- | :--- | :--- |
| **`READ_UNCOMMITTED`** | Allowed | Allowed | Allowed | Maximum |
| **`READ_COMMITTED`** (PostgreSQL Default) | Prevented | Allowed | Allowed | High |
| **`REPEATABLE_READ`** (MySQL Default) | Prevented | Prevented | Allowed | Moderate |
| **`SERIALIZABLE`** | Prevented | Prevented | Prevented | Lowest (Row/Range Locks) |

- **Dirty Read**: Transaction A reads uncommitted data written by Transaction B (B rolls back $\rightarrow$ stale data).
- **Non-Repeatable Read**: Transaction A reads a row twice and gets different column values because Transaction B modified and committed it in between.
- **Phantom Read**: Transaction A executes a range query twice and gets additional rows because Transaction B inserted and committed new matching rows.

### 2.2 Transaction Propagation Behaviors

```java
@Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.READ_COMMITTED)
public void outerMethod() { ... }
```

- **`REQUIRED`** (Default): Joins current active transaction if present; creates a new transaction if none exists.
- **`REQUIRES_NEW`**: Suspends current active transaction and opens a brand-new independent transaction. (Useful for audit logging or independent sub-tasks that must commit regardless of outer transaction rollback).
- **`MANDATORY`**: Must run inside an existing active transaction; throws `IllegalTransactionStateException` if none exists.
- **`SUPPORTS`**: Runs in transaction if caller has one; runs non-transactionally if caller has none.
- **`NOT_SUPPORTED`**: Suspends active transaction during method execution; resumes caller transaction afterwards.
- **`NEVER`**: Throws exception if active transaction exists.
- **`NESTED`**: Executes within a nested savepoint if supported by underlying DB vendor.

### 2.3 The Self-Invocation Proxy Trap

Spring `@Transactional` uses **AOP Proxies**. Calling `@Transactional` method within the **same class** bypasses the Spring Proxy!

```java
// BAD: Self-invocation bypasses transactional proxy!
@Service
public class OrderService {

    public void processOrder() {
        // Direct internal call does NOT trigger @Transactional aspect!
        this.saveAuditLog(); 
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog() {
        // Will run in SAME transaction as caller, ignoring REQUIRES_NEW!
    }
}
```

#### 🟢 Solution: Inject Self or Separate Dedicated Service

```java
// GOOD: Separate dedicated service ensures Spring proxy execution
@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog(AuditEntry entry) {
        auditRepository.save(entry); // Executes in brand-new independent transaction
    }
}
```

---

## 3. N+1 Query Problem & Production Solutions

### What is the N+1 Query Problem?
Occurs when fetching 1 parent entity triggers $N$ additional SQL queries to fetch lazy-loaded child entities.

```java
// BAD: Triggers 1 query for Orders + 100 queries for Customers = 101 queries!
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    System.out.println(order.getCustomer().getName()); // Triggers SQL query per iteration!
}
```

### Production Fixes

#### Solution 1: `JOIN FETCH` JPQL Query

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o JOIN FETCH o.customer JOIN FETCH o.lineItems WHERE o.status = :status")
    List<Order> findAllWithCustomerAndItems(@Param("status") OrderStatus status);
}
```

#### Solution 2: `@EntityGraph` Annotation

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"customer", "lineItems"})
    List<Order> findByStatus(OrderStatus status);
}
```

#### Solution 3: DTO Projections (Best Performance for Reads)

```java
// Avoids loading entire entity state into Hibernate L1 Cache altogether!
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("""
        SELECT new com.example.demo.dto.OrderSummaryDto(
            o.id, o.orderDate, c.name, c.email
        )
        FROM Order o JOIN o.customer c
        WHERE o.status = :status
    """)
    List<OrderSummaryDto> findOrderSummaries(@Param("status") OrderStatus status);
}
```

---

## 4. Indexing Strategies & `EXPLAIN ANALYZE` Reading

### 4.1 PostgreSQL Index Types

- **B-Tree Index** (Default): Optimal for equality (`=`), range queries (`<`, `>=`, `BETWEEN`), and sorting (`ORDER BY`).
- **Composite Index**: Index on multiple columns `(tenant_id, status, created_at)`.
  - **Leftmost Prefix Rule**: Index is used ONLY if queries include `tenant_id` in `WHERE` clause.
- **Partial Index**: Index filtered by a predicate:
  ```sql
  CREATE INDEX idx_unprocessed_orders ON orders (created_at) WHERE status = 'PENDING';
  ```
  *Saves memory by indexing only matching rows.*

### 4.2 Reading PostgreSQL `EXPLAIN ANALYZE` Execution Plans

Given query:
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 4500 AND status = 'PENDING';
```

Sample output:
```text
Bitmap Heap Scan on orders  (cost=4.30..15.80 rows=5 width=42) (actual time=0.045..0.052 rows=3 loops=1)
  Recheck Cond: (customer_id = 4500 AND status = 'PENDING'::text)
  ->  Bitmap Index Scan on idx_orders_customer_status  (cost=0.00..4.30 rows=5 width=0) (actual time=0.021..0.021 rows=3 loops=1)
        Index Cond: ((customer_id = 4500) AND (status = 'PENDING'::text))
Planning Time: 0.120 ms
Execution Time: 0.085 ms
```

- **`Seq Scan`**: Bad (Full table scan). Indicates missing index or non-selective query.
- **`Index Scan`**: Good (Navigates B-Tree index and reads table heap).
- **`Index Only Scan`**: Best (All required columns found in index without touching heap disk pages).
- **`cost=4.30..15.80`**: `4.30` is startup cost (time to return first row); `15.80` is total cost.
- **`actual time=0.045..0.052`**: Real execution time in milliseconds.

---

## 💡 Senior Interview Summary Tips

1. **Always calculate HikariCP pool size using the formula**, never guess arbitrary connection numbers.
2. **Set `hikari.leak-detection-threshold = 2000`** to automatically identify unclosed transactions or blocking network calls inside transactions.
3. **Use DTO Projections** for read-heavy API endpoints to bypass Hibernate L1 cache overhead and prevent N+1 queries.
