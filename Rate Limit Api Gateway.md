# Rate Limiting Concepts

# Why Rate Limit?
- ### Protect Downstream Services
  Prevent traffic spikes (e.g. “thundering herd”) from overwhelming your back-end.

- ### Fair‐use Enforcement
  Ensure no single client consumes disproportionate resources.

- ### Cost Control
  Many cloud services charge per request; limiting can cap your bill.

- ### Security
  Mitigate brute-force or scraping attacks.


| Algorithm          | Description                                                                             | Characteristics                        |
| ------------------ | --------------------------------------------------------------------------------------- | -------------------------------------- |
| **Fixed Window**   | Count requests in discrete time windows (e.g., per minute).                             | Simple, but suffers “boundary spikes.” |
| **Sliding Window** | Window moves continuously; track requests with timestamps.                              | More accurate, more storage overhead.  |
| **Token Bucket**   | Tokens are added at a fixed rate; a request consumes a token.                           | Allows bursts up to bucket size.       |
| **Leaky Bucket**   | Requests enter queue (“bucket”) at any rate; leak out (are processed) at constant rate. | Smoothes bursts, queue overflow drops. |




# Spring Cloud Gateway Rate Limiting

Spring Cloud Gateway ships with a Redis-backed Token Bucket RateLimiter out of the box.

### Dependencies
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

### Configuration (application.yml)
```yml
spring:
  redis:
    host: localhost
    port: 6379

spring:
  cloud:
    gateway:
      routes:
      - id: user-service
        uri: lb://USER-SERVICE
        predicates:
        - Path=/api/users/**
        filters:
        - name: RequestRateLimiter
          args:
            # Use the RedisRateLimiter
            redis-rate-limiter.replenishRate: 5    # tokens added per second
            redis-rate-limiter.burstCapacity: 10   # max tokens stored
            # Extract key per client (here: user’s API key header)
            key-resolver: "#{@apiKeyResolver}"
```

### KeyResolver Bean
```java
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Component
public class ApiKeyResolverConfig {
  @Bean
  public KeyResolver apiKeyResolver() {
    return exchange -> 
      Mono.justOrEmpty(exchange.getRequest()
        .getHeaders().getFirst("X-API-KEY"))
        .defaultIfEmpty("anonymous");
  }
}
```

- replenishRate: how many tokens per second to add

- burstCapacity: maximum tokens allowed => max burst



### Customizing Response on Limit Exceeded

By default, a 429 (Too Many Requests) is returned. To customize:

```java
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.filter.NettyWriteResponseFilter;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.context.annotation.Bean;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Component
public class RateLimitFallback implements GlobalFilter {

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, 
                           GatewayFilterChain chain) {
    // Check if RATE_LIMITED_ATTR is set
    if (exchange.getAttributeOrDefault(
            ServerWebExchangeUtils.GATEWAY_RATE_LIMITED_ATTR,
            false)) {
      exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
      byte[] bytes = "Rate limit exceeded".getBytes();
      return exchange.getResponse()
                     .writeWith(Mono.just(
                        exchange.getResponse()
                                .bufferFactory()
                                .wrap(bytes)));
    }
    return chain.filter(exchange);
  }

  @Bean
  public OrderedGatewayFilter nettyWriteResponseFilter() {
    return new OrderedGatewayFilter(this, 
      NettyWriteResponseFilter.WRITE_RESPONSE_FILTER_ORDER + 1);
  }
}
```



# AWS API Gateway Rate Limiting

AWS API Gateway uses Usage Plans and API Keys to throttle and quota.

# 3.1 Steps to Configure

## 1. Create or Import API
- In the **API Gateway Console**, select your REST or HTTP API.

## 2. Configure Stages & Deployment
- Deploy your API to a stage (e.g., `prod`).

## 3. Create a Usage Plan
- **Throttle Settings:**
  - **Rate:** (requests per second)
  - **Burst:** (additional capacity)
- **Quota (optional):**
  - Example: 100,000 requests per month

## 4. Associate API Stages
- Select your deployed stage(s).
- Define stage-level throttle overrides if needed.

## 5. Create API Keys
- Generate an API key per client.
- Associate each API key to the Usage Plan.



# Best Practices & Considerations

## Granularity
- Consider per‐user, per‐IP, or per‐API‐key limits based on your use case.

## Dynamic Limits
- Store custom limits per client in a database or DynamoDB for adjustability.

## Distributed Coordination
- Use Redis or DynamoDB to maintain shared counters across gateway instances.

## Graceful Degradation
- Provide helpful error payloads and include `Retry-After` headers.

## Monitoring & Alerts
- Monitor 429 rates using Prometheus/Grafana or CloudWatch.
- Set up alerts to notify when clients are being throttled excessively.



# Testing Strategies

### Unit Tests

- Mock the KeyResolver and RedisRateLimiter.

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
public class RateLimiterIT {
  @Autowired
  private WebTestClient client;

  @Test
  void whenExceedLimit_thenReceive429() {
    for (int i = 0; i < 15; i++) {
      client.get().uri("/api/users")
        .header("X-API-KEY", "test-key")
        .exchange();
    }
    client.get().uri("/api/users")
      .header("X-API-KEY", "test-key")
      .exchange()
      .expectStatus().isEqualTo(HttpStatus.TOO_MANY_REQUESTS);
  }
}

```


## Load Tests
- Use Gatling or JMeter to simulate bursts and measure throttle behavior.
