---
name: security-observability-sre
description: >
  DevSecOps, Database Lifecycle (Flyway/HikariCP), and Production Observability/SRE 
  reference skill for GitHub Copilot in IntelliJ.
---

# DevSecOps, DB Migrations & SRE Skill

## Purpose
Provides technical guidance across security, database migration management, connection pool tuning, and production observability for Spring Boot microservices.

---

## 1. DevSecOps & Security Engineering

### A. Spring Security 6.x Stateless API Standards
- Use SecurityFilterChain bean definition with OAuth2 Resource Server / JWT validation.
- Enforce strict authorization rules:
  ```java
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
      return http
          .csrf(AbstractHttpConfigurer::disable)
          .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/actuator/health/**", "/v3/api-docs/**").permitAll()
              .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
              .anyRequest().authenticated()
          )
          .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))
          .build();
  }
  ```

### B. OWASP Defensive Checklist
1. **Broken Object Level Authorization (BOLA)**: Always check `userId` against authenticated SecurityContext JWT claims before returning entity records.
2. **Sensitive Data Exposure**: Mask PII/PCI fields (credit card, SSN) in Logback MDC using a custom masking pattern or converter.
3. **SQL Injection Defense**: Never concatenate strings in Spring Data JPA native queries; use named parameters (`:paramId`).

---

## 2. Database Lifecycle & Migrations (Flyway / Liquibase)

### A. Flyway Migration Naming Conventions
- Path: `src/main/resources/db/migration/`
- Naming format: `V<Version>__<Description>.sql` (e.g., `V1__init_order_schema.sql`, `V2__add_index_on_customer_id.sql`).
- **Never modify an existing applied migration file**: Always write a new incremented migration file (`V3__...sql`) to alter tables or indexes.

### B. N+1 Query Elimination & HikariCP Tuning
- **Eliminate N+1**: Use `@EntityGraph(attributePaths = {"items"})` or DTO projections (`interface OrderSummaryProjection`) instead of lazy collection loops.
- **HikariCP Production Settings**:
  ```yaml
  spring:
    datasource:
      hikari:
        maximum-pool-size: 20
        minimum-idle: 10
        idle-timeout: 300000
        max-lifetime: 1800000
        connection-timeout: 30000
        leak-detection-threshold: 2000
  ```

---

## 3. Production Observability & SRE Standards

### A. Structured Logging & MDC Propagation
- Configure Logback for ECS JSON layout:
  ```xml
  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>traceId</includeMdcKeyName>
      <includeMdcKeyName>spanId</includeMdcKeyName>
      <includeMdcKeyName>userId</includeMdcKeyName>
  </encoder>
  ```
- **Async Thread MDC Propagation**: Wrap `@Async` or ThreadPool Executors with `TaskDecorator` to copy MDC context across thread boundaries.

### B. Kubernetes Probes & Graceful Shutdown
```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  docker:
    compose:
      enabled: false

management:
  endpoints:
    web:
      exposure:
        include: "health,prometheus,metrics,info"
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when_authorized
```
- K8s Readiness Probe: `/actuator/health/readiness`
- K8s Liveness Probe: `/actuator/health/liveness`
