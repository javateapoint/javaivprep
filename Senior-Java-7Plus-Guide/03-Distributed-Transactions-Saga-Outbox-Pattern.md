# Distributed Transactions, Saga Pattern & Transactional Outbox Master Guide

A production-grade reference guide for Senior Java Engineers (7+ YOE) covering **the Dual-Write problem**, **Transactional Outbox Pattern with Debezium CDC**, **Saga Pattern (Orchestration vs. Choreography)**, **compensating transactions**, **Idempotency Keys**, and **CQRS / Event Sourcing**.

---

## 1. The Dual-Write Problem in Microservices

In a microservices architecture, a single business operation frequently requires updating a local database AND publishing an event to a message broker (e.g. Apache Kafka).

```java
// BAD: Dual-Write Vulnerability!
@Transactional
public void createOrder(OrderRequest dto) {
    Order order = orderRepository.save(new Order(dto)); // 1. Local DB Write
    kafkaTemplate.send("order-created", new OrderCreatedEvent(order.getId())); // 2. Network I/O
}
```

### Failure Modes of Dual-Write
1. **Network Failure after DB Commit**: Order is saved to DB, but `kafkaTemplate.send()` fails due to network glitch or Kafka broker downtime. Event is lost; downstream microservices (Payment, Shipping) never process the order.
2. **Crash before DB Commit**: `kafkaTemplate.send()` succeeds asynchronously, but `orderRepository.save()` rolls back due to a DB constraint violation. Downstream services receive event for an order that does NOT exist in DB.

> **Key Rule**: Two-Phase Commit (2PC) / XA transactions across DB and message brokers are slow, blocking, non-scalable, and unsupported by modern cloud systems. **We must use eventual consistency patterns.**

---

## 2. Transactional Outbox Pattern with Change Data Capture (CDC)

### Architecture Flow

```text
[ Microservice Application ]
      │
      ├─► [ Local Transaction Boundary (@Transactional) ]
      │         ├── 1. INSERT INTO orders (...)
      │         └── 2. INSERT INTO outbox_events (...)  ◄── Same DB Transaction!
      │
      ▼
[ PostgreSQL / MySQL Database ]
      │
      ├─► Write-Ahead Log (WAL) / Binlog
      │
      ▼
[ Debezium CDC / Kafka Connect Engine ]
      │ (Reads WAL Asynchronously - Zero Impact on Application Threads)
      ▼
[ Apache Kafka Topics ]
```

### 2.1 Outbox Database Schema

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(255) NOT NULL,  -- e.g. "ORDER"
    aggregate_id VARCHAR(255) NOT NULL,    -- e.g. "ORDER-789"
    event_type VARCHAR(255) NOT NULL,      -- e.g. "ORDER_CREATED"
    payload JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL
);
```

### 2.2 Production Spring Boot Outbox Service

```java
package com.example.demo.outbox;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.UUID;

@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    public OrderApplicationService(OrderRepository orderRepository, OutboxRepository outboxRepository, ObjectMapper objectMapper) {
        this.orderRepository = orderRepository;
        this.outboxRepository = outboxRepository;
        this.objectMapper = objectMapper;
    }

    @Transactional
    public Order createOrderWithOutbox(OrderRequest dto) throws Exception {
        // 1. Save Domain Entity
        Order order = new Order(dto);
        Order savedOrder = orderRepository.save(order);

        // 2. Save Outbox Event in SAME Database Transaction!
        OrderCreatedEvent event = new OrderCreatedEvent(savedOrder.getId(), savedOrder.getAmount());
        
        OutboxEvent outbox = new OutboxEvent();
        outbox.setId(UUID.randomUUID());
        outbox.setAggregateType("ORDER");
        outbox.setAggregateId(savedOrder.getId().toString());
        outbox.setEventType("ORDER_CREATED");
        outbox.setPayload(objectMapper.writeValueAsString(event));
        outbox.setCreatedAt(Instant.now());

        outboxRepository.save(outbox); // Committed atomically with order entity!

        return savedOrder;
    }
}
```

---

## 3. Saga Pattern: Managing Long-Running Transactions

When a business transaction spans multiple microservices (e.g. Order Service $\rightarrow$ Payment Service $\rightarrow$ Inventory Service), a failure in any downstream service requires executing **Compensating Transactions** to undo previous steps.

### 3.1 Choreography vs. Orchestration

| Aspect | Choreography (Event-Driven) | Orchestration (Centralized) |
| :--- | :--- | :--- |
| **Control** | Decentralized; each service listens to events & emits new ones. | Centralized Saga Orchestrator controls state machine execution. |
| **Coupling** | Loose coupling. | Moderate coupling (Orchestrator knows service sequence). |
| **Complexity** | Difficult to trace & track global state for complex flows. | Easy to track global state; explicit workflow definition. |
| **Best Used For** | Simple workflows (2 to 3 microservices). | Complex workflows (4+ microservices with branching logic). |

### 3.2 Saga Orchestrator Flow & Compensating Actions

```text
[ Client ]
   │
   ▼
[ Order Saga Orchestrator ]
   │
   ├── 1. Reserve Credit (Payment Service) ───────► SUCCESS
   │
   ├── 2. Reserve Stock (Inventory Service) ──────► FAILURE (Out of Stock!)
   │
   └── 3. EXECUTE COMPENSATING ACTION:
            └─► Refund Credit (Payment Service) ──► COMPENSATED!
```

### Production Saga State Machine Interface

```java
package com.example.demo.saga;

public interface SagaStep<T> {
    void execute(T context);
    void compensate(T context); // Undo action executed on downstream failure!
}
```

```java
@Service
public class OrderSagaOrchestrator {

    private final List<SagaStep<OrderSagaContext>> steps;

    public OrderSagaOrchestrator(List<SagaStep<OrderSagaContext>> steps) {
        this.steps = steps;
    }

    public void executeSaga(OrderSagaContext context) {
        int executedSteps = 0;
        try {
            for (SagaStep<OrderSagaContext> step : steps) {
                step.execute(context);
                executedSteps++;
            }
        } catch (Exception e) {
            // Trigger Rollback / Compensation in REVERSE order!
            for (int i = executedSteps - 1; i >= 0; i--) {
                try {
                    steps.get(i).compensate(context);
                } catch (Exception rollbackException) {
                    // Log critical alert for manual operator intervention
                }
            }
            throw new SagaExecutionException("Saga failed and compensated", e);
        }
    }
}
```

---

## 4. Idempotency Keys in Distributed Systems

Network retries in message queues guarantee **at-least-once delivery**. Message consumers **MUST BE IDEMPOTENT** to prevent duplicate payments or duplicate orders.

### Idempotency Enforcement Pattern (Redis / DB)

```java
package com.example.demo.consumer;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class IdempotentPaymentConsumer {

    private final StringRedisTemplate redisTemplate;

    public IdempotentPaymentConsumer(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void processPaymentEvent(String idempotencyKey, PaymentEvent event) {
        String redisKey = "idempotency:payment:" + idempotencyKey;

        // Set key ONLY if absent (NX) with 24 hour TTL
        Boolean isFirstAttempt = redisTemplate.opsForValue()
                .setIfAbsent(redisKey, "PROCESSING", Duration.ofHours(24));

        if (Boolean.FALSE.equals(isFirstAttempt)) {
            // Duplicate message detected! Ignore execution safely.
            return;
        }

        try {
            // Execute actual payment processing logic
            executePayment(event);
            redisTemplate.opsForValue().set(redisKey, "COMPLETED", Duration.ofHours(24));
        } catch (Exception e) {
            // On failure, delete idempotency key to allow retry
            redisTemplate.delete(redisKey);
            throw e;
        }
    }

    private void executePayment(PaymentEvent event) { ... }
}
```

---

## 5. CQRS & Event Sourcing Essentials

- **Command Query Responsibility Segregation (CQRS)**: Separates the write model (optimizing commands, domain validation, and transactional invariants) from the read model (denormalized read tables / Elasticsearch optimized for fast querying).
- **Event Sourcing**: Instead of storing current state, store the append-only sequence of immutable events (`OrderCreated`, `ItemAdded`, `PaymentProcessed`). Current state is reconstructed by replaying events.

---

## 💡 Senior Interview Summary Tips

1. **Never use 2PC / XA transactions in microservices**; explain dual-write pitfalls and advocate for Transactional Outbox + CDC.
2. **Every Kafka consumer in a Saga architecture MUST be idempotent** using Idempotency Keys stored in Redis or DB unique constraints.
3. **Compensating transactions must be designed to never fail**; if compensation fails, emit an alert for manual operator intervention.
