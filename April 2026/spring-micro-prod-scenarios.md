# 15 Real-World Spring Boot & Microservices Incidents – A Practical Playbook

_A production-focused handbook for Java/Spring Boot engineers preparing for scenario-based interviews and real on-call life._

This book walks through 15 common production incidents one by one. For each incident you will get:
- A realistic **story** from a digital banking / payments system.
- A **step-by-step investigation plan** (what to check first, second, third).
- Likely **root causes** and **fixes**.
- How to **explain your answer in an interview** like a senior engineer.

We assume you are working with:
- Java 17+, Spring Boot 3.
- Kubernetes, Docker.
- Observability stack (Actuator, Micrometer, Prometheus, ELK/EFK, tracing).

---

## 0. Mindset for Scenario-Based Questions

Before jumping into the scenarios, understand what interviewers are testing:
- Can you **stay calm** and think systematically?
- Do you know how to use **metrics, logs, and traces** together?[web:39][web:42]
- Can you **balance quick mitigation and long-term fix**?

A simple structure that works for almost all questions:
1. **Clarify the context** – what changed recently, what environment, any error messages.
2. **Stabilize the system** – rate-limit, roll back, scale up, feature-toggle.
3. **Investigate with data** – metrics, logs, traces, thread dumps, DB checks.
4. **Fix and verify** – patch, redeploy, run regression checks.
5. **Prevent recurrence** – tests, alerts, runbooks, documentation.

In interviews, always walk through this structure aloud.

---

## 1. CPU Spikes to 90% in Production

> _“Your Spring Boot service CPU suddenly spikes to 90% in production. How will you investigate and fix it?”_

### 1.1 Realistic Scenario

You maintain `statement-service` that generates monthly statements. Suddenly, during normal business hours, alerts fire: CPU for the service pods goes above 90%, and response times increase.

### 1.2 Immediate Stabilization

1. **Check monitoring dashboards** (Prometheus/Grafana, Dynatrace, New Relic).[web:41][web:42]
   - Confirm if it’s one pod or all.
   - Look at CPU, memory, GC pause time, request rate, error rate.
2. If requests are high, **temporarily scale out** pods to handle load.
3. If batch work is running at peak hours, **pause non-critical jobs**.

### 1.3 Deep Investigation

1. **Identify hotspots**:
   - Use `/actuator/metrics` and `/actuator/threaddump` or JDK tools (jstack, jfr) to see which threads use most CPU.[web:39][web:42]
   - Many threads from a single pool? A suspicious scheduled task?
2. **Common causes**:
   - Infinite or very large loops (bad `while` conditions or missing breaks).[web:38]
   - Expensive operations in `equals/hashCode` or `toString` called repeatedly.
   - Excessive logging in tight loops.
   - Hot paths doing heavy in-memory sorting or JSON serialization.
3. **Correlate with code**:
   - Use stack traces from hottest threads to map to packages/classes.
   - Check recent commits touching those areas.

### 1.4 Example Root Cause and Fix

**Root cause:** A newly deployed feature calculates all historical statements in a single request, running a heavy aggregation over millions of records without pagination.

**Fixes:**
- Change implementation to **process in pages** (e.g., 1000 rows per batch).
- Offload heavy computation to an **async job** triggered by a command, storing results in a table for later download.
- Add **load tests** for large data volumes.

### 1.5 How to Answer in Interview

Structure your answer:
- “First I will confirm from **metrics** which instance and which time window the spike started.[web:42]
- Next, I’ll capture **thread dumps** and use them to locate the hot methods.[web:39]
- Parallelly, I’ll **mitigate** by scaling out and pausing heavy background jobs.
- Once I identify heavy loops or queries, I’ll **optimize** them using batching or pre-computation.
- Finally, I’ll add **alerts and load tests** so we catch similar patterns before production.”

---

## 2. Intermittent 500 Errors After Deployment

> _“After deployment, your service starts throwing intermittent 500 errors. How will you debug this issue?”_

### 2.1 Scenario

After deploying a new version of `payment-service`, the error rate jumps from 0.1% to 3%. Only some requests fail with 500; others succeed.

### 2.2 Immediate Steps

1. **Validate deployment status** – check if all pods of new version are healthy.
2. Look at **error rate graphs** and **logs** for stack traces around the time window.
3. If customer impact is high and rollback is easy, **roll back to previous stable version** while you analyze.

### 2.3 Investigation Plan

1. **Compare logs pre/post deployment**:
   - Filter logs by `traceId`/`correlationId` to see failing requests.[web:43][web:46]
   - Identify commonality: same endpoint, same downstream system, specific payloads.
2. **Check for configuration differences**:
   - Misconfigured environment variables or feature flags.
   - Wrong DB URL, credentials, or missing secrets.
3. **Check dependency and schema changes**:
   - New field added not handled in some paths.
   - DB migration not applied in one environment.

### 2.4 Example Root Cause & Fix

**Root cause:** A new validation rule for `paymentType` was deployed, but some existing clients still send the old value. When the service tries to map it to an enum, it throws `IllegalArgumentException`, leading to 500.

**Fixes:**
- Make mapping logic tolerant of unknown types and respond with 4xx.
- Log unknown values with a clear message.
- Update clients gradually with feature flags.

### 2.5 Interview Answer Shape

- Start with **rollout safety**: health checks, canary or blue–green.
- Then talk about using **centralized logs** and **trace IDs** to correlate failures across pods and downstream calls.[web:37][web:49]
- Emphasize config/schema mismatches as typical root causes.

---

## 3. One Microservice Down Causing Chain Failure

> _“One microservice goes down and causes a chain failure in other services. How will you prevent this in future?”_

### 3.1 Scenario

`profile-service` goes down due to DB issues. `payment-service` waits on it synchronously, causing timeouts. The gateway times out, and the entire user journey fails.

### 3.2 Core Ideas

This is a **resilience and isolation** problem, not just a bug:
- Synchronous dependencies create **cascading failures**.
- You need timeouts, circuit breakers, bulkheads, and fallbacks.

### 3.3 Prevention Strategies

1. **Timeouts Everywhere**
   - Configure client timeouts (WebClient/RestTemplate, Feign) to a sane lower value than gateway timeout.
2. **Circuit Breakers**
   - Use Resilience4j or similar; open the circuit if failure rate or latency crosses threshold.
3. **Bulkheads**
   - Separate thread pools per downstream dependency so that one slow service can’t consume all threads.
4. **Graceful Degradation**
   - For optional data (e.g., profile picture), return partial responses or cached/stale data.

### 3.4 Example

In `payment-service` using Resilience4j:

```java
@CircuitBreaker(name = "profileService", fallbackMethod = "profileFallback")
@TimeLimiter(name = "profileService")
public CompletableFuture<Profile> fetchProfile(String customerId) {
    return CompletableFuture.supplyAsync(() -> profileClient.getProfile(customerId));
}

public CompletableFuture<Profile> profileFallback(String customerId, Throwable ex) {
    // Return minimal profile or cached data
    return CompletableFuture.completedFuture(Profile.minimal(customerId));
}
```

### 3.5 Interview Answer

Explain that you would:
- Analyze the call graph using **distributed tracing**, identify critical sync dependencies.[web:44]
- Add **timeouts, circuit breakers, bulkheads** and potentially **asynchronous messaging** to decouple services.
- Use **graceful degradation**: failing closed on non-critical features.

---

## 4. API Latency Increased from 200ms to 3s After Release

> _“Your API response time increased from 200ms to 3 seconds after a new release. How will you identify the root cause?”_

### 4.1 Scenario

`account-service` balance endpoint, previously fast, is now slow. P99 latency is 3s.

### 4.2 Investigation Checklist

1. **Compare metrics before/after deployment**:
   - Latency histograms for the endpoint.
   - DB query timings, external calls.[web:42][web:45]
2. **Check DB**:
   - Slow query logs, missing indexes, increased locks.
3. **Check code**:
   - New remote calls added to the endpoint.
   - Changes in pagination or filtering logic.
4. **Use tracing**:
   - Look at trace waterfall to see where time is spent.

### 4.3 Example Root Cause & Fix

**Root cause:** A new feature loads transaction history as part of the balance call, resulting in heavy `JOIN` query and sorting without proper indexes.

**Fixes:**
- Split endpoint: keep balance endpoint simple; load history separately.
- Add appropriate DB indexes.
- Pre-compute or cache popular queries.

### 4.4 Interview Answer

Mention:
- “I will first look at **latency metrics and traces** to see exactly which span expanded.[web:44][web:48]
- Then I’ll check DB slow query logs and indexes.
- I’ll aim to **decouple heavy operations** from latency-sensitive endpoints.”

---

## 5. DB Connections Exhausted Under Load

> _“Database connections are getting exhausted under load. What steps will you take to fix this?”_

### 5.1 Scenario

At peak traffic, logs show `Connection is not available, request timed out after 30000ms`. The connection pool (HikariCP) hits its maximum size.

### 5.2 Investigation

1. **Check pool configuration** – `maximumPoolSize`, connection timeout.
2. **Identify leaks**:
   - Long-running transactions not committed/rolled back.
   - Blocking calls between `begin` and `commit`.
3. **Use metrics**:
   - Expose Hikari metrics via Micrometer (active connections, pending requests).[web:39][web:48]

### 5.3 Fixes

- Optimize queries and shorten transactions.
- Use pagination for large queries.
- Avoid opening transactions for operations that don’t require them.
- Increase pool size only after fixing code issues and confirming DB can handle more connections.

### 5.4 Interview Answer

Show you understand both **application** and **database** sides:
- “I’d inspect Hikari metrics and slow queries, fix leaks, and only then adjust pool size based on DB capacity and connection limits.”

---

## 6. Unreliable Third-Party Service (Frequent Timeouts)

> _“A third-party service you depend on is timing out frequently. How will you handle this in your system?”_

### 6.1 Key Patterns

- **Timeouts and retries with backoff**.
- **Circuit breaker** to stop hammering the third party.
- **Fallbacks** and **caching** of responses.

### 6.2 Example

For `credit-score-service` that calls external bureau:

- Use short client timeouts (e.g., 500–800ms).
- Retry only for safe, idempotent requests (GET-like), with jitter.
- Cache results for short periods to reduce load.

### 6.3 Interview Answer

- Emphasize **protecting your own system first**: timeouts, circuit breakers, rate limiting.
- Then talk about user experience: degrading gracefully, queuing requests, or allowing manual retries.

---

## 7. Duplicate Transactions in the System

> _“You observe duplicate transactions happening in your system. How will you prevent this?”_

### 7.1 Scenario

Due to retries in `payment-gateway`, some `PaymentCompletedEvent` messages are processed twice, leading to double ledger entries.

### 7.2 Strategies

1. **Idempotency keys**:
   - Use `requestId` or `idempotencyKey` in API.
   - Store request + result; if the same key comes again, return the previous result.
2. **Idempotent consumers**:
   - Store processed message IDs in DB or cache.
   - Use unique constraints on ledger table (`event_id` unique).
3. **Transactional outbox**:
   - Write domain changes and outgoing events atomically to same DB; publish from outbox to Kafka.

### 7.3 Example

```java
@Transactional
public PaymentResult processPayment(PaymentCommand cmd) {
    Optional<Payment> existing = paymentRepository.findByIdempotencyKey(cmd.idempotencyKey());
    if (existing.isPresent()) {
        return PaymentResult.from(existing.get());
    }

    Payment payment = createAndPersistPayment(cmd);
    paymentRepository.save(payment);
    outboxRepository.save(PaymentCompletedEvent.from(payment));

    return PaymentResult.from(payment);
}
```

### 7.4 Interview Answer

- Mention **idempotency on both HTTP and messaging**.
- Explain DB-level guards (unique constraints) + consumer deduplication.[web:26][web:31]

---

## 8. Logs Too Large and Distributed – Poor Observability

> _“Logs are too large and distributed, making debugging difficult. How will you improve observability?”_

### 8.1 Desired State

- **Centralized logging** (ELK/EFK or cloud-managed services).[web:37][web:49]
- **Structured JSON logs** with correlation IDs.
- Integration with metrics and traces.

### 8.2 Implementation Sketch

1. Configure each Spring Boot service with Logback JSON encoder.
2. Use Filebeat/Fluentd/CloudWatch agents to ship logs to Elasticsearch.[web:37][web:46]
3. Include fields: `timestamp`, `service`, `level`, `traceId`, `spanId`, `userId`, `endpoint`.
4. Build Kibana/Grafana dashboards for:
   - Error counts by service.
   - Latency, request volume, correlation of errors with deployments.

### 8.3 Interview Answer

Explain how you:
- Move from grepping individual pods to **searching in a single place**.
- Use correlation IDs to join logs with traces.
- Add **log sampling** to keep volume under control.

---

## 9. Memory Leak – Usage Keeps Growing Until Crash

> _“Memory usage keeps increasing and your service crashes after some time. How will you detect and fix memory leaks?”_

### 9.1 Investigation

1. Confirm pattern from metrics – heap usage steadily climbs without dropping after GC.[web:45][web:48]
2. Capture **heap dumps** periodically (Actuator `/heapdump` or jmap).
3. Analyze with tools (Eclipse MAT, VisualVM) to find:
   - Large collections growing unbounded.
   - Caches without eviction.
   - Thread-local or static references.

### 9.2 Typical Root Causes

- In-memory cache keyed by user/session without TTL.
- Large results kept in `List` across requests.
- Using `@SessionAttributes` or storing big objects in HTTP session in stateful services.

### 9.3 Fixes

- Add size/TTL to caches and monitor hit rates.
- Stream data instead of building huge collections.
- Clear references after use.

### 9.4 Interview Answer

Talk about:
- Using **metrics + heap dumps** to locate leak.
- Fixing design (cache, collections) instead of only adding RAM.

---

## 10. Works on Local but Fails in Production

> _“Your microservice works fine locally but fails in production. How will you approach debugging?”_

### 10.1 Common Differences

- Config (URLs, credentials, feature flags).
- Data (test vs real).
- Scale (one instance vs many).
- Security (auth, TLS, firewalls).

### 10.2 Approach

1. Reproduce issue on a **staging environment** similar to prod.
2. Compare configuration: use `/actuator/env` or config server.[web:39]
3. Check network – DNS, firewalls, security groups.
4. Look at **prod-only integrations** (real third-party endpoints).

### 10.3 Interview Answer

Highlight you won’t just say “works on my machine”:
- You’ll rely on **infrastructure-as-code, config management, and staging environments**.

---

## 11. Deployment Breaks One Feature – Safe Rollback

> _“A new deployment breaks one feature but works for others. How will you safely roll back?”_

### 11.1 Practices

- Use **immutable artifacts** (Docker images) with clear versioning.
- Deploy using **blue–green** or **rolling update** strategies.
- Maintain **backward-compatible DB schema** during rollout.[web:21]

### 11.2 Steps

1. Detect issue via canary metrics or user reports.
2. Route traffic back to previous version (blue–green) or roll back deployment.
3. Ensure DB migrations are backward-compatible so rollback is safe.
4. Run automated smoke tests after rollback.

### 11.3 Interview Answer

Emphasize:
- Separation of **deploy** and **release** (feature flags).
- Observability + automation for rollback.

---

## 12. 5x Traffic Spike – Scaling Strategy

> _“Traffic suddenly spikes 5x during peak hours and your service becomes slow. How will you scale?”_

### 12.1 Short-Term

- **Horizontally scale** pods using HPA based on CPU or custom QPS metrics.
- Ensure DB, caches, and downstreams can handle increased load.

### 12.2 Long-Term

- Introduce **caching** (per-user data, configuration lookups).
- Optimize hot paths and DB queries.
- Use **rate limiting** and **back-pressure** for non-critical requests.

### 12.3 Interview Answer

Show understanding that scaling is end-to-end:
- App pods, DB, cache, queues, third parties.

---

## 13. Network Latency Between Services

> _“Inter-service communication is failing due to network latency. How will you optimize it?”_

### 13.1 Strategies

- Reduce **chatty calls** – combine multiple small requests into one.
- Use **gRPC** or binary protocols if suitable.
- Deploy services in same region/availability zone.
- Implement **client-side caching** and **timeouts**.

### 13.2 Interview Answer

Combine architecture and infra:
- “I’ll examine traces to find noisy neighbors and chatty patterns, then redesign APIs or use messaging where synchronous latency is not critical.”[web:44]

---

## 14. Tracing a Request Across Multiple Services

> _“You need to trace a single request across multiple services during a failure. How will you implement tracing?”_

### 14.1 Implementation

1. Use **OpenTelemetry / Spring Cloud Sleuth** to generate trace/span IDs.
2. Propagate trace headers (`traceparent`, `X-B3-*`) through HTTP/Kafka.
3. Configure exporters to Zipkin/Jaeger.
4. Incorporate `traceId` into logs.

### 14.2 Interview Answer

Explain:
- How traces allow you to see the full **call graph** and latencies end-to-end.[web:44][web:47]
- How you click on a failing span and correlate it with logs.

---

## 15. Data Inconsistency Across Multiple Services

> _“A bug in one service causes inconsistent data across multiple services. How will you handle data consistency?”_

### 15.1 Concepts

- **Eventual consistency** vs strong consistency.
- **Saga pattern** for long-running transactions.
- Compensating actions.

### 15.2 Example – Payment Saga

1. `payment-service` reserves amount and publishes `PaymentReserved`.
2. `ledger-service` writes entries; `notification-service` sends messages.
3. If ledger write fails, a **compensating event** triggers refund.

### 15.3 Fixing Existing Inconsistency

- Build **reconciliation jobs** that compare states between services (e.g., sums in ledger vs account balances).
- Fix data with controlled scripts and audit.

### 15.4 Interview Answer

Show that you:
- Design for **eventual consistency**.
- Provide **operational tools** (reconciliation, backfills) to correct issues when they inevitably happen.[web:22]

---

## Conclusion – How to Practice

To truly internalize these scenarios:
- Implement a **demo microservices system** and deliberately inject bugs (slow queries, unbounded caches, failing dependencies).
- Use **Prometheus, Grafana, ELK, and tracing** to debug them the way you described.[web:39][web:42][web:49]
- Convert each scenario into a **personal story** from your own projects so that answers sound natural in interviews.

If you can walk through these 15 incidents calmly and systematically, you already operate at true production level.
