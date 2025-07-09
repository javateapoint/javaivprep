## 1. How would you approach migrating a legacy CRM system to a Java-based platform?
I would begin with a discovery phase to map out existing features, data models, integrations, and pain points. This involves stakeholder workshops, code inventory, and performance profiling.

I’d define a target architecture based on domain-driven design, choosing between a modular monolith or a microservices approach. Key considerations include scalability requirements, team size, and operational complexity.

I’d plan data migration using an ETL pipeline: extract from the legacy database, transform to fit the new schema, and load into the Java platform. Parallel runs and data reconciliation checks ensure consistency.

I’d adopt the strangler pattern to incrementally replace legacy modules. Each cut-over step would include automated tests, rollback scripts, and user acceptance testing.

## 2. Design an Audit Logging Module
I’d centralize audit logging as a cross-cutting concern using Spring AOP. Define pointcuts for service or repository layer operations to capture create, update, and delete events.

The audit record schema would include

timestamp

user identity

operation type

entity name and identifier

payload diff or context

I’d store logs in a dedicated audit table or an append-only data store like Elasticsearch for fast querying. Asynchronous logging using a message queue (Kafka or RabbitMQ) ensures minimal latency in the main transaction.

I’d also expose an API for audit retrieval with filtering, pagination, and RBAC checks to secure sensitive history data.

## 3. Versioning Strategy for REST APIs
I favor URI versioning (e.g., /api/v1/...) for clear client contracts and caching friendliness. This approach coexists with semantic versioning of the service.

When introducing backward-incompatible changes, I’d publish a new major version and maintain the old endpoints until all clients migrate. Non-breaking changes (new optional fields) can live within the same version.

I’d document each version in OpenAPI (Swagger) and maintain separate specs per version. Automated integration tests validate behavior against each spec.

## 4. What are different types of classloaders in Java and when are they used?
The Java ClassLoader hierarchy includes:

Bootstrap ClassLoader: loads core JDK classes (rt.jar).

Extension ClassLoader: loads classes from the jre/lib/ext directory.

Application (System) ClassLoader: loads classes from the application’s classpath.

Custom or third-party ClassLoaders (e.g., in application servers or OSGi containers) enable dynamic loading and isolation of modules. I’ve used custom loaders to support hot-reload during development.

## 5. Design a Role-Based Access Control (RBAC) System
I’d model three core entities: User, Role, and Permission. A Role aggregates permissions, and a User can have multiple roles.

Key components:

Authentication layer issues a JWT containing roles and permissions.

Authorization filter validates the token and enforces permission checks per endpoint.

Admin UI to manage role-permission mappings and user assignments.

Permissions would be fine-grained (e.g., customer:create, order:approve) and stored in a database table for dynamic changes. I’d leverage Spring Security’s @PreAuthorize annotations to enforce checks at the service layer.

## 6. How would you implement a retry mechanism in Spring Boot?
I’d use Spring Retry or Resilience4j. With Spring Retry, annotate methods with @Retryable and configure backoff parameters:

java
@Retryable(
  value = {TransientDataAccessException.class}, 
  maxAttempts = 5, 
  backoff = @Backoff(delay = 2000, multiplier = 2))
public void callRemoteService() { … }
Combine with @Recover to define fallback logic. Resilience4j offers similar capabilities with more fine-tuned circuit breaker and rate limiter integration.

## 7. How does dependency injection work in Spring?
Spring uses the Inversion of Control (IoC) container to manage object lifecycles and dependencies. Beans are registered via annotations (@Component, @Service, @Repository, or @Configuration methods) or XML.

At startup, the container scans and instantiates beans, resolves their constructor or setter dependencies, and wires them together. This decouples implementation from instantiation, making classes easier to test and swap.

## 8. How do you secure a REST API using OAuth2?
I’d use Spring Security OAuth2 Resource Server for token validation. Clients obtain JWTs or opaque tokens from an Authorization Server (e.g., Keycloak).

The Resource Server config looks like:

yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/
Endpoints are secured via @PreAuthorize("hasAuthority('SCOPE_read')") or HTTP security matchers. Refresh tokens and scopes manage session and access granularity.

## 9. What’s the difference between JPA and Hibernate?
JPA is the Java specification for object-relational mapping, defining interfaces like EntityManager, CriteriaBuilder, and annotations. Hibernate is a specific implementation of JPA and adds proprietary features like @NaturalId, StatelessSession, and advanced caching.

Using JPA ensures portability across providers (EclipseLink, OpenJPA), while Hibernate’s extensions can optimize specific use cases.

## 10. What is Spring AOP? How have you used it in logging or security?
Spring AOP provides method-level interception using proxies or AspectJ weaving. I’ve used it to implement:

Audit logging: intercept service methods, capture input/output, and write structured logs asynchronously.

Security checks: enforce dynamic permissions based on method arguments without scattering if statements across services.

This abstraction keeps business logic clean and centralizes cross-cutting concerns.

## 11. Explain optimistic vs pessimistic locking. When to use each?
Optimistic locking assumes minimal contention: each entity carries a version field (@Version). Upon update, the version is compared, and a OptimisticLockException is thrown if it changed. This suits high-read, low-write scenarios.

Pessimistic locking acquires a database lock (SELECT … FOR UPDATE) at transaction start. This prevents concurrent updates but can reduce throughput. It’s ideal when contention is high or stale reads are unacceptable (e.g., booking systems).

## 12. Design a High Availability Payment System
I’d build idempotent, stateless payment microservices behind a load balancer. Key components:

API gateway for routing and rate limiting.

Payment processor cluster with health checks and autoscaling.

Shared event bus (Kafka) for payment events and reconciliation.

Primary and standby databases with automated failover.

Data idempotency is ensured via unique transaction IDs and deduplication tables. I’d run multiple instances across availability zones to tolerate zone failures.

## 13. How would you design a presence service for a Unified Communication system?
I’d use a lightweight socket-based service (WebSocket or MQTT) to track user connections. Each client registers on login and sends heartbeat messages.

A distributed in-memory data store (Redis) holds live presence states and supports pub/sub to notify interested services. Presence changes are broadcast to subscribers in real time.

## 14. How would you safely handle concurrent payment transactions?
I’d implement idempotency keys for each transaction request to prevent double-charges. At the database level, I’d use optimistic locking on account balance records.

In code:

Read account with version.

Apply debit or credit.

Attempt update; on version mismatch, retry or abort.

Complement with distributed locks (Redisson) if cross-service coordination is needed.

## 15. What are your strategies for database schema versioning in microservices? (Flyway, Liquibase, etc.)
I prefer Flyway for its simplicity: versioned SQL migrations stored alongside code and executed on startup. Each microservice owns its own migration history table.

Liquibase offers XML/JSON changelogs, more flexibility, and rollback support. I choose it when dynamic change sets or advanced refactoring is required.

I enforce CI pipeline checks to prevent migration conflicts and run migrations in a staging environment before production.

## 16. How do you deploy Spring Boot apps to GCP?
I package the app as a Docker container and push it to Google Container Registry (GCR). I deploy to GKE using Helm charts or Terraform, defining Kubernetes Deployments, Services, and Ingress.

For serverless, I use Cloud Run:

bash
gcloud run deploy my-app --image gcr.io/project-id/my-app --platform managed
I configure autoscaling, VPC connectors, and managed SSL via Cloud Load Balancing.

## 17. How do you manage secrets securely in GCP?
I use Secret Manager to store API keys, database credentials, and certificates. Applications retrieve secrets at runtime using the Google Cloud client libraries with IAM roles scoped to least privilege.

For Kubernetes, I mount secrets as environment variables or projected volumes via CSI driver. I rotate secrets programmatically and revoke old versions.

## 18. What CI/CD pipeline do you follow?
I follow these stages:

Build & Unit Test: Maven build in Jenkins or GitHub Actions, run unit tests and static code analysis (SonarQube).

Integration & Containerization: Build Docker image, run integration tests in a disposable environment.

Security & Compliance: Scan images for vulnerabilities (Trivy) and validate infrastructure as code.

Staging Deployment: Deploy to staging cluster, run end-to-end and performance tests.

Production Promotion: Manual approval gates, blue-green or canary deployment to production.

## 19. How do you manage memory leaks in a long-running Java application?
I start by enabling heap dumps on OutOfMemoryError and use tools like Eclipse MAT or VisualVM to analyze retained objects.

Common leak sources include unbounded caches, ThreadLocal misuse, and unclosed JDBC connections. I fix leaks by:

Setting proper cache eviction policies.

Explicitly removing ThreadLocal variables.

Ensuring try-with-resources for I/O and JDBC.

I also enable continuous profiling (e.g., with async-profiler) to catch anomalies before production issues arise.

## 20. How do you profile and optimize a slow Spring Boot application?
I’d attach a profiler (YourKit, async-profiler) to identify CPU hotspots, thread contention, and garbage collection pauses.

I’d analyze:

Endpoint latencies with an APM tool (New Relic, Elastic APM).

Database query performance using slow-query logs and explain plans.

JVM GC logs to tune heap size and GC algorithm (G1 vs. ZGC).

Based on findings, I’d optimize or rewrite inefficient algorithms, add indexes, or refactor blocking calls to reactive streams if necessary.

## 21. How do you resolve design conflicts among senior developers?
I encourage a data-driven discussion: evaluate trade-offs (performance, scalability, maintainability), and prototype critical paths where feasible.

If opinions remain divided, we involve a neutral architect or refer to company standards. I document the chosen approach and rationale to maintain alignment.

## 22. How do you ensure code quality in a team of 10+ developers?
I establish a shared style guide, enforce it via IDE plugins and CI checks (Checkstyle, SpotBugs).

Code reviews are mandatory, focusing on correctness, clarity, and test coverage. We measure quality metrics (code coverage, cyclomatic complexity) and track them on dashboards. Regular guild sessions share best practices and emerging patterns.

## 23. How do you mentor junior developers?
I pair-program on real tasks, encouraging questions and explaining decision rationales. I assign bite-sized improvements (refactoring, adding tests) and review their work with constructive feedback.

I schedule regular 1:1s to discuss career goals, technical challenges, and recommend learning resources. This tailored approach accelerates their growth and confidence.
