
# Async Order Summary with `CompletableFuture` in Spring Boot

A production-ready example of using `CompletableFuture` in a Spring Boot app to fetch and aggregate data in parallel—fully non-blocking, with timeouts, fallbacks, and a dedicated thread pool.

---

## Table of Contents

1. [Introduction](#introduction)  
2. [Asynchronous Service Layer](#asynchronous-service-layer)  
   - [Async Configuration](#async-configuration)
   - [Service Implementation](#service-implementation)
   - [Non-Blocking REST Endpoint](#non-blocking-rest-endpoint)
   - [Key Concepts & Explanations](#key-concepts--explanations)
   - [Why This Is Production-Ready](#why-this-is-production-ready)
   - [Best Practices](#best-practices)

---

## Introduction

Modern microservices often need to orchestrate many downstream calls—user info, orders, pricing, etc.—and combine the results into a single response.  
Using Java 8’s `CompletableFuture` you can:

- Run I/O tasks in parallel  
- Apply timeouts and fallbacks per stage  
- Keep servlet threads free until all work is done  
- Avoid extra dependencies on Reactor/RxJava for simple pipelines  

---

## Asynchronous Service Layer

Below is everything you need under one roof: configuration, implementation, endpoint, and deep dive explanations.

### Async Configuration

Define your own bounded thread pool rather than using the common ForkJoinPool. This prevents blocking I/O threads from starving CPU-bound tasks.

```java
// src/main/java/com/example/config/AsyncConfig.java
package com.example.config;

import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler;
import org.springframework.context.annotation.*;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Bean(name = "asyncExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);            // Minimum number of threads
        executor.setMaxPoolSize(50);             // Maximum number of threads
        executor.setQueueCapacity(500);          // Tasks queued before rejecting
        executor.setKeepAliveSeconds(60);        // Idle thread timeout
        executor.setThreadNamePrefix("Async-");
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            System.err.printf(
              "Async error in %s.%s(): %s%n",
              method.getDeclaringClass().getSimpleName(),
              method.getName(),
              ex.getMessage()
            );
    }
}
````

### Service Implementation

Here we define three async methods—fetch user info, orders, and price map—with per-call timeouts and fallback logic. Then we fan-out those calls and fan-in the results.

```java
// src/main/java/com/example/service/OrderAggregationService.java
package com.example.service;

import com.example.client.*;
import com.example.model.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;
import java.util.stream.Collectors;

import static java.util.concurrent.TimeUnit.SECONDS;

@Service
public class OrderAggregationService {

    private final UserClient userClient;
    private final OrderClient orderClient;
    private final PriceClient priceClient;
    private final Executor executor;

    public OrderAggregationService(UserClient userClient,
                                   OrderClient orderClient,
                                   PriceClient priceClient,
                                   @Qualifier("asyncExecutor") Executor executor) {
        this.userClient  = userClient;
        this.orderClient = orderClient;
        this.priceClient = priceClient;
        this.executor    = executor;
    }

    public CompletableFuture<UserInfo> getUserInfo(Long userId) {
        return CompletableFuture
          .supplyAsync(() -> userClient.fetchUser(userId), executor)
          .orTimeout(2, SECONDS)
          .exceptionally(ex -> UserInfo.defaultFor(userId));
    }

    public CompletableFuture<List<Order>> getRecentOrders(Long userId) {
        return CompletableFuture
          .supplyAsync(() -> orderClient.fetchOrders(userId), executor)
          .orTimeout(3, SECONDS)
          .exceptionally(ex -> Collections.emptyList());
    }

    public CompletableFuture<Map<Long, BigDecimal>> getPriceMap(List<Order> orders) {
        List<Long> ids = orders.stream()
                               .map(Order::getProductId)
                               .collect(Collectors.toList());

        return CompletableFuture
          .supplyAsync(() -> priceClient.fetchPrices(ids), executor)
          .orTimeout(2, SECONDS)
          .exceptionally(ex -> Collections.emptyMap());
    }

    public CompletableFuture<OrderSummary> aggregate(Long userId) {
        // Fan-out: start user + order fetch in parallel
        CompletableFuture<UserInfo>    userFut   = getUserInfo(userId);
        CompletableFuture<List<Order>> ordersFut = getRecentOrders(userId);

        // Fan-in: once orders arrive, fetch prices then combine with user
        return ordersFut
          .thenCompose(orders ->
            getPriceMap(orders)
              .thenCombine(userFut, (priceMap, user) ->
                new OrderSummary(user, orders, priceMap)
              )
          );
    }
}
```

### Non-Blocking REST Endpoint

Spring MVC auto-detects a `CompletableFuture<?>` return type and completes the HTTP response when the future completes—freeing up the servlet thread immediately.

```java
// src/main/java/com/example/controller/OrderSummaryController.java
package com.example.controller;

import com.example.model.OrderSummary;
import com.example.service.OrderAggregationService;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.util.concurrent.CompletableFuture;

@RestController
@RequestMapping("/api/summary")
public class OrderSummaryController {

    private final OrderAggregationService service;

    public OrderSummaryController(OrderAggregationService service) {
        this.service = service;
    }

    @GetMapping("/{userId}")
    public CompletableFuture<ResponseEntity<OrderSummary>> getSummary(@PathVariable Long userId) {
        return service.aggregate(userId)
                      .thenApply(ResponseEntity::ok)
                      .exceptionally(ex ->
                          ResponseEntity
                            .status(HttpStatus.SERVICE_UNAVAILABLE)
                            .body(OrderSummary.empty(userId))
                      );
    }
}
```

### Key Concepts & Explanations

#### ThreadPoolTaskExecutor & Executor

* A Spring wrapper around `java.util.concurrent.ThreadPoolExecutor`.
* `corePoolSize` / `maxPoolSize` / `queueCapacity` tune throughput vs. latency.
* Naming threads (`setThreadNamePrefix`) helps in tracing and logging.

#### CompletableFuture Basics

* Implements `CompletionStage<T>` + `Future<T>`.
* Non-blocking: you register callbacks (`thenApply`, `thenAccept`) instead of blocking on `get()`.

#### Timeouts & Fallbacks with `orTimeout()` and `exceptionally()`

* `orTimeout(long, TimeUnit)` auto-completes the future exceptionally if it doesn’t finish in time (Java 9+).
* `exceptionally(fn)` lets you recover or fall back to a default value when exceptions occur.

#### Fan-Out / Fan-In Patterns

* **Fan-Out:** trigger multiple independent async calls in parallel (e.g., user + orders).
* **Fan-In:** combine their results (e.g., price lookup depends on orders, then merge with user).
* Use `thenCompose` for flat-mapping an async stage, `thenCombine` to join two futures.

#### Spring MVC Auto-Detection of `CompletableFuture`

* Returning `CompletableFuture<T>` from a `@RestController` method delegates response handling to Spring’s `DeferredResult` under the hood.
* The servlet thread is released immediately; when the future completes, Spring writes the body.

### Why This Is Production-Ready

1. **Dedicated Executor**
   Keeps blocking I/O off the common pool and prevents starvation.

2. **Per-Call Timeouts & Fallbacks**
   Each external dependency has bounded latency and graceful defaults on failure.

3. **Parallel Orchestration**
   Maximizes throughput by overlapping remote calls.

4. **Error Isolation**
   Failures in one stage don’t kill the entire pipeline—each stage handles its own exceptions.

5. **True Non-Blocking End-to-End**
   Servlet container threads are never tied up waiting for I/O.

### Best Practices

* Always use a custom, bounded Executor for I/O bound tasks.
* Instrument thread-pool metrics (queue depth, active threads) with Micrometer.
* Propagate tracing spans across async boundaries with Spring Cloud Sleuth or OpenTelemetry.
* Choose between `handle()`, `whenComplete()`, and `exceptionally()` based on whether you need result-only, exception-only, or unified handling.
* Avoid deep nesting—structure your pipeline with flat `thenCompose`/`thenCombine` calls.


```

