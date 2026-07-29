# Ultimate Senior Java Daily 30-Minute Revision Master Cheat Sheet

An ultra-dense, comprehensive, single-page rapid revision cheat sheet covering **every core domain** in the repository. Designed for senior Java candidates to review 30 minutes before any technical interview.

---

## ⚡ 1. Java 21 & Concurrency Rapid Reference

- **Virtual Threads (`Thread.ofVirtual()`)**:
  - $M:N$ mapping: $M$ user-mode Virtual Threads mounted on $N$ OS Carrier Threads (`ForkJoinPool`).
  - **Do NOT pool Virtual Threads!** Create a new one per task via `Executors.newVirtualThreadPerTaskExecutor()`.
- **Thread Pinning**:
  - Caused by blocking I/O inside `synchronized` blocks or `JNI` native calls.
  - **Fix**: Replace `synchronized` with `java.util.concurrent.locks.ReentrantLock`.
  - **Detect**: Run JVM with `-Djdk.tracePinnedThreads=full`.
- **Structured Concurrency (`StructuredTaskScope.ShutdownOnFailure`)**:
  - Treats concurrent sub-tasks as a single unit of work; cancels remaining sub-tasks if one fails.
- **Scoped Values (`ScopedValue<T>`)**:
  - Immutable, bounded context propagation across virtual threads (replaces `ThreadLocal`).
- **Sequenced Collections (`SequencedCollection<E>`)**:
  - Unified methods: `addFirst()`, `addLast()`, `getFirst()`, `getLast()`, `reversed()`.

---

## ⚡ 2. Java 17 LTS Features Rapid Reference

- **Records (`public record UserDto(Long id, String name) {}`)**:
  - Immutable data carrier. Compact constructor for validation: `public UserDto { Objects.requireNonNull(name); }`.
- **Sealed Classes (`public abstract sealed class Shape permits Circle, Square`)**:
  - Restricts subclassing. Subclasses must be `final`, `sealed`, or `non-sealed`. Enables exhaustive `switch`.
- **Pattern Matching for `switch` & `instanceof`**:
  - Guard conditions: `case String s && s.length() > 5 -> ...`.
  - Type cast atomic binding: `if (obj instanceof String s) { s.toLowerCase(); }`.

---

## ⚡ 3. Database Tuning, JPA & Transactions Rapid Reference

- **HikariCP Pool Sizing Formula**:
  $$\text{Max Pool Size} = (\text{CPU Cores} \times 2) + \text{Effective Spindles}$$
  - *Example*: 8 CPU cores on NVMe SSD $\rightarrow (8 \times 2) + 1 = 17$ connections.
  - Leak detection: `hikari.leak-detection-threshold=2000` (logs warning if thread holds connection $> 2\text{s}$).
- **`@Transactional` Propagation Matrix**:
  - `REQUIRED` (default): Joins existing transaction or creates new one.
  - `REQUIRES_NEW`: Suspends active transaction and opens brand-new transaction.
  - **Self-Invocation Trap**: Internal call inside same class bypasses Spring proxy! Fix by moving to separate `@Service`.
- **Isolation Levels & Anomalies**:
  - `READ_COMMITTED` (PostgreSQL default): Prevents Dirty Reads.
  - `REPEATABLE_READ` (MySQL default): Prevents Dirty & Non-Repeatable Reads.
  - `SERIALIZABLE`: Prevents Phantom Reads (maximum locking).
- **N+1 Query Elimination (3 Fixes)**:
  1. `JOIN FETCH o.customer` in JPQL.
  2. `@EntityGraph(attributePaths = {"customer"})` on Repository method.
  3. DTO Projection: `@Query("SELECT new com.example.UserDto(...) FROM Order o")`.

---

## ⚡ 4. Distributed Transactions & Microservices Rapid Reference

- **The Dual-Write Problem**: Writing to DB + sending Kafka message in one method leads to inconsistency on failure.
- **Transactional Outbox Pattern**:
  1. Write domain entity AND outbox event record in the **SAME local DB transaction**.
  2. Debezium CDC engine reads DB Write-Ahead Log (WAL) asynchronously and publishes to Kafka.
- **Saga Pattern**:
  - **Choreography**: Event-driven; services listen and react to domain events.
  - **Orchestration**: Centralized Saga Orchestrator manages state machine and executes **Compensating Transactions** on failure in reverse order.
- **Idempotency Key Pattern**:
  - Enforce idempotency on consumers using Redis: `opsForValue().setIfAbsent("idempotency:" + key, "PROCESSING", 24h)`.

---

## ⚡ 5. JVM Diagnostics & Production Troubleshooting Rapid Reference

- **The 4 OutOfMemoryError (OOM) Types**:
  1. `Java heap space`: Young/Old Gen full. Fix: Analyze heap dump in Eclipse MAT.
  2. `Metaspace`: Classloader leak / dynamic proxies. Fix: `-XX:MaxMetaspaceSize`.
  3. `Direct buffer memory`: Off-heap Netty/NIO buffer leak. Fix: Check Netty resource leaks.
  4. `unable to create native thread`: Process OS thread limit hit. Fix: Virtual Threads / `ulimit -u`.
- **Eclipse MAT Retained vs. Shallow Heap**:
  - *Shallow Heap*: Memory of object itself. *Retained Heap*: Total memory freed if object is garbage collected.
- **High-CPU Thread RCA (`top -H` to `jstack`)**:
  1. Get high-CPU thread PID via `top -H -p <JVM_PID>`.
  2. Convert decimal Thread LWP ID to hex: `printf "%x\n" <LWP_ID>` (e.g. `14205` $\rightarrow$ `377d`).
  3. Search thread dump: `jcmd <PID> Thread.print | grep -A 30 "nid=0x377d"`.
- **Low-Overhead Profiling**:
  - JFR: `jcmd <PID> JFR.start duration=60s filename=/tmp/prof.jfr`.
  - async-profiler: `./asprof -d 30 -f /tmp/flamegraph.html <PID>`.

---

## ⚡ 6. Resilience4j & Observability Rapid Reference

- **Circuit Breaker States**:
  - `CLOSED` (Normal) $\rightarrow$ Failure Rate $> 50\% \rightarrow$ `OPEN` (Fast Failure) $\rightarrow$ Wait Duration Expires $\rightarrow$ `HALF_OPEN` (Trial Probes) $\rightarrow$ `CLOSED`.
- **Exponential Backoff + Jitter**:
  $$\text{Delay} = \text{Initial} \times (\text{Multiplier}^{\text{Attempt}}) \pm \text{Random Jitter}$$
  - *Jitter prevents Thundering Herd spikes on downstream recovery.*
- **W3C `traceparent` Header Format**:
  - `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01` (`version-traceId-spanId-flags`).
- **Async MDC Preservation**:
  - Use custom Spring `TaskDecorator` to copy `MDC` context map from caller thread to worker pool thread.

---

## ⚡ 7. Security, OAuth2 & OWASP Rapid Reference

- **OAuth2 PKCE (Proof Key for Code Exchange)**:
  - Client sends `code_challenge` (SHA-256 of `code_verifier`) during authorization; sends `code_verifier` during token exchange.
  - Resource Server verifies JWT asynchronously via JWKS endpoint (`/.well-known/jwks.json`).
- **Spring Security 6 Stateless Config**:
  - `SecurityFilterChain`: `.csrf(csrf -> csrf.disable())`, `.sessionManagement(s -> s.sessionCreationPolicy(STATELESS))`.
- **OWASP BOLA / IDOR Defense**:
  - Authorize at the **object level**, not just URI level: `@PreAuthorize("@orderSecurity.isOwner(#id, authentication.name)")`.

---

## ⚡ 8. Core System Design Capacity Formulas

- **Throughput (Requests Per Second)**: $\text{RPS} = \frac{\text{Concurrent Active Users} \times \text{Requests per User}}{\text{Session Duration (seconds)}}$
- **Latency & Concurrency (Little's Law)**: $\text{Concurrent Requests} = \text{RPS} \times \text{Average Latency (seconds)}$
- **Bandwidth Estimation**: $\text{Bandwidth} = \text{RPS} \times \text{Average Payload Size}$

---

## 💡 Pre-Interview Checklist (Last 5 Minutes)

- [ ] I can explain why HikariCP connection pool size should be small ($(\text{Cores} \times 2) + 1$).
- [ ] I can explain Thread Pinning in Java 21 Virtual Threads and how to fix it (`ReentrantLock`).
- [ ] I can describe the Transactional Outbox + Debezium CDC pattern to solve Dual-Writes.
- [ ] I know how to convert Linux LWP thread ID to hex `nid` (`printf "%x\n" <LWP_ID>`) to find high CPU line numbers in a `jstack` dump.
- [ ] I can explain BOLA/IDOR security fixes using Spring Security `@PreAuthorize` with SpEL.
