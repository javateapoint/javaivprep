# HTTP Status Codes in REST API Design

A comprehensive guide to HTTP status codes for RESTful APIs, detailing standard status code categories, production usage scenarios, and Spring Boot 3 implementations.

---

## 1. Quick Reference Matrix

| Status Code | Name | Meaning & Typical Use Case | Example |
| :--- | :--- | :--- | :--- |
| **200** | OK | Request succeeded; body contains resource representation. | `GET /api/v1/users/123` |
| **201** | Created | Resource created; `Location` header contains URI. | `POST /api/v1/orders` |
| **202** | Accepted | Request accepted for asynchronous background processing. | `POST /api/v1/reports` |
| **204** | No Content | Request succeeded; no body returned (e.g. deletion). | `DELETE /api/v1/sessions/abc` |
| **400** | Bad Request | Client error (malformed JSON, path/query validation failed). | `POST /api/v1/users` with invalid email |
| **401** | Unauthorized | Authentication required or token invalid/expired. | `GET /api/v1/profile` without Bearer token |
| **403** | Forbidden | Client authenticated but lacks required role/permission. | Non-admin calling `DELETE /api/v1/users` |
| **404** | Not Found | Target resource or URI path does not exist. | `GET /api/v1/products/9999` |
| **409** | Conflict | Request conflicts with current state (e.g. duplicate key). | `POST /api/v1/users` with existing username |
| **415** | Unsupported Media Type | Request payload format unsupported (e.g. XML sent to JSON API). | `POST /api/v1/data` with `Content-Type: text/plain` |
| **422** | Unprocessable Entity | Syntax is valid, but semantic business rules failed. | `POST /api/v1/orders` with out-of-stock SKU |
| **429** | Too Many Requests | Rate limit exceeded. | Exceeding 100 requests/minute |
| **500** | Internal Server Error | Unhandled server-side exception or bug. | NullPointerException in backend service |
| **502** | Bad Gateway | Upstream server returned invalid response. | Reverse proxy receiving invalid HTTP response |
| **503** | Service Unavailable | Service down for maintenance or overloaded. | High load / circuit breaker OPEN |
| **504** | Gateway Timeout | Upstream service did not respond within deadline. | Gateway timed out calling legacy service |

---

## 2. Deep Dive: 202 Accepted in Asynchronous Architectures

When processing long-running operations (such as video rendering, batch report generation, or complex event-driven workflows), returning `200 OK` or `201 Created` synchronously blocks the client connection. Use **`202 Accepted`** with a `Location` header pointing to a polling/status endpoint.

```java
package com.example.demo.controller;

import com.example.demo.dto.OrderRequestDto;
import com.example.demo.model.Order;
import com.example.demo.service.OrderService;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService orderService;
    private final ApplicationEventPublisher eventPublisher;

    public OrderController(OrderService orderService, ApplicationEventPublisher eventPublisher) {
        this.orderService = orderService;
        this.eventPublisher = eventPublisher;
    }

    @PostMapping
    public ResponseEntity<Void> createOrderAsync(@RequestBody OrderRequestDto dto) {
        // 1. Initial validation & light persistence
        Order order = orderService.createPendingOrder(dto);

        // 2. Publish event to Kafka/RabbitMQ or Spring Async Event Bus
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));

        // 3. Immediately return 202 Accepted with status monitoring URI
        URI statusUri = URI.create("/api/v1/orders/" + order.getId() + "/status");
        return ResponseEntity
                .accepted()
                .location(statusUri)
                .build();
    }
}
```

---

## 3. RFC 7807 / ProblemDetails Error Responses in Spring Boot 3

Spring Boot 3 natively supports **RFC 7807 (Problem Details for HTTP APIs)** via `ProblemDetail` and `@RestControllerAdvice`.

```java
package com.example.demo.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;

import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setProperty("timestamp", System.currentTimeMillis());
        return problem;
    }

    @ExceptionHandler(BusinessValidationException.class)
    public ProblemDetail handleValidation(BusinessValidationException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
        problem.setTitle("Unprocessable Entity");
        problem.setType(URI.create("https://api.example.com/errors/validation-failed"));
        return problem;
    }
}
```
