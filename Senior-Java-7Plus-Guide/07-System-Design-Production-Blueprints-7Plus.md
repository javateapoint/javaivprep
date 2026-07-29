# Senior System Design Production Blueprints (7+ YOE)

Production-grade System Design blueprints tailored for Senior, Lead, and Principal Java Software Engineer interviews. Features complete architectural designs, component breakdowns, database schemas, sequence flows, and failure recovery modes for real-world enterprise systems.

---

## 🏗️ Blueprint 1: High-Throughput Idempotent Payment Processing Gateway

### 1. Functional & Non-Functional Requirements

- **Functional Requirements**:
  1. Process payments across multiple upstream payment gateways (Stripe, Adyen, PayPal).
  2. Guarantee **strict idempotency** (zero double-charging under network retries or client duplicate clicks).
  3. Track state transitions (`INITIATED` $\rightarrow$ `PROCESSING` $\rightarrow$ `SUCCESS` / `FAILED` / `REFUNDED`).
- **Non-Functional Requirements**:
  1. **Low Latency**: p99 response time $< 300\text{ms}$.
  2. **High Availability**: $99.999\%$ uptime (Multi-AZ active-active deployment).
  3. **Data Safety**: Financial auditability, zero data loss, eventual consistency via CDC.

---

### 2. High-Level System Architecture Diagram

```text
[ Client Mobile / Web ]
         │
         ├── 1. POST /api/v1/payments (Header: X-Idempotency-Key)
         ▼
[ API Gateway / Rate Limiter ] ──(Token Bucket per IP/User)
         │
         ▼
[ Payment Microservice Cluster (Spring Boot 3 + Virtual Threads) ]
         │
         ├── 2. Acquire Distributed Lock (Redisson) on "lock:payment:{idempotencyKey}"
         │
         ├── 3. Query Redis Cache / DB for existing Idempotency Key
         │         └── If Exists: Return Cached Response immediately!
         │
         ├── 4. Begin Local DB Transaction (@Transactional):
         │         ├── INSERT INTO payments (status = 'PROCESSING')
         │         └── INSERT INTO outbox_events (event = 'PAYMENT_INITIATED')
         │
         ├── 5. Call External Payment Gateway (Stripe/Adyen) via Resilience4j Circuit Breaker
         │
         ├── 6. Update Payment Status ('SUCCESS' / 'FAILED') & Release Redis Lock
         │
         ▼
[ PostgreSQL Primary-Replica Cluster ]
         │
         ▼ (Write-Ahead Log CDC via Debezium)
[ Apache Kafka Event Bus ] ──► [ Audit Service / Analytics / Reconciler ]
```

---

### 3. Database Schema & State Machine Design

```sql
-- Payment Transaction Ledger
CREATE TABLE payment_transactions (
    id UUID PRIMARY KEY,
    idempotency_key VARCHAR(128) UNIQUE NOT NULL,
    user_id BIGINT NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    gateway_name VARCHAR(50) NOT NULL,
    gateway_transaction_id VARCHAR(128),
    status VARCHAR(30) NOT NULL, -- INITIATED, PROCESSING, SUCCESS, FAILED, REFUNDED
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL,
    version BIGINT NOT NULL -- JPA Optimistic Locking Version
);

CREATE INDEX idx_payment_user_status ON payment_transactions (user_id, status);
CREATE INDEX idx_payment_created ON payment_transactions (created_at);
```

---

### 4. Production Java Idempotency & Payment Service

```java
package com.example.demo.payment;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.concurrent.TimeUnit;

@Service
public class PaymentProcessingService {

    private final RedissonClient redissonClient;
    private final PaymentRepository paymentRepository;
    private final PaymentGatewayClient gatewayClient;

    public PaymentProcessingService(RedissonClient redissonClient, 
                                    PaymentRepository paymentRepository, 
                                    PaymentGatewayClient gatewayClient) {
        this.redissonClient = redissonClient;
        this.paymentRepository = paymentRepository;
        this.gatewayClient = gatewayClient;
    }

    public PaymentResponse processPayment(String idempotencyKey, PaymentRequest request) throws Exception {
        String lockKey = "lock:payment:" + idempotencyKey;
        RLock lock = redissonClient.getLock(lockKey);

        // 1. Acquire Distributed Lock (Wait 3s, auto-lease 10s)
        if (!lock.tryLock(3, 10, TimeUnit.SECONDS)) {
            throw new PaymentConcurrencyException("Concurrent payment request already in progress for key: " + idempotencyKey);
        }

        try {
            // 2. Check DB for existing payment record with this idempotency key
            var existingPayment = paymentRepository.findByIdempotencyKey(idempotencyKey);
            if (existingPayment.isPresent()) {
                return PaymentResponse.fromEntity(existingPayment.get());
            }

            // 3. Initiate payment in local database (Status: PROCESSING)
            Payment payment = paymentRepository.save(new Payment(idempotencyKey, request));

            // 4. Call Upstream Payment Gateway
            GatewayResult result = gatewayClient.charge(request.getAmount(), request.getCurrency());

            // 5. Update status based on gateway result
            if (result.isSuccessful()) {
                payment.setStatus(PaymentStatus.SUCCESS);
                payment.setGatewayTransactionId(result.getGatewayTxId());
            } else {
                payment.setStatus(PaymentStatus.FAILED);
            }

            paymentRepository.save(payment);
            return PaymentResponse.fromEntity(payment);

        } finally {
            lock.unlock(); // Always release lock
        }
    }
}
```

---

## 🏗️ Blueprint 2: Scalable Multi-Channel Notification Engine

### 1. High-Level Requirements

- **Multi-Channel**: Supports Email (SES/SendGrid), SMS (Twilio), and Push Notifications (FCM/APNS).
- **Scale**: Handles **100 Million notifications per day** (~1,200 notifications/sec average, 5,000/sec peak).
- **Priority Queuing**: High-priority OTPs (One-Time Passwords) must be delivered within $< 5\text{s}$, while low-priority marketing emails are throttled.

---

### 2. Architectural Blueprint Diagram

```text
[ Client Services (Order, Auth, Fraud) ]
         │
         ├── POST /api/v1/notifications
         ▼
[ Notification Ingestion API Gateway ]
         │
         ▼
[ Priority Router & Validator ]
         │
         ├── High-Priority (OTPs, Security Alerts)  ──► [ Kafka Topic: priority-high (16 Partitions) ]
         │
         └── Low-Priority (Promotions, Digests)    ──► [ Kafka Topic: priority-low (8 Partitions) ]
                                                                 │
                                                                 ▼
[ Consumer Worker Cluster (Java 21 Virtual Threads) ] ◄──────────┘
         │
         ├── 1. Read Notification Template & Hydrate Placeholders (Thymeleaf/Handlebars)
         ├── 2. Check Recipient Rate Limits & Unsubscribe Preferences in Redis
         ├── 3. Execute Provider Call via Resilience4j Circuit Breaker (Twilio / SendGrid)
         │
         ├── SUCCESS ──► Update Notification Log Status to 'SENT'
         │
         └── FAILURE ──► Push to [ Dead Letter Queue (DLQ) ] for Retries with Backoff
```

---

### 3. Senior Interview Trade-Off Questions & Answers

#### Q1: How do you prevent sending duplicate SMS OTPs if Twilio times out?
- **Answer**: Pass a unique `providerMessageId` or hash of `(userId + templateId + timestamp_minute)` to the provider client. Use Redis `SETNX` with a 5-minute expiration window before invoking the Twilio API.

#### Q2: How do you handle a complete outage of the primary SMS provider (e.g. Twilio down)?
- **Answer**: Implement Resilience4j Fallback mechanism coupled with dynamic provider routing. When Twilio Circuit Breaker trips to `OPEN`, fallback method automatically reroutes traffic to secondary SMS provider (e.g. Infobip / MessageBird).

---

## 💡 Senior Interview Summary Checklist

1. **Always draw clear boundary layers**: API Gateway $\rightarrow$ Service Layer $\rightarrow$ Cache/Locking Layer $\rightarrow$ DB Layer $\rightarrow$ CDC/Messaging Layer.
2. **Emphasize Financial Idempotency**: Highlight Redisson distributed locking + database unique constraints.
3. **Emphasize Priority Partitioning**: Separate high-priority queues (OTPs) from low-priority queues (marketing emails) to prevent queue head-of-line blocking.
