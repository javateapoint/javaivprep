# Resilience4j, Distributed Tracing & Observability Master Guide

A production-grade reference guide for Senior Java Engineers (7+ YOE) covering **Resilience4j Circuit Breakers**, **Bulkhead & Rate Limiter isolation**, **Exponential Backoff with Jitter**, **W3C Trace Context propagation**, **Micrometer Tracing + OpenTelemetry**, and **MDC context preservation across async boundaries**.

---

## 1. Resilience4j Circuit Breaker Architecture

Circuit Breakers protect upstream microservices from cascading failures when downstream dependencies slow down or fail.

```text
               ┌────────────────────────────────────────────────────────┐
               │                                                        │
               ▼                                                        │
      ┌─────────────────┐       Failure Rate > Threshold       ┌────────────────┐
      │     CLOSED      │ ───────────────────────────────────► │      OPEN      │
      │ (Normal State)  │                                      │ (Fast Failure) │
      └─────────────────┘                                      └───────┬────────┘
               ▲                                                       │
               │                                                       │ Wait Duration
               │                  Success Rate >= Threshold            │ Expired
               └────────────────────────────────────────────── ┌───────▼────────┐
                                                               │   HALF_OPEN    │
                                                               │ (Trial Probes) │
                                                               └────────────────┘
```

### Circuit Breaker States
1. **`CLOSED`**: Requests flow normally. Failures are recorded in a sliding window.
2. **`OPEN`**: Failure threshold exceeded. **All requests fail fast immediately** (throwing `CallNotPermittedException`) without calling downstream service.
3. **`HALF_OPEN`**: After `waitDurationInOpenState` expires, a configurable trial number of requests are permitted through to probe downstream recovery.

### 1.1 Production Spring Boot 3 `application.yml` Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 100        # Evaluate last 100 requests
        minimum-number-of-calls: 20     # Minimum requests before calculating failure rate
        failure-rate-threshold: 50       # Open breaker if 50% calls fail
        slow-call-rate-threshold: 80     # Open breaker if 80% calls exceed slow-call duration
        slow-call-duration-threshold: 2s # Calls taking > 2s considered "slow"
        wait-duration-in-open-state: 10s # Stay OPEN for 10 seconds before trial HALF_OPEN
        permitted-number-of-calls-in-half-open-state: 10 # Send 10 trial requests
        automatic-transition-from-open-to-half-open-enabled: true
```

---

## 2. Bulkhead & Rate Limiter Isolation Patterns

### 2.1 Bulkhead: Preventing Resource Exhaustion
Isolates concurrent requests to a specific service component so that a slow dependency cannot consume all Tomcat HTTP threads.

- **Semaphore Bulkhead**: Restricts the maximum number of concurrent executions (lightweight, same-thread).
- **ThreadPool Bulkhead**: Executes requests in a dedicated bounded thread pool with a separate queue.

```yaml
resilience4j:
  bulkhead:
    instances:
      thirdPartyApi:
        max-concurrent-calls: 25  # Limit concurrent calls to 25
        max-wait-duration: 500ms  # Wait up to 500ms for available slot
```

### 2.2 Retry with Exponential Backoff + Jitter

Plain retries without backoff or randomness cause **Thundering Herds** (amplifying backend load). Adding **Jitter** adds random variance to backoff intervals.

$$\text{Backoff Interval} = \text{Initial Delay} \times (\text{Multiplier}^{\text{Attempt}}) \pm \text{Random Jitter}$$

```java
package com.example.demo.client;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Component;

@Component
public class ExternalPaymentClient {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    public String executePayment(PaymentRequest request) {
        // External REST call
        return restTemplate.postForObject("https://api.payments.com/charge", request, String.class);
    }

    // Fallback executed when Breaker is OPEN or Retries are exhausted
    public String paymentFallback(PaymentRequest request, Throwable t) {
        // Return degraded fallback or write to pending queue
        return "DEGRADED_PAYMENT_QUEUED";
    }
}
```

---

## 3. Distributed Tracing & W3C Trace Context

In a microservices architecture, a single user request traverses multiple services. **Distributed Tracing** correlates logs across services using unique trace identifiers.

### 3.1 W3C `traceparent` Header Specification

The W3C standard HTTP header passed across microservice boundaries:

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                │
             │  └─ Trace ID (32 hex chars)       └─ Parent Span   └─ Trace Flags (01 = Sampled)
             └─ Version                            ID (16 hex)
```

- **`Trace ID`**: Globality unique ID for the entire user request transaction.
- **`Span ID`**: Identifies work done within a specific microservice boundary.

---

## 4. Micrometer Tracing & MDC Context Propagation across Async Boundaries

Spring Boot 3 replaced Spring Cloud Sleuth with **Micrometer Tracing** (backed by OpenTelemetry or Zipkin).

### 4.1 Production Maven Dependencies

```xml
<dependencies>
    <!-- Micrometer Tracing Core -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <!-- OpenTelemetry Exporter -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
</dependencies>
```

### 4.2 Standard Logback Pattern for Trace Correlation

File: `src/main/resources/logback-spring.xml`

```xml
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{traceId:-},%X{spanId:-}] %-5level %logger{36} - %msg%n</pattern>
```

### 4.3 Propagating MDC Tracing Context across `@Async` Threads

When delegating tasks to a background thread pool, standard SLF4J `MDC` (Mapped Diagnostic Context) is lost because `MDC` relies on `ThreadLocal`.

#### 🟢 Solution: `TaskDecorator` for MDC Context Propagation

```java
package com.example.demo.config;

import org.slf4j.MDC;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskDecorator;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.Map;

@Configuration
public class AsyncTracingConfig {

    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setTaskDecorator(new MdcTaskDecorator()); // Attach MDC Decorator!
        executor.initialize();
        return executor;
    }

    // Decorator copies MDC map from caller thread to worker thread!
    public static class MdcTaskDecorator implements TaskDecorator {
        @Override
        public Runnable decorate(Runnable runnable) {
            Map<String, String> contextMap = MDC.getCopyOfContextMap();
            return () -> {
                try {
                    if (contextMap != null) {
                        MDC.setContextMap(contextMap);
                    }
                    runnable.run();
                } finally {
                    MDC.clear(); // Prevent MDC leakage across reused pool threads!
                }
            };
        }
    }
}
```

---

## 💡 Senior Interview Summary Tips

1. **Always wrap Retries inside Circuit Breakers** so that retries are stopped instantly when the Breaker flips to `OPEN`.
2. **Add Jitter to Exponential Backoff** to prevent thundering herd spikes on downstream service recovery.
3. **Use a custom `TaskDecorator`** to copy `MDC` context maps across thread pool boundaries in Spring `@Async` methods.
