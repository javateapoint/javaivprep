---
title: "RestTemplate vs WebClient Comparison"
date: "2025-06-16"
---

# RestTemplate vs WebClient Comparison

This document provides a quick reference on the distinctions between **RestTemplate** and **WebClient**, two popular HTTP clients in Spring.

## Overview

- **RestTemplate:**  
  A traditional, blocking, synchronous HTTP client built on the Servlet API. It's widely used in legacy applications or scenarios where a blocking call model is sufficient.

- **WebClient:**  
  A modern, non-blocking, asynchronous HTTP client that is part of Spring WebFlux. It is ideal for microservices and high concurrency applications, as it returns reactive types (`Mono` and `Flux`).

## Comparison Table

| **Aspect**                | **RestTemplate**                                               | **WebClient**                                                  |
|---------------------------|----------------------------------------------------------------|----------------------------------------------------------------|
| **Paradigm**              | Blocking, synchronous                                          | Non-blocking, asynchronous (reactive)                          |
| **Thread Model**          | One-thread-per-request; ties up a thread until completion      | Uses a small number of threads; releases them during I/O wait    |
| **Built On**              | Traditional Servlet API                                        | Spring WebFlux and Project Reactor                             |
| **Return Type**           | Direct response object (e.g., `ResponseEntity<T>`)             | Reactive types: `Mono<T>` for single values, `Flux<T>` for collections  |
| **Error Handling**        | Uses try-catch blocks for exceptions                           | Uses reactive operators like `onErrorResume()`, `onErrorReturn()` |
| **Use Cases**             | Legacy systems, simple monolithic applications                 | New microservices, high concurrency systems, streaming APIs    |
| **Boilerplate Code**      | More verbose, imperative API style                             | Concise, declarative, functional API style                     |
| **Future Direction**      | In maintenance mode (for backward compatibility)               | Actively developed with ongoing enhancements                   |
| **Default Behavior**      | Synchronous HTTP calls                                         | Asynchronous, non-blocking HTTP calls                           |
| **Integration**           | Suited for traditional Spring MVC applications                 | Integrates seamlessly with reactive, non-blocking stacks         |

## Summary

- **RestTemplate** is best employed in simple, blocking scenarios or in legacy applications that don't require reactive capabilities.  
- **WebClient** is recommended for new development, particularly when building scalable microservices, or when dealing with asynchronous or streaming data.

---
---
title: "Real-Time Examples and Use Cases: RestTemplate vs WebClient"
author: "AI Copilot"
date: "2025-06-16"
---



# Real-Time Examples and Use Cases: RestTemplate vs WebClient



## Overview

- **RestTemplate** is a blocking HTTP client suitable for scenarios with low to moderate concurrent usage and legacy applications.
- **WebClient** is a non-blocking, asynchronous client ideal for high-concurrency environments, microservices, and real-time streaming data.


---

## Real-World Use Cases

- **Aggregating External Data:**  
  A microservice might need to combine data from multiple external sources. For example, requesting weather details and news headlines to display on a dashboard.

- **Legacy Application Integration:**  
  An existing application built on Spring MVC may use RestTemplate to interact synchronously with other internal microservices.

- **High-Concurrency Microservices:**  
  When building a reactive microservice that receives many simultaneous requests (e.g., live market data or notifications), WebClient's non-blocking behavior helps manage resources efficiently.

- **Streaming Data / Long-Running Connections:**  
  WebClient supports reactive streams, making it well-suited for scenarios like subscribing to server-sent events (SSE) or WebSocket data streams.

---

## Example 1: Consuming a REST API with RestTemplate

### Scenario

Imagine we have a microservice that needs to consume a public "User" API to fetch user details (e.g., from a user management system).

### Code

**1. Configure and Use RestTemplate:**

```java
// UserDto.java
package com.example.demo.dto;

public class UserDto {
    private Long id;
    private String username;
    private String email;

    // Constructors, getters, setters, and toString() omitted for brevity
    public UserDto() { }

    public UserDto(Long id, String username, String email) {
        this.id = id;
        this.username = username;
        this.email = email;
    }
    
    // Getters and setters...
    
    @Override
    public String toString() {
        return "UserDto{" + "id=" + id + ", username='" + username + '\'' + ", email='" + email + '\'' + '}';
    }
}
```


```java

// RestTemplateConfig.java
package com.example.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

  @Bean
  public RestTemplate restTemplate() {
    return new RestTemplate();
  }
}

```

```java
// UserService.java
package com.example.demo.service;

import com.example.demo.dto.UserDto;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class UserService {

  @Autowired
  private RestTemplate restTemplate;

  // Example endpoint URL. In real scenarios, consider using service discovery.
  private static final String USER_API_URL = "https://jsonplaceholder.typicode.com/users/{id}";

  public UserDto getUserById(Long id) {
    ResponseEntity<UserDto> responseEntity = restTemplate.getForEntity(USER_API_URL, UserDto.class, id);
    return responseEntity.getBody();
  }
}
```


```java
// UserController.java
package com.example.demo.controller;

import com.example.demo.dto.UserDto;
import com.example.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/rest")
public class UserController {

  @Autowired
  private UserService userService;

  @GetMapping("/users/{id}")
  public UserDto getUser(@PathVariable Long id) {
    return userService.getUserById(id);
  }
}
```

# Example 2: Consuming a REST API with WebClient

Consider the same requirement but within a microservice that must scale to handle hundreds or thousands of concurrent requests. We'll use WebClient for non-blocking, reactive interaction with the external API.

```java
// WebClientConfig.java
package com.example.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

  @Bean
  public WebClient webClient() {
    return WebClient.builder().baseUrl("https://jsonplaceholder.typicode.com").build();
  }
}
```


```java
// ReactiveUserService.java
package com.example.demo.service;

import com.example.demo.dto.UserDto;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Service
public class ReactiveUserService {

  @Autowired
  private WebClient webClient;

  public Mono<UserDto> getUserById(Long id) {
    return webClient.get()
                    .uri("/users/{id}", id)
                    .retrieve()
                    .bodyToMono(UserDto.class)
                    .onErrorResume(error -> {
                        // Log error or perform fallback logic
                        return Mono.empty();
                    });
  }
}
```

```java
// ReactiveUserController.java
package com.example.demo.controller;

import com.example.demo.dto.UserDto;
import com.example.demo.service.ReactiveUserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api/webclient")
public class ReactiveUserController {

  @Autowired
  private ReactiveUserService reactiveUserService;

  @GetMapping("/users/{id}")
  public Mono<UserDto> getUser(@PathVariable Long id) {
    return reactiveUserService.getUserById(id);
  }
}
```

### Explanation
- ReactiveUserController exposes an endpoint using reactive types.

- ReactiveUserService uses WebClient to perform a non-blocking GET request.

- The result is a Mono<UserDto> that is returned without tying up a thread during I/O operations.

- Error handling is performed using reactive operators (onErrorResume).

This use case is ideal for high-concurrency scenarios where efficient resource utilization is essential.

