# 15 Real-World Spring Boot & Microservices Incidents – Deep-Dive Edition

_A production-focused handbook for Java/Spring Boot engineers preparing for scenario-based interviews and real on-call life._

This extended edition goes much deeper than the first version. For each of the 15 incidents you will get:
- A **detailed story** from a digital banking / payments system.
- A **signal-based investigation plan** (what you see in metrics/logs/traces and how to interpret it).
- Typical **root causes** with concrete code/config examples.
- **Fix strategies** at code, infra, and process level.
- How to **answer like a senior engineer** in interviews.

We assume you are working with:
- Java 17+, Spring Boot 3.
- Kubernetes, Docker.
- Observability stack: Actuator, Micrometer/Prometheus, ELK/EFK, and OpenTelemetry tracing.[web:55][web:63]

---

## 0. Production Debugging Mindset

Before jumping into scenarios, set your mindset correctly.

### 0.1 Principles

1. **Data first, opinions later** – always look at metrics, logs, and traces before guessing.[web:55]
2. **Stabilize, then analyze** – protect customers with quick mitigations, then dig deeper.
3. **Change-aware** – most incidents are caused by a **recent change**: code, config, infra, data.
4. **System thinking** – think across services, database, network, queue, and external dependencies.

### 0.2 Core Observability Tools

- **Metrics**: Micrometer + Prometheus + Grafana – CPU, memory, GC, request rate, latency, DB pool metrics.[web:42][web:48]
- **Logs**: centralized JSON logs via ELK/EFK (Logstash/Fluentd + Elasticsearch + Kibana).[web:37][web:58]
- **Traces**: OpenTelemetry for end-to-end request flows across services.[web:54][web:60]
- **Dumps**: heap dumps and thread dumps from Actuator endpoints (`/actuator/heapdump`, `/actuator/threaddump`).[web:51]

In interviews, explicitly mention these tools and how you combine them.

---

## 1. CPU Spikes to 90% in Production

> _“Your Spring Boot service CPU suddenly spikes to 90% in production. How will you investigate and fix it?”_

### 1.1 Detailed Scenario

`statement-service` generates PDF statements for millions of customers. Normally CPU hovers around 40–50%. After a new release that introduces “download last 24 months statement” feature, CPU spikes to 90–100% in all pods. Alerts fire, requests time out, and Kubernetes starts restarting pods.

### 1.2 Signals to Look At

1. **Metrics dashboard**:
   - CPU usage per pod over time.
   - Request rate and error rate for key endpoints.
   - GC metrics: GC pause time, frequency.[web:45][web:59]
2. **Thread dump** from one of the hot pods:
   - Threads stuck in `Runnable` state in tight loops.
   - High number of threads in a specific pool (`statementWorkerPool`).[web:51]
3. **Traces** for slow requests:
   - Long spans in `generateStatements()`.

### 1.3 Investigation Workflow

1. **Confirm scope** – is it all pods or only some? If only some, suspect **noisy neighbor** or traffic imbalance.
2. **Capture 2–3 thread dumps** spaced a few seconds apart and compare stacks; repeated stacks indicate hotspot functions.[web:51]
3. **Correlate with recent deploys** – check Git history and release notes for heavy operations: new loops, big joins, PDF generation.
4. **Profile locally** – reproduce on staging with load test; attach Java Flight Recorder or async-profiler to find hot methods.[web:38]

### 1.4 Typical Root Causes & Code Examples

1. **Unbounded loops / heavy collections**:

```java
// BAD: iterates entire history synchronously per request
for (Transaction txn : allTransactions) {
    pdfBuilder.append(txn.format());
}
```

2. **Parallel streams gone wrong** – parallelizing CPU-heavy tasks on already busy thread pools.

3. **Inefficient serialization** – large objects converted to JSON repeatedly instead of once.

4. **Accidental busy-wait loop**:

```java
while (!jobCompleted.get()) {
    // missing sleep or wait -> burns CPU
}
```

### 1.5 Fix Strategies

- **Batch and paginate** heavy work:

```java
PageRequest page = PageRequest.of(0, 1000);
Page<Transaction> txns;

while (!(txns = repo.findByAccount(accountId, page)).isEmpty()) {
    process(txns.getContent());
    page = page.next();
}
```

- Move large statement generation to **async batch jobs** triggered via message queue, store result location and return job ID.
- Optimize algorithms: pre-aggregate data, cache templates.
- Add **load/performance tests** simulating worst-case usage.

### 1.6 Interview Answer Template

In the interview, walk them through:

1. “I first look at **CPU and GC metrics** to confirm it’s compute-bound, not blocked I/O.[web:45]”
2. “I capture **thread dumps** and use them to find hot methods.[web:51]”
3. “Then I profile on staging with production-like load to reproduce and optimize using **batching and async jobs**.”
4. “Finally, I add **alerts and load tests** to avoid regressions.”

---

## 2. Intermittent 500 Errors After Deployment

> _“After deployment, your service starts throwing intermittent 500 errors. How will you debug this issue?”_

### 2.1 Detailed Scenario

A new version of `payment-service` adds support for a loyalty points feature. Post-deploy, about 3% of checkout requests fail with `500 Internal Server Error`, mainly for older mobile app versions.

### 2.2 Signals and Dashboards

- Error-rate panel shows a spike for `POST /payments` only on pods with the new version.
- Central logs show stack traces like `IllegalArgumentException: No enum constant PaymentType.REWARD`.
- Traces show failures happening before any DB call – purely in validation layer.

### 2.3 Root Cause Pattern

The new code assumes `PaymentType` enum has only modern values.

```java
PaymentType type = PaymentType.valueOf(request.getType()); // throws for unknown
```

Older clients still send `"REWARD"`, which no longer exists.

### 2.4 Fix and Hardening

- Replace `valueOf` with safe mapping:

```java
public Optional<PaymentType> from(String type) {
    return Arrays.stream(values())
        .filter(t -> t.name().equalsIgnoreCase(type))
        .findFirst();
}
```

- For unknown types, respond `400 Bad Request` with clear message instead of `500`.
- Add **contract tests** or **consumer-driven contracts** to catch incompatible changes before deployment.

### 2.5 Interview Answer Emphasis

- Mention **roll back or route traffic** to old version if impact is high.
- Show that you rely on **centralized logs** and **trace correlation IDs** to find which payloads are failing.[web:37][web:58]
- Talk about **API versioning and backward compatibility**.

---

## 3. Chain Failure When One Service Is Down

> _“One microservice goes down and causes a chain failure in other services. How will you prevent this in future?”_

### 3.1 Scenario

`profile-service` DB becomes unavailable. `payment-service` calls it synchronously at checkout to fetch KYC level. Call timeouts pile up, thread pool gets exhausted, causing 5xx at gateway and eventually cascading failure across the payment flow.

### 3.2 Detection

- Traces show long spans for the call to `profile-service` (>5s).
- Metrics show thread pool (`Tomcat`, `WebClient`) saturation; active threads ~ max.
- Logs show `Read timed out` errors from profile client.

### 3.3 Long-Term Prevention Design

1. **Set timeouts shorter than upstream** (e.g., `1s` client timeout if gateway is `3s`).
2. Add **circuit breaker**:

```java
@CircuitBreaker(name = "profile", fallbackMethod = "fallbackKyc")
@TimeLimiter(name = "profile")
public CompletableFuture<KycInfo> kyc(String customerId) { ... }
```

3. Use **bulkheads** – separate thread pools per downstream to avoid full exhaustion.
4. Apply **graceful degradation** – if KYC is optional, proceed with reduced limits; else, tell user clearly to retry later.

### 3.4 Interview Answer

Articulate:
- “I’ll use **timeouts, circuit breakers, bulkheads, and async messaging** to prevent one dependency from taking down everything.”
- “Tracing and metrics help me identify which hop is failing and for how long.”[web:54][web:55]

---

## 4. Latency Jump from 200ms to 3s After Release

> _“Your API response time increased from 200ms to 3 seconds after a new release. How will you identify the root cause?”_

### 4.1 Scenario

`GET /accounts/{id}/balance` now also loads last 50 transactions to show in mobile app. Latency jumps; customers time out.

### 4.2 Observability Flow

1. **Latency metrics** – p50, p95, p99 per endpoint show clear regression after deployment.[web:42]
2. **Traces** – new spans appear: `fetchRecentTransactions`, DB calls taking ~2.5s.
3. **DB slow query logs** – reveal full table scans due to missing index on `account_id, tx_date`.

### 4.3 Fix Plan

- Split API: keep `balance` separate from `recentTransactions` to protect latency-critical endpoint.
- Add index:

```sql
CREATE INDEX idx_tx_account_date ON tx(account_id, tx_date DESC);
```

- Introduce caching for most recent transactions per account.

### 4.4 Interview Messaging

Explain how you use **tracing to pinpoint slow spans**, then **DB analysis** and **schema changes** to fix them.[web:60][web:63]

---

## 5. DB Connections Exhausted Under Load

> _“Database connections are getting exhausted under load. What steps will you take to fix this?”_

### 5.1 Scenario

Connection pool (Hikari) size is 30. During campaign, active connections hover at 30 and many threads wait, causing timeouts.

### 5.2 Observations

- Micrometer exposes `hikari.connections.active`, `idle`, `pending`.[web:48]
- Logging shows `Timeout after 30000ms waiting for connection from pool`.
- DB monitoring shows long-running queries and locks.

### 5.3 Root Cause Examples

1. **Slow queries** – complex joins, missing indexes.
2. **Leaked connections** – manual `DataSource.getConnection()` without close.
3. **Too-long transactions** – business logic (network calls) inside transaction boundaries.

### 5.4 Fix and Optimization

- Refactor code to keep DB transactions **short and focused**.
- Move long business logic out of transactional block.
- Add proper indexes and limit query result size.
- Only after optimization, adjust `maxPoolSize` based on DB capacity.

### 5.5 Interview Angle

Stress that you **fix the cause first**, not just increase pool size; also mention using **pool metrics** to prove improvement.[web:45]

---

## 6. Flaky Third-Party Service (Timeouts)

> _“A third-party service you depend on is timing out frequently. How will you handle this in your system?”_

### 6.1 Scenario

`credit-bureau-api` becomes unreliable during evenings. Your `risk-service` calls it for every high-value payment; user experience suffers.

### 6.2 Strategies in Layers

- **Client layer**: timeouts, retries with exponential backoff + jitter, circuit breaker.
- **Domain layer**: risk-based decision; maybe allow low-value payments with stale or cached score.
- **UX layer**: show clear messages or queue request and notify later.

### 6.3 Example with WebClient

```java
WebClient client = WebClient.builder()
    .baseUrl(bureauUrl)
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create()
            .responseTimeout(Duration.ofMillis(800))
    ))
    .build();
```

Use caching (e.g., Redis) for last-known score for 24 hours.

### 6.4 Interview Answer

Show layered thinking: **protect your system, protect your customers, then help third party** (by reducing load).

---

## 7. Duplicate Transactions

> _“You observe duplicate transactions happening in your system. How will you prevent this?”_

### 7.1 Detailed Story

UPI payments sometimes process twice because PSP retries on network errors; both requests hit your API with same details. Kafka also redelivers `PaymentCompletedEvent` on consumer restart.

### 7.2 End-to-End Idempotency

1. **API Layer** – require `Idempotency-Key` header or `requestId`.
2. **Service Layer** – store idempotency key + status in DB table.
3. **Messaging Layer** – store processed message IDs or enforce unique constraint on event ID.

### 7.3 Example Schema

```sql
CREATE TABLE payment (
  id BIGSERIAL PRIMARY KEY,
  idempotency_key VARCHAR(64) UNIQUE,
  status VARCHAR(20) NOT NULL,
  amount NUMERIC(18,2) NOT NULL
);

CREATE TABLE processed_event (
  event_id UUID PRIMARY KEY,
  processed_at TIMESTAMP NOT NULL
);
```

### 7.4 Interview Emphasis

Tie back to **exactly-once semantics**, **Kafka idempotent producers**, and **consumer deduplication**.[web:31]

---

## 8. Logs Too Large and Distributed

> _“Logs are too large and distributed, making debugging difficult. How will you improve observability?”_

### 8.1 Centralized Logging Architecture

- Each service logs JSON using `logstash-logback-encoder` with fields like `service`, `env`, `traceId`, `spanId`, `userId`.[web:37]
- Filebeat/Fluentd agents forward logs to Logstash/Elasticsearch.
- Kibana dashboards visualize error trends and support powerful search filters.[web:58][web:61]

### 8.2 Example Logback Config Snippet

```xml
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
  <destination>logstash:5000</destination>
  <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
    <providers>
      <timestamp/>
      <pattern>
        <pattern>{"level":"%level","service":"${spring.application.name}","traceId":"%X{traceId}","spanId":"%X{spanId}","msg":"%msg"}</pattern>
      </pattern>
    </providers>
  </encoder>
</appender>
```

### 8.3 Interview Points

- Explain shift from **scattered text logs** to **queryable structured logs**.
- Mention index lifecycle management, retention policies, and PII masking.

---

## 9. Memory Usage Keeps Growing

> _“Memory usage keeps increasing and your service crashes after some time. How will you detect and fix memory leaks?”_

### 9.1 Signal-Based Workflow

1. Micrometer metrics: heap usage plateauing at high level with growing GC pause times.[web:45][web:59]
2. Capture heap dumps periodically via `/actuator/heapdump` and analyze in Eclipse MAT or specialized tools.[web:51][web:62]
3. Identify largest retained objects and which references keep them alive.

### 9.2 Classic Leak Examples

- `ConcurrentHashMap` as ad-hoc session store without eviction in `@Service`.
- `@Cacheable` caches without TTL or max size.
- Large `List` kept as static field for debugging and never cleared.[web:51][web:62]

### 9.3 Fixes

- Use proper cache libraries with TTL and maximum size (Caffeine, Redis).
- Remove long-lived collections or bound them explicitly.
- Fix thread pools with unbounded queues that hold tasks and references.[web:56]

### 9.4 Interview Answer

Describe **tool-based debugging** rather than guesswork; mention at least one workflow using heap dump + MAT.[web:51]

---

## 10. Works Locally, Fails in Production

> _“Your microservice works fine locally but fails in production. How will you approach debugging?”_

### 10.1 Types of Differences

- Configuration (URLs, credentials, timeouts).
- Data volumes and shapes (test vs prod).
- Security and networking (TLS, firewalls, service mesh).
- Resource limits (CPU/memory) and autoscaling.

### 10.2 Practical Steps

1. Compare effective configuration via `env` and config server.
2. Reproduce in a **staging cluster** with similar infra.
3. Use **feature flags** to disable risky code paths in prod until fixed.
4. Check for missing secrets or wrong IAM roles.

### 10.3 Interview Line

- Stress that you treat “works on my machine” as useless; you strive for **environment parity** and **config-as-code**.

---

## 11. Deployment Breaks One Feature – Safe Rollback

> _“A new deployment breaks one feature but works for others. How will you safely roll back?”_

### 11.1 Deployment Strategy

- Use **blue–green** or **canary** deployment (e.g., via Kubernetes + service mesh or ingress rules).[web:21]
- Maintain DB migrations as **backward compatible** (add columns, don’t drop immediately).

### 11.2 Rollback Process

1. Detect failure quickly via synthetic checks and error-rate alerts.
2. Route traffic back to previous version (blue–green) or roll back ReplicaSet.
3. Confirm health via smoke tests and dashboards.
4. Perform **postmortem** to avoid same mistake.

### 11.3 Interview Answer

Show maturity:
- “Rollback should be a **button click**, not a manual script.”

---

## 12. 5x Traffic Spike – Scaling Strategy

> _“Traffic suddenly spikes 5x during peak hours and your service becomes slow. How will you scale?”_

### 12.1 Short-Term Response

- Increase pod replicas via HPA using CPU/QPS metrics.
- Ensure DB read replicas and caches are sized appropriately.
- If necessary, enable **read-only degradation** (e.g., disable some non-critical writes).

### 12.2 Long-Term Architecture

- Introduce **caching** at app and CDN layers.
- Throttle or **rate limit** abusive clients.
- Use **queue + worker** model for non-critical tasks (emails, reports).

### 12.3 Interview Points

Discuss trade-offs between vertical and horizontal scaling and mention capacity planning via load tests.

---

## 13. Network Latency Between Services

> _“Inter-service communication is failing due to network latency. How will you optimize it?”_

### 13.1 Signals

- Traces show long client spans with no work, just waiting.[web:57][web:60]
- Network dashboards reveal high RTT between zones or regions.

### 13.2 Remedies

- **Co-locate** dependent services in same region/zone.
- Reduce chatty calls – move to **coarse-grained APIs** or **GraphQL**.
- For internal communication, consider **gRPC** for better performance.
- Use **connection pooling** and keep-alive configs.

---

## 14. Tracing a Request End-to-End

> _“You need to trace a single request across multiple services during a failure. How will you implement tracing?”_

### 14.1 Implementing OpenTelemetry in Spring Boot

- Use Spring Boot’s Micrometer Tracing with OpenTelemetry exporter to OTLP.[web:54][web:63]
- Configure service name, environment, and exporter endpoint.
- Ensure trace headers are propagated via HTTP clients and Kafka.

### 14.2 Operational Use

- When failure occurs, grab a **trace ID** from logs and open it in Jaeger/Tempo.
- Inspect spans to see where latency or errors occur.

### 14.3 Interview Answer

Anchor on:
- “Traces complement logs and metrics; they give **timeline + context** across services.”[web:52][web:57]

---

## 15. Data Inconsistency Across Multiple Services

> _“A bug in one service causes inconsistent data across multiple services. How will you handle data consistency?”_

### 15.1 Reconciliation and Repair

- Build **reconciliation jobs** that compare authoritative sources (ledger vs account balances, payment vs orders).
- Identify mismatches and produce **repair events**.

### 15.2 Preventive Architecture

- Use **outbox pattern** and **event sourcing** to avoid lost updates.
- Apply **Saga patterns**: orchestrated or choreographed workflows with compensating actions.

### 15.3 Interview Frame

Show that you understand **eventual consistency** and that operations teams need tools to heal data.

---

## 16. How to Practice These Incidents

1. Create a small microservices playground (account, payment, notification).
2. Set up **Prometheus, Grafana, ELK, and OpenTelemetry** as described in modern guides.[web:55][web:58]
3. Intentionally inject problems (slow queries, memory leaks, crashed dependencies) and follow the playbook.
4. Convert each resolved incident into a **personal story** with clear metrics before/after.

If you can walk through these 15 incidents with concrete steps, tools, and metrics, you will sound like someone who has actually run production systems—not just read theory—and interviewers will treat you as a true senior engineer.
