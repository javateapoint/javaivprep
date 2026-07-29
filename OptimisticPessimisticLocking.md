# Optimistic vs. Pessimistic Locking in Spring Boot Microservices

A high-concurrency guide on preventing race conditions, double-allocations, and oversells using JPA Optimistic Locking (`@Version`), JPA Pessimistic Locking (`@Lock`), and Distributed Locking (Redis / Redisson).

---

## 1. Real-World Scenario: Flash Sale Contention

Imagine a flash sale where only 1 unit of a product remains in inventory. Two independent purchase requests arrive simultaneously at different API instances:

- **Without Concurrency Control**: Both threads read `quantity = 1`, both compute `quantity - 1 = 0`, and both update the database. Inventory becomes `-1` (oversell).
- **With Locking**: Exactly one transaction succeeds while the other is handled gracefully via retry mechanisms or explicit out-of-stock failure.

---

## 2. Optimistic Locking ("Trust & Verify")

### Best Used When
- Read operations significantly outnumber write operations (Low to Moderate Contention).
- High throughput and non-blocking database connections are critical.
- Application logic can cleanly handle or retry on concurrency failures.

### Execution Flow
1. Thread A and Thread B read the record simultaneously: `quantity = 1, version = 42`.
2. Thread A updates: `SET quantity = 0, version = 43 WHERE sku = 'ECHO-DOT' AND version = 42`.
3. Thread B attempts to update: `SET quantity = 0, version = 43 WHERE sku = 'ECHO-DOT' AND version = 42`.
4. Database reports `0 rows updated` for Thread B. JPA throws `ObjectOptimisticLockingFailureException` (or `OptimisticLockException`).
5. Thread B catches the exception and retries: fresh read sees `quantity = 0` → throws `OutOfStockException`.

### Spring Data JPA Implementation

```java
package com.example.demo.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

@Entity
@Table(name = "inventory")
@Getter
@Setter
public class Inventory {

    @Id
    private String sku;

    private int quantity;

    @Version
    private Long version; // JPA automatically increments version on update
}
```

```java
package com.example.demo.service;

import com.example.demo.entity.Inventory;
import com.example.demo.exception.OutOfStockException;
import com.example.demo.repository.InventoryRepository;
import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class InventoryService {

    private final InventoryRepository repository;

    public InventoryService(InventoryRepository repository) {
        this.repository = repository;
    }

    @Retryable(
        retryFor = { ObjectOptimisticLockingFailureException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2.0)
    )
    @Transactional
    public void reserveOptimistic(String sku, int requestedQty) {
        Inventory inv = repository.findById(sku)
                .orElseThrow(() -> new IllegalArgumentException("SKU not found: " + sku));

        if (inv.getQuantity() < requestedQty) {
            throw new OutOfStockException("Insufficient stock for SKU: " + sku);
        }

        inv.setQuantity(inv.getQuantity() - requestedQty);
        repository.save(inv); // Includes version check in SQL WHERE clause
    }
}
```

---

## 3. Pessimistic Locking ("Lock & Load")

### Best Used When
- High contention on specific hot items (e.g., concert tickets, limited flash sales).
- The cost of retrying transactions is higher than waiting for a lock.
- Strict serializability is required at the database level.

### Execution Flow
1. Thread A executes `SELECT * FROM inventory WHERE sku = 'ECHO-DOT' FOR UPDATE`. Database locks row.
2. Thread B executes the same query but blocks immediately, waiting for Thread A's transaction to commit or rollback.
3. Thread A updates `quantity = 0` and commits transaction. Row lock is released.
4. Thread B acquires lock, reads updated `quantity = 0`, and aborts with `OutOfStockException`.

### Spring Data JPA Implementation

```java
package com.example.demo.repository;

import com.example.demo.entity.Inventory;
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface InventoryRepository extends JpaRepository<Inventory, String> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT i FROM Inventory i WHERE i.sku = :sku")
    Optional<Inventory> findBySkuForUpdate(@Param("sku") String sku);
}
```

```java
@Transactional
public void reservePessimistic(String sku, int requestedQty) {
    // Acquires DB row lock immediately
    Inventory inv = repository.findBySkuForUpdate(sku)
            .orElseThrow(() -> new IllegalArgumentException("SKU not found: " + sku));

    if (inv.getQuantity() < requestedQty) {
        throw new OutOfStockException("Insufficient stock for SKU: " + sku);
    }

    inv.setQuantity(inv.getQuantity() - requestedQty);
    // Commit releases row lock
}
```

---

## 4. Distributed Locking (Redis / Redisson)

In multi-region or microservice deployments, database-level locking can bottleneck database connection pools. Distributed locks using Redis (Redisson) offload lock management outside the database.

```java
@Service
public class DistributedInventoryService {

    private final RedissonClient redissonClient;
    private final InventoryRepository repository;

    public DistributedInventoryService(RedissonClient redissonClient, InventoryRepository repository) {
        this.redissonClient = redissonClient;
        this.repository = repository;
    }

    public void reserveDistributed(String sku, int qty) throws InterruptedException {
        RLock lock = redissonClient.getLock("lock:inventory:" + sku);
        
        // Wait up to 5 seconds to acquire lock; auto-unlock after 10 seconds
        if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
            try {
                Inventory inv = repository.findById(sku).orElseThrow();
                if (inv.getQuantity() < qty) {
                    throw new OutOfStockException("Out of stock");
                }
                inv.setQuantity(inv.getQuantity() - qty);
                repository.save(inv);
            } finally {
                lock.unlock();
            }
        } else {
            throw new ConcurrencyFailureException("Could not acquire lock for SKU: " + sku);
        }
    }
}
```

---

## 5. Architectural Comparison Matrix

| Feature | Optimistic Locking | Pessimistic Locking | Distributed Locking (Redis) |
| :--- | :--- | :--- | :--- |
| **Contention Level** | Low to Moderate | High | Very High / Multi-Service |
| **Database Overhead** | Minimal (Version column check) | High (Database connection held during lock) | Very Low (Offloaded to Redis) |
| **Throughput** | Maximum (Non-blocking) | Limited (Serial processing per key) | High (Fast in-memory locking) |
| **Retry Requirement** | Required (Application layer retry) | Not needed (Threads wait for lock) | Optional (Timeout based) |
| **Deadlock Risk** | None at database layer | Possible if multiple tables locked | Preventable with lock lease timeout |
