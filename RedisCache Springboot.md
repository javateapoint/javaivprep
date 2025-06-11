# Redis Cache in Spring Boot


---

## 1. Why Cache with Redis?

- **Performance & Latency**  
  Offload expensive reads (DB queries, remote calls) to an in-memory store → sub-millisecond access.

- **Throughput**  
  Reduce database load under high concurrency.

- **Scalability**  
  Redis is horizontally scalable via clustering/sharding.

- **Feature Rich**  
  TTL, eviction policies, built-in data structures (lists, hashes, sorted sets), Pub/Sub.

---

## 2. Core Concepts

| Concept               | Description                                                                                         |
|-----------------------|-----------------------------------------------------------------------------------------------------|
| **Cache Aside**       | Application checks cache first; on miss it loads from source and populates the cache.              |
| **Read-Through**      | Cache sits in front of the data source; reads go through the cache automatically.                   |
| **Write-Through**     | Writes go to cache first, then synchronously to the database.                                       |
| **Write-Behind**      | Writes go to cache first, then asynchronously to the database.                                      |
| **Eviction Policies** | `LRU` (Least Recently Used), `LFU` (Least Frequently Used), TTL-based.                              |
| **Data Structures**   | Strings (key→value), Hashes (grouped fields), Sorted Sets (leaderboards), Lists, Sets, etc.         |
| **Clustering & Persistence** | Cluster mode for sharding; AOF/RDB snapshots for durability.                              |

---

## 3. Typical Use Cases

1. **Session Storage** in stateless services  
2. **API Response Caching** (e.g. product catalog, lookup tables)  
3. **Rate Limiting Counters** (token bucket, leaky bucket)  
4. **Leaderboards & Counters** (sorted sets)  
5. **Distributed Locks** (Redlock algorithm)  

---

## 4. Spring Boot Integration

### 4.1 Dependencies

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

```
### 4.2 Configuration (application.yml)

```yml
spring:
  redis:
    host: redis-host
    port: 6379
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0

  cache:
    type: redis
    redis:
      time-to-live: 600000  # global TTL in ms (optional)

```

### 4.3 Enable Caching

```java
@Configuration
@EnableCaching
public class CacheConfig {
  // Optionally customize a RedisCacheManager for per-cache TTLs, serialization, etc.
}

```

### 5. Real-Time Example: Product Catalog Lookup

```java
@Service
public class ProductService {

  @Autowired
  private ProductRepository repo;

  @Cacheable(value = "products", key = "#id")
  public Product getById(Long id) {
    return repo.findById(id)
               .orElseThrow(() -> new NotFoundException(id));
  }

  @CachePut(value = "products", key = "#result.id")
  public Product updateProduct(Product p) {
    return repo.save(p);
  }

  @CacheEvict(value = "products", key = "#id")
  public void deleteProduct(Long id) {
    repo.deleteById(id);
  }
}


```


- @Cacheable checks the "products" cache for id; on miss, loads from DB and stores the result.

- @CachePut always executes the method and updates the cache.

- @CacheEvict removes the key when a product is deleted.


### 6. Advanced Configuration

```java

@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
  RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
    .entryTtl(Duration.ofMinutes(5))
    .disableCachingNullValues()
    .serializeValuesWith(
      SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

  Map<String, RedisCacheConfiguration> caches = Map.of(
    "products", defaultConfig.entryTtl(Duration.ofMinutes(10)),
    "sessions", defaultConfig.entryTtl(Duration.ofHours(1))
  );

  return RedisCacheManager.builder(factory)
    .withInitialCacheConfigurations(caches)
    .transactionAware()
    .build();
}



```


- Per-cache TTLs

- JSON serialization

- Null-value filtering


### 6.2 Manual Cache-Aside

```java

@Autowired
private StringRedisTemplate redis;

public Product getByIdManual(Long id) {
  String key = "product:" + id;
  String json = redis.opsForValue().get(key);
  if (json != null) {
    return objectMapper.readValue(json, Product.class);
  }
  Product p = repo.findById(id).orElseThrow(...);
  redis.opsForValue()
       .set(key, objectMapper.writeValueAsString(p), Duration.ofMinutes(10));
  return p;
}

```


### 7. Redis in a Microservices Architecture

```java
[ Client ]
    │
    ▼
[ API Gateway ] — auth, rate-limit, routing
    │
    ▼
┌─────────┬─────────┬─────────┐
│Service A│Service B│Service C│
│— cache: products    │— cache: sessions    │— pub/sub notifications
│— DB                 │                     │
└─────────┴─────────┴─────────┘
```

- Shared Cache Layer: All services point to the same Redis cluster → fast data sharing.

- Data Ownership: Each service owns its own key-spaces (e.g. products:*, sessions:*).

- Independent Scaling: Redis clusters scale separately from service instances.


### 8. Deciding When to Cache

- Read-Heavy / Expensive calls vs. underlying data change frequency.

- Latency Sensitivity: critical user flows require sub-ms responses.

- Cost Considerations: reduce database compute costs.

- Data Freshness: TTLs and explicit evictions for dynamic data.

- Consistency Needs: balance staleness vs. performance.


### 9. Best Practices

1. Namespace Keys: service:cacheName:key to avoid collisions.

2. Monitor Metrics: cache hit rate, evictions, memory via Redis INFO or exporters → Prometheus.

3. Graceful Degradation: fall back to DB on Redis outages.

3. Tune Eviction: choose TTLs and policies based on data volatility.

4. Secure Redis: enable AUTH, TLS, and restrict network access to your VPC.

