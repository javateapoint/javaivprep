# Master Java & Senior Backend Engineering Interview Prep Suite

Welcome to the **Master Java & Senior Backend Engineering Repository**. This repository contains production guides, architecture cheat sheets, system design mega-guides, AI agent suites, and interview preparation materials covering **Java 17/21**, **Spring Boot 3.x**, **Microservices**, **Kafka**, **Redis**, **DevSecOps**, **SRE**, and **System Design**.

---

## 🌟 Senior Java (7+ YOE) Master Interview Suite

A dedicated, production-grade master suite engineered specifically for Senior, Lead, and Principal Software Engineers:

- 🚀 [01. Java 21 Virtual Threads & Modern Concurrency Guide](Senior-Java-7Plus-Guide/01-Java21-Virtual-Threads-Concurrency.md) — Project Loom, Carrier Threads, Thread Pinning refactoring (`synchronized` $\rightarrow$ `ReentrantLock`), Structured Concurrency (`StructuredTaskScope`), Scoped Values, and Sequenced Collections.
- 🛢️ [02. Database Tuning, HikariCP & JPA Transactions Guide](Senior-Java-7Plus-Guide/02-Database-Tuning-HikariCP-JPA-Transactions.md) — HikariCP pool sizing formula, leak detection, `@Transactional` isolation & propagation, N+1 query elimination (`JOIN FETCH`, `@EntityGraph`, Projections), and `EXPLAIN ANALYZE` profiling.
- 🔄 [03. Distributed Transactions, Saga & Outbox Pattern Guide](Senior-Java-7Plus-Guide/03-Distributed-Transactions-Saga-Outbox-Pattern.md) — Dual-write problem, Transactional Outbox + Debezium CDC, Saga Orchestration vs Choreography, Compensating transactions, and Idempotency keys.
- 🔍 [04. JVM Production Diagnostics: Memory & Thread Dumps Guide](Senior-Java-7Plus-Guide/04-JVM-Production-Diagnostics-Memory-Thread-Dumps.md) — OOM variants (`Java heap space`, `Metaspace`, `Direct buffer memory`), Eclipse MAT retained heap analysis, mapping Linux `top -H` LWP ID to `jstack` hex `nid`, JFR, and async-profiler.
- 🛡️ [05. Resilience4j, Distributed Tracing & Observability Guide](Senior-Java-7Plus-Guide/05-Resilience4j-Distributed-Tracing-Observability.md) — Circuit Breaker states (`CLOSED`, `OPEN`, `HALF_OPEN`), Rate Limiter, Bulkheads, Exponential Backoff + Jitter, W3C `traceparent` headers, Micrometer Tracing, and `MDC` context propagation.
- 🔒 [06. OAuth2, Spring Security 6.x & DevSecOps Guide](Senior-Java-7Plus-Guide/06-OAuth2-Spring-Security6-DevSecOps.md) — PKCE Authorization Code flow, JWKS validation, stateless `SecurityFilterChain`, custom `OncePerRequestFilter`, and OWASP API Top 10 (BOLA/IDOR, BFLA, SQLi).
- 🏗️ [07. Senior System Design Production Blueprints Guide](Senior-Java-7Plus-Guide/07-System-Design-Production-Blueprints-7Plus.md) — High-throughput Idempotent Payment Gateway and Multi-Channel Notification Engine architectures with diagrams, DB schemas, and trade-off Q&A.
- 🤖 [08. Modern Senior Java Interview Pattern & AI-Assisted Prep Guide (2025–2026)](Senior-Java-7Plus-Guide/08-Modern-Interview-Pattern-AI-Roundwise-Guide.md) — Round-by-round breakdown, AI-assisted coding/pair programming, PR teardowns, Spring AI & RAG architectures, and STAR behavioral leadership.
- ⚡ [09. Ultimate Senior Java Daily 30-Minute Revision Master Cheat Sheet](Senior-Java-7Plus-Guide/09-Ultimate-Daily-Revision-Cheatsheet.md) — Ultra-dense rapid revision cheat sheet covering Virtual Threads, HikariCP formulas, Outbox/Saga, JVM OOM/Thread dumps, Security 6, and System Design capacity formulas in one place.
- 🧠 [10. End-to-End AI & LLM Engineering Master Cheat Sheet](Senior-Java-7Plus-Guide/10-End-To-End-AI-LLM-Master-Cheatsheet.md) — Transformers, Self-Attention ($Q,K,V$), RAG Architecture, Vector DBs (Pgvector/HNSW), AI Agents (ReAct), MCP Protocol, and Spring Boot 3 Function Calling.

---

## 📚 General Topic Index

### 1. Core Java & Concurrency
- 📖 [240 Core Java Questions & Answers](240QuesJava.md) — Fundamental questions on JVM, OOP, collections, and primitives.
- ⚡ [Java 17 LTS Features Guide](Java%2017%20Features.md) — Records, Sealed Classes, Pattern Matching, Text Blocks, and PRNGs.
- 🔄 [CompletableFuture & Spring Async Guide](CompletableFutureSpringboot.md) — Asynchronous pipeline composition, thread pools, and error handling.
- 🗑️ [JVM Memory & Garbage Collection Guide](GarbageCollector.md) — Heap structure, G1GC, Generational ZGC, and GC tuning flags.
- 📦 [Java Serialization vs Externalizable](Serializable.md) — Binary serialization, `serialVersionUID`, and modern JSON/Protobuf alternatives.
- 📊 [Java Collections Time & Space Complexity](JavaCollecTimeAndSpaceComplexity.md) & [Java Collections Master Guide](JavaCollections.md) — Data structures, internal hashing mechanics, and thread safety.
- 🔢 [XOR Bitwise Operations Deep Dive](Nov%202025/XOR-Deep-Dive-Java.md) — Bitwise math, missing numbers, and single-number problems.
- 🌊 [Java Stream API 2025 Guide](Nov%202025/java-stream-api-interview-guide-2025.md) — Stream operations, collector grouping, mapping, and parallel streams.

---

### 2. Spring Boot 3.x & Framework Engineering
- 🔁 [Spring Bean Lifecycle & Callback Hooks](SpringBeanLifeCycle.md) — Initializing hooks, `BeanPostProcessor`, `@PostConstruct`, and AOP proxies.
- ⚙️ [Spring Boot Auto-Configuration & Startup](SpringbootAutoConfiguration.md) — `AutoConfiguration.imports`, conditional bean loading, and starter development.
- 🌐 [Spring WebClient Guide](WebClient.md) — Reactive non-blocking HTTP requests, error handling, retries, and exchange filters.
- 📜 [HTTP Status Codes in REST API Design](HttpStatusCodesRestAPI.md) — RFC 7807 problem details, 202 Accepted async workflows, and status codes.
- 🔒 [Optimistic vs. Pessimistic Locking](OptimisticPessimisticLocking.md) — Preventing oversells with `@Version`, `@Lock(PESSIMISTIC_WRITE)`, and Redisson.
- 🚀 [Redis Caching & Spring Boot Integration](RedisCache%20Springboot.md) & [Redis Architecture](RedisCache.md) — Eviction policies, cache stampede prevention, Lettuce vs Redisson.
- 📦 [Maven Dependency Scopes & Transitive Resolution](maven.md) — Dependency tree management, exclusions, BOM imports, and scope matrix.
- 🔄 [Spring Batch Complete Course](Nov%202025/Spring-Batch-Complete-Course.md) & [Production Guide](Nov%202025/Spring-Batch-Production-Guide.md) — Chunk processing, partitioners, skip/retry policies.

---

### 3. Distributed Systems, Microservices & Messaging
- 💬 [Apache Kafka Event-Driven Microservices](ApacheKafka.md) — Producer/Consumer architecture, partition reassignment, consumer rebalancing.
- 🧱 [Monolith to Microservices Migration Guide](Monolith%20to%20Microservices.md) — Strangler fig pattern, domain decomposition, event-driven decoupling.
- 🚦 [Rate Limiting & API Gateway Design](Rate%20Limit%20Api%20Gateway.md) — Token Bucket, Leaky Bucket, Sliding Window algorithms, and Spring Cloud Gateway.
- 🛡️ [Spring Boot Microservices Production Scenarios](April%202026/spring-micro-prod-scenarios-deep.md) — Circuit breakers (Resilience4j), distributed tracing, and fault tolerance.

---

### 4. System Design, Object-Oriented Design & SRE
- 🏗️ [System Design Mega Guide](Nov%202025/System-Design-Mega-Guide.md) — High-availability architectures, scaling strategies, and CAP theorem.
- 📐 [LLD Master Guide for Java](Nov%202025/LLD-Master-Guide-Java.md) — Low-Level Design interview problems, class diagrams, and modular designs.
- 🧩 [Design Patterns in Java](Nov%202025/Design-Patterns-Java.md) — Creational, Structural, and Behavioral design patterns with code.
- 📐 [SOLID Principles Guide](Nov%202025/SOLID-Principles-Java.md) — Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.
- 🛑 [OS Signals & Graceful Shutdown Guide](Nov%202025/signals-shutdown-guide.md) — SIGTERM vs SIGKILL, JVM shutdown hooks, and Kubernetes container teardown.
- 🔍 [Splunk RCA Cheatsheet for Spring Boot](Splunk/splunk-cheatsheet-springboot-rca.md) — Log search, SPL queries, thread dump diagnostics, and root cause analysis.

---

### 5. Custom AI Agents & Tooling
- 🤖 [Custom GitHub Copilot AI Agent Suite](custom-ai-agent/README.md) — Enterprise suite of 6 agent personas, modular skills, and caveman token optimization.
- 🧰 [OpenSource AI Agent Toolkit 2026 Guide](OpenSource%20Ai/opensource-ai-agent-toolkit-2026.md) — Building agentic workflows with open-source tools.

---

### 6. Real-World Interview Question Collections
- 📝 [Java Interview Questions Asked 2025 (Part 1)](Java%20Interview%20Questions%20Asked%202025%20P1.md) — Real interview questions asked in 2025 with complete solutions.
- 📝 [Java Interview Questions Asked 2025 (Part 2)](Create%20Java%20Interview%20Questions%20Asked%202025%20P2.md) — Advanced interview scenario questions.
- 💻 [Coding Questions Practice Collection](Coding%20Questions%20May-Jul-2025.md) — Algorithms, string manipulation, run-length decoding, permutations.
- 💼 [Senior Java Backend Interview Master Guide](Nov%202025/senior-java-backend-interview-guide-2025.md) — Senior engineer interview preparation.
- ⏱️ [Time & Space Complexity Reference Guide](Nov%202025/Time-Space-Complexity.md) — Big O notation reference.
