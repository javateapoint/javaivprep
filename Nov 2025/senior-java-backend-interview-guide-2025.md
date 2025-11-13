# Senior Java Backend Developer Interview Preparation Guide - 2025

## Complete Documentation for 100% Interview Readiness

**Prepared on:** November 13, 2025  
**Target Role:** Senior Java Backend Developer  
**Interview Timeline:** December 2025

---

## Table of Contents

1. [Introduction](#introduction)
2. [Core Java & OOPs Concepts](#core-java--oops-concepts)
3. [Java 8+ Features (Java 8, 11, 17, 21)](#java-8-features)
4. [Spring Framework & Spring Boot](#spring-framework--spring-boot)
5. [Microservices Architecture](#microservices-architecture)
6. [Database & Persistence (Hibernate/JPA)](#database--persistence)
7. [REST API Design & Best Practices](#rest-api-design--best-practices)
8. [Multithreading & Concurrency](#multithreading--concurrency)
9. [Design Patterns](#design-patterns)
10. [System Design for Senior Engineers](#system-design-for-senior-engineers)
11. [Security (Spring Security, OAuth2, JWT)](#security)
12. [Messaging & Event-Driven Architecture](#messaging--event-driven-architecture)
13. [DevOps, CI/CD & Containerization](#devops-cicd--containerization)
14. [Testing (JUnit, Mockito, Integration Testing)](#testing)
15. [Performance Optimization & Memory Management](#performance-optimization--memory-management)
16. [Cloud Services (AWS Basics)](#cloud-services-aws-basics)
17. [Behavioral Interview Questions](#behavioral-interview-questions)
18. [Coding Challenges & Algorithms](#coding-challenges--algorithms)
19. [Interview Day Preparation Checklist](#interview-day-preparation-checklist)

---

## 1. Introduction

This comprehensive guide covers everything you need to prepare for senior Java backend developer interviews in 2025. The guide is structured based on real interview patterns from top companies and includes detailed answers, code examples, and practical scenarios.

### What Senior Interviewers Look For:
- **Depth of Knowledge**: Not just syntax, but understanding "why" and "when"
- **System Thinking**: Ability to design scalable, maintainable systems
- **Trade-off Analysis**: Understanding pros/cons of different approaches
- **Production Experience**: Real-world problem-solving examples
- **Leadership & Mentoring**: Guiding teams and making architectural decisions

---

## 2. Core Java & OOPs Concepts

### 2.1 The Four Pillars of OOPs

**Q: Explain the four main principles of Object-Oriented Programming.**

**Answer:**

1. **Encapsulation**: Bundling data and methods that operate on that data within a single unit (class). It hides internal state and requires interaction through methods.
   ```java
   public class BankAccount {
       private double balance; // Hidden from outside
       
       public void deposit(double amount) {
           if (amount > 0) {
               balance += amount;
           }
       }
       
       public double getBalance() {
           return balance;
       }
   }
   ```

2. **Inheritance**: Mechanism where a new class derives properties and behaviors from an existing class.
   ```java
   public class Animal {
       protected String name;
       public void eat() { }
   }
   
   public class Dog extends Animal {
       public void bark() { }
   }
   ```

3. **Polymorphism**: Ability of objects to take multiple forms. Two types:
   - **Compile-time (Method Overloading)**
   - **Runtime (Method Overriding)**
   ```java
   // Overloading
   public void print(String s) { }
   public void print(int i) { }
   
   // Overriding
   @Override
   public void eat() {
       System.out.println("Dog is eating");
   }
   ```

4. **Abstraction**: Hiding complex implementation details and showing only essential features.
   ```java
   public abstract class Vehicle {
       abstract void start();
       abstract void stop();
   }
   ```

### 2.2 Key Differences

**Q: What is the difference between Abstract Class and Interface?**

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| Methods | Can have abstract and concrete methods | Only abstract methods (before Java 8) |
| Variables | Can have instance variables | Only constants (public static final) |
| Inheritance | Single inheritance | Multiple inheritance supported |
| Access Modifiers | Can have any access modifier | Methods are public by default |
| Use Case | When classes share common base | For defining contracts |

**Java 8+ Update**: Interfaces can now have default and static methods.

```java
public interface PaymentProcessor {
    void processPayment(double amount);
    
    // Default method (Java 8+)
    default void logTransaction() {
        System.out.println("Transaction logged");
    }
}
```

### 2.3 Exception Handling

**Q: Explain checked vs unchecked exceptions.**

**Checked Exceptions:**
- Must be caught or declared in method signature
- Checked at compile time
- Examples: IOException, SQLException
```java
public void readFile() throws IOException {
    FileReader fr = new FileReader("file.txt");
}
```

**Unchecked Exceptions:**
- Runtime exceptions, not checked at compile time
- Examples: NullPointerException, ArrayIndexOutOfBoundsException
```java
public void divide(int a, int b) {
    return a / b; // Can throw ArithmeticException
}
```

**Best Practices:**
- Don't catch generic Exception
- Use specific exceptions
- Log exceptions properly
- Create custom exceptions for business logic
- Don't use exceptions for control flow

### 2.4 Collections Framework

**Q: How does HashMap work internally?**

**Answer:**
HashMap uses an array of buckets with linked lists (or trees after Java 8). 

**Key Points:**
1. **Initial Capacity**: Default 16 buckets
2. **Load Factor**: Default 0.75 (resizes when 75% full)
3. **Hash Calculation**: Uses hashCode() to determine bucket
4. **Collision Handling**: Linked list or Red-Black tree (if > 8 elements)

```java
// Internal Structure (simplified)
class HashMap<K,V> {
    Node<K,V>[] table;  // Array of buckets
    
    static class Node<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    }
    
    public V put(K key, V value) {
        int hash = hash(key);
        int index = hash & (table.length - 1);
        // Add to bucket at index
    }
}
```

**HashMap vs ConcurrentHashMap:**
| Feature | HashMap | ConcurrentHashMap |
|---------|---------|-------------------|
| Thread Safety | Not thread-safe | Thread-safe |
| Null Keys | Allows one null key | Doesn't allow null |
| Performance | Fast | Slightly slower |
| Use Case | Single-threaded | Multi-threaded |

### 2.5 SOLID Principles

**Q: Explain SOLID principles with examples.**

**1. Single Responsibility Principle (SRP)**
- A class should have only one reason to change
```java
// Bad
class Employee {
    void calculateSalary() { }
    void saveToDatabase() { }  // Database concern
    void generateReport() { }   // Reporting concern
}

// Good
class Employee {
    void calculateSalary() { }
}
class EmployeeRepository {
    void save(Employee e) { }
}
class EmployeeReportGenerator {
    void generate(Employee e) { }
}
```

**2. Open/Closed Principle (OCP)**
- Open for extension, closed for modification
```java
public interface PaymentMethod {
    void pay(double amount);
}

public class CreditCardPayment implements PaymentMethod {
    public void pay(double amount) { }
}

public class PayPalPayment implements PaymentMethod {
    public void pay(double amount) { }
}
```

**3. Liskov Substitution Principle (LSP)**
- Subtypes must be substitutable for their base types
```java
public class Bird {
    void fly() { }
}

// Bad - Penguin can't fly
public class Penguin extends Bird {
    void fly() {
        throw new UnsupportedOperationException();
    }
}

// Good - Better abstraction
public abstract class Bird { }
public abstract class FlyingBird extends Bird {
    abstract void fly();
}
public class Sparrow extends FlyingBird { }
public class Penguin extends Bird { }
```

**4. Interface Segregation Principle (ISP)**
- No client should be forced to depend on methods it doesn't use
```java
// Bad
interface Worker {
    void work();
    void eat();
}

// Good
interface Workable {
    void work();
}
interface Eatable {
    void eat();
}
```

**5. Dependency Inversion Principle (DIP)**
- Depend on abstractions, not concrete classes
```java
// Bad
public class UserService {
    private MySQLDatabase database = new MySQLDatabase();
}

// Good
public class UserService {
    private Database database;  // Interface
    
    public UserService(Database database) {
        this.database = database;
    }
}
```

---

## 3. Java 8+ Features

### 3.1 Java 8 Features

**Q: Explain Lambda Expressions and Functional Interfaces.**

**Lambda Expression:** Concise way to represent anonymous functions
```java
// Old way
Comparator<String> comp = new Comparator<String>() {
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);
    }
};

// Lambda way
Comparator<String> comp = (s1, s2) -> s1.compareTo(s2);
```

**Functional Interface:** Interface with exactly one abstract method
```java
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);
}

Calculator add = (a, b) -> a + b;
Calculator multiply = (a, b) -> a * b;
```

**Common Functional Interfaces:**
- `Predicate<T>`: Takes T, returns boolean
- `Function<T,R>`: Takes T, returns R
- `Consumer<T>`: Takes T, returns void
- `Supplier<T>`: Takes nothing, returns T

### 3.2 Stream API

**Q: Explain Stream API with examples.**

**Stream Operations:**
```java
List<Employee> employees = getEmployees();

// Filter and map
List<String> names = employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .map(Employee::getName)
    .collect(Collectors.toList());

// Reduce
int totalSalary = employees.stream()
    .map(Employee::getSalary)
    .reduce(0, Integer::sum);

// Group by
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// Sorting
List<Employee> sorted = employees.stream()
    .sorted(Comparator.comparing(Employee::getName))
    .collect(Collectors.toList());
```

**Interview Tip:** Always mention:
- Streams are lazy (intermediate operations)
- Terminal operations trigger execution
- Parallel streams for large datasets
- Don't reuse streams

### 3.3 Optional Class

**Q: How does Optional help avoid NullPointerException?**

```java
// Without Optional
public String getUserCity(Long userId) {
    User user = findUser(userId);
    if (user != null) {
        Address address = user.getAddress();
        if (address != null) {
            return address.getCity();
        }
    }
    return "Unknown";
}

// With Optional
public String getUserCity(Long userId) {
    return findUser(userId)
        .flatMap(User::getAddress)
        .map(Address::getCity)
        .orElse("Unknown");
}
```

**Best Practices:**
- Never use `Optional.get()` without checking
- Use `orElse()`, `orElseThrow()`, or `ifPresent()`
- Don't use Optional as method parameters
- Use for return types when result might be absent

### 3.4 Java 11 Features

**Key Features:**
1. **Local Variable Syntax (var)**
```java
var list = new ArrayList<String>();
var stream = list.stream();
```

2. **New String Methods**
```java
String str = "  Hello  ";
str.isBlank();           // true if whitespace
str.strip();             // Better than trim()
str.lines();             // Stream of lines
"Java".repeat(3);        // JavaJavaJava
```

3. **HTTP Client API**
```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com"))
    .build();
HttpResponse<String> response = client.send(request,
    HttpResponse.BodyHandlers.ofString());
```

### 3.5 Java 17 Features (LTS)

**1. Sealed Classes**
```java
public sealed class Shape permits Circle, Rectangle, Triangle {
}

public final class Circle extends Shape { }
public final class Rectangle extends Shape { }
public non-sealed class Triangle extends Shape { }
```

**2. Pattern Matching for instanceof**
```java
// Old way
if (obj instanceof String) {
    String str = (String) obj;
    System.out.println(str.length());
}

// New way
if (obj instanceof String str) {
    System.out.println(str.length());
}
```

**3. Records**
```java
// Immutable data carrier
public record Person(String name, int age) { }

Person p = new Person("John", 30);
System.out.println(p.name());  // Auto-generated
```

### 3.6 Java 21 Features (Latest LTS)

**1. Virtual Threads (Project Loom)**
```java
// Traditional Thread
Thread thread = new Thread(() -> {
    System.out.println("Traditional Thread");
});
thread.start();

// Virtual Thread
Thread.startVirtualThread(() -> {
    System.out.println("Virtual Thread");
});

// Executor Service with Virtual Threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> processTask());
}
```

**Benefits:**
- Millions of threads possible
- Lower memory footprint
- Better for I/O-bound operations

**2. Sequenced Collections**
```java
interface SequencedCollection<E> extends Collection<E> {
    void addFirst(E element);
    void addLast(E element);
    E getFirst();
    E getLast();
    E removeFirst();
    E removeLast();
    SequencedCollection<E> reversed();
}
```

---

## 4. Spring Framework & Spring Boot

### 4.1 Core Spring Concepts

**Q: What is Dependency Injection and Inversion of Control?**

**Inversion of Control (IoC):**
- Framework controls object creation, not the application
- Spring IoC container manages bean lifecycle

**Dependency Injection:**
- Way to achieve IoC
- Dependencies provided from outside

**Types of DI:**
```java
// 1. Constructor Injection (Recommended)
@Service
public class UserService {
    private final UserRepository repository;
    
    @Autowired
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}

// 2. Setter Injection
@Service
public class UserService {
    private UserRepository repository;
    
    @Autowired
    public void setRepository(UserRepository repository) {
        this.repository = repository;
    }
}

// 3. Field Injection (Not recommended)
@Service
public class UserService {
    @Autowired
    private UserRepository repository;
}
```

### 4.2 Spring Boot Essentials

**Q: What are the advantages of Spring Boot?**

**Key Advantages:**
1. **Auto-configuration**: Automatically configures beans
2. **Embedded Servers**: Tomcat/Jetty embedded
3. **Starter Dependencies**: Pre-configured dependency sets
4. **Production-ready**: Actuator for monitoring
5. **No XML Configuration**: Java-based configuration

**Main Annotations:**
```java
@SpringBootApplication  // Combines @Configuration, @EnableAutoConfiguration, @ComponentScan
@RestController        // @Controller + @ResponseBody
@Service              // Business logic layer
@Repository           // Data access layer
@Component            // Generic component
@Configuration        // Java-based configuration
@Bean                 // Method-level, creates bean
```

### 4.3 Spring Boot Application Properties

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
  
  profiles:
    active: dev

server:
  port: 8080
  
logging:
  level:
    root: INFO
    com.myapp: DEBUG
```

### 4.4 Spring Boot Actuator

**Q: What is Spring Boot Actuator?**

Provides production-ready features for monitoring and managing applications.

**Common Endpoints:**
- `/actuator/health`: Application health
- `/actuator/metrics`: Application metrics
- `/actuator/info`: Application information
- `/actuator/env`: Environment properties
- `/actuator/loggers`: Logging configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
```

### 4.5 Transaction Management

**Q: Explain @Transactional annotation.**

```java
@Service
public class OrderService {
    
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryService.reduceStock(order.getItems());
        paymentService.processPayment(order.getAmount());
        // All or nothing - atomicity guaranteed
    }
    
    @Transactional(readOnly = true)
    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }
    
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        rollbackFor = Exception.class
    )
    public void complexOperation() { }
}
```

**Propagation Types:**
- `REQUIRED` (default): Join existing or create new
- `REQUIRES_NEW`: Always create new transaction
- `NESTED`: Nested within existing
- `SUPPORTS`: Join if exists, else non-transactional
- `NOT_SUPPORTED`: Execute non-transactionally
- `NEVER`: Throw exception if transaction exists
- `MANDATORY`: Throw exception if no transaction

### 4.6 Spring Profiles

```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        // Development database
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // Production database
    }
}
```

---

## 5. Microservices Architecture

### 5.1 Microservices vs Monolithic

**Q: When should you use microservices over monolithic architecture?**

| Aspect | Monolithic | Microservices |
|--------|------------|---------------|
| **Deployment** | Single unit | Independent services |
| **Scaling** | Scale entire app | Scale individual services |
| **Technology** | Same stack | Polyglot architecture |
| **Team** | Single team | Multiple teams |
| **Complexity** | Low initially | High infrastructure complexity |
| **Database** | Shared database | Database per service |

**Use Microservices When:**
- Large, complex applications
- Different scalability requirements
- Multiple teams
- Need for technology diversity
- Continuous deployment needed

**Challenges:**
- Distributed system complexity
- Data consistency
- Service discovery
- Inter-service communication
- Monitoring and debugging

### 5.2 Service Discovery

**Q: Explain service discovery in microservices.**

**Service Registry (Eureka Example):**
```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication { }

// Eureka Client (Microservice)
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication { }

// application.yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    preferIpAddress: true
```

**Service Communication:**
```java
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    public Product getProduct(Long productId) {
        // Using service name instead of hardcoded URL
        return restTemplate.getForObject(
            "http://PRODUCT-SERVICE/api/products/" + productId,
            Product.class
        );
    }
}
```

### 5.3 API Gateway

**Q: What is the role of API Gateway in microservices?**

**Purpose:**
- Single entry point for all clients
- Request routing
- Load balancing
- Authentication & Authorization
- Rate limiting
- Request/Response transformation

**Spring Cloud Gateway Example:**
```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .addRequestHeader("X-Request-Source", "Gateway")
                    .circuitBreaker(config -> config
                        .setName("orderCircuitBreaker")
                        .setFallbackUri("/fallback/orders"))
                )
                .uri("lb://ORDER-SERVICE")
            )
            .route("product-service", r -> r
                .path("/api/products/**")
                .uri("lb://PRODUCT-SERVICE")
            )
            .build();
    }
}
```

### 5.4 Circuit Breaker Pattern

**Q: Explain Circuit Breaker pattern with Resilience4j.**

**States:**
1. **Closed**: Normal operation, requests pass through
2. **Open**: Failures exceed threshold, requests fail fast
3. **Half-Open**: Test if service recovered

```java
@Service
public class ProductService {
    
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductFallback")
    @Retry(name = "productService")
    @RateLimiter(name = "productService")
    public Product getProductById(Long id) {
        return restTemplate.getForObject(
            "http://PRODUCT-SERVICE/api/products/" + id,
            Product.class
        );
    }
    
    public Product getProductFallback(Long id, Exception e) {
        return new Product(id, "Unavailable", "Service temporarily unavailable");
    }
}
```

**Configuration:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 3
```

### 5.5 Distributed Transactions - Saga Pattern

**Q: How do you handle distributed transactions in microservices?**

**Saga Pattern - Choreography Approach:**
```java
// Order Service
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(OrderRequest request) {
        // 1. Create order
        Order order = orderRepository.save(new Order(request));
        
        // 2. Publish event
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(),
            order.getItems(),
            order.getTotalAmount()
        ));
    }
    
    @EventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        // Compensating transaction
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
}

// Payment Service
@Service
public class PaymentService {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            processPayment(event.getOrderId(), event.getAmount());
            eventPublisher.publish(new PaymentSuccessEvent(event.getOrderId()));
        } catch (Exception e) {
            eventPublisher.publish(new PaymentFailedEvent(event.getOrderId()));
        }
    }
}
```

**Orchestration Approach:**
```java
@Service
public class OrderOrchestrator {
    
    public void processOrder(OrderRequest request) {
        // Step 1: Create Order
        Order order = orderService.createOrder(request);
        
        try {
            // Step 2: Reserve Inventory
            inventoryService.reserveInventory(order.getItems());
            
            // Step 3: Process Payment
            paymentService.processPayment(order.getTotalAmount());
            
            // Step 4: Confirm Order
            orderService.confirmOrder(order.getId());
            
        } catch (Exception e) {
            // Compensate
            inventoryService.releaseInventory(order.getItems());
            orderService.cancelOrder(order.getId());
            throw e;
        }
    }
}
```

### 5.6 Microservices Communication

**Synchronous (REST/gRPC):**
```java
@FeignClient(name = "inventory-service")
public interface InventoryClient {
    
    @GetMapping("/api/inventory/{productId}")
    InventoryResponse checkInventory(@PathVariable Long productId);
}
```

**Asynchronous (Message Queue):**
```java
// Producer
@Service
public class OrderEventPublisher {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent(order);
        rabbitTemplate.convertAndSend("order-exchange", 
            "order.created", event);
    }
}

// Consumer
@Service
public class NotificationService {
    @RabbitListener(queues = "notification-queue")
    public void handleOrderCreated(OrderEvent event) {
        sendNotification(event);
    }
}
```

---

## 6. Database & Persistence (Hibernate/JPA)

### 6.1 JPA/Hibernate Basics

**Q: What is the difference between JPA and Hibernate?**

- **JPA**: Specification (interface) for ORM
- **Hibernate**: Implementation of JPA (also has additional features)

**Entity Example:**
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(name = "full_name")
    private String name;
    
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "created_at")
    private Date createdAt;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
    
    // Getters and setters
}
```

### 6.2 Relationships

**One-to-Many:**
```java
@Entity
public class Department {
    @Id
    @GeneratedValue
    private Long id;
    
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees;
}

@Entity
public class Employee {
    @Id
    @GeneratedValue
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "dept_id")
    private Department department;
}
```

**Many-to-Many:**
```java
@Entity
public class Student {
    @Id
    private Long id;
    
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses;
}

@Entity
public class Course {
    @Id
    private Long id;
    
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students;
}
```

### 6.3 Lazy vs Eager Loading

**Q: Explain lazy loading vs eager loading.**

```java
@Entity
public class User {
    @Id
    private Long id;
    
    // Lazy: Load when accessed
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    private List<Order> orders;
    
    // Eager: Load immediately with user
    @ManyToOne(fetch = FetchType.EAGER)
    private Department department;
}
```

**Best Practice:**
- Default to LAZY loading
- Use JOIN FETCH for specific queries
- Avoid N+1 problem

**Solving N+1 Problem:**
```java
// Bad - N+1 queries
List<User> users = userRepository.findAll();
users.forEach(user -> {
    user.getOrders().size(); // Separate query for each user
});

// Good - Single query with JOIN FETCH
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### 6.4 Spring Data JPA

**Repository Interface:**
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Query Methods (auto-generated)
    List<User> findByEmail(String email);
    List<User> findByNameContaining(String name);
    List<User> findByCreatedAtAfter(Date date);
    
    // Custom JPQL
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    Optional<User> findByEmailCustom(String email);
    
    // Native SQL
    @Query(value = "SELECT * FROM users WHERE email = ?1", 
           nativeQuery = true)
    User findByEmailNative(String email);
    
    // With Pagination
    Page<User> findByNameContaining(String name, Pageable pageable);
    
    // Projections
    @Query("SELECT new com.example.dto.UserDTO(u.id, u.name, u.email) " +
           "FROM User u WHERE u.id = ?1")
    UserDTO findUserDTOById(Long id);
}
```

### 6.5 Transaction Management & Propagation

```java
@Service
public class OrderService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void createOrder(Order order) {
        orderRepository.save(order);
        updateInventory(order); // Joins transaction
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAudit(AuditLog log) {
        auditRepository.save(log);
        // Separate transaction - commits independently
    }
}
```

### 6.6 Database Connection Pooling

**HikariCP Configuration (Spring Boot default):**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

---

## 7. REST API Design & Best Practices

### 7.1 RESTful Principles

**Q: What are the key principles of REST?**

1. **Client-Server Architecture**
2. **Stateless**: Each request contains all information needed
3. **Cacheable**: Responses should define if they're cacheable
4. **Uniform Interface**: Consistent API design
5. **Layered System**: Client doesn't know if connected to end server
6. **Code on Demand** (optional): Server can send executable code

### 7.2 HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Update/Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Delete resource | Yes | No |

### 7.3 REST Controller Example

```java
@RestController
@RequestMapping("/api/v1/users")
@Validated
public class UserController {
    
    @Autowired
    private UserService userService;
    
    // GET all users with pagination
    @GetMapping
    public ResponseEntity<Page<UserDTO>> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy) {
        
        Pageable pageable = PageRequest.of(page, size, 
            Sort.by(sortBy));
        return ResponseEntity.ok(userService.getAllUsers(pageable));
    }
    
    // GET single user
    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUserById(
            @PathVariable @Min(1) Long id) {
        return userService.getUserById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // POST create user
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserDTO> createUser(
            @Valid @RequestBody UserRequest request) {
        UserDTO created = userService.createUser(request);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.getId())
            .toUri();
        return ResponseEntity.created(location).body(created);
    }
    
    // PUT update user
    @PutMapping("/{id}")
    public ResponseEntity<UserDTO> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UserRequest request) {
        return ResponseEntity.ok(userService.updateUser(id, request));
    }
    
    // PATCH partial update
    @PatchMapping("/{id}")
    public ResponseEntity<UserDTO> partialUpdate(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        return ResponseEntity.ok(
            userService.partialUpdate(id, updates));
    }
    
    // DELETE user
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```

### 7.4 Global Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );
        
        ValidationErrorResponse response = new ValidationErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            errors,
            LocalDateTime.now()
        );
        return new ResponseEntity<>(response, HttpStatus.BAD_REQUEST);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal server error",
            LocalDateTime.now()
        );
        return new ResponseEntity<>(error, 
            HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

### 7.5 API Versioning

**URI Versioning:**
```java
@RequestMapping("/api/v1/users")
public class UserControllerV1 { }

@RequestMapping("/api/v2/users")
public class UserControllerV2 { }
```

**Header Versioning:**
```java
@GetMapping(value = "/users", headers = "API-Version=1")
public ResponseEntity<UserV1> getUserV1() { }

@GetMapping(value = "/users", headers = "API-Version=2")
public ResponseEntity<UserV2> getUserV2() { }
```

**Accept Header (Content Negotiation):**
```java
@GetMapping(value = "/users", 
    produces = "application/vnd.company.v1+json")
public ResponseEntity<UserV1> getUserV1() { }

@GetMapping(value = "/users", 
    produces = "application/vnd.company.v2+json")
public ResponseEntity<UserV2> getUserV2() { }
```

### 7.6 API Best Practices

**1. Use Proper Status Codes**
- 200 OK: Success
- 201 Created: Resource created
- 204 No Content: Success, no body
- 400 Bad Request: Client error
- 401 Unauthorized: Authentication required
- 403 Forbidden: No permission
- 404 Not Found: Resource doesn't exist
- 500 Internal Server Error: Server error

**2. Use Nouns, Not Verbs**
```
✓ GET /api/users
✓ POST /api/users
✗ GET /api/getUsers
✗ POST /api/createUser
```

**3. Plural Resource Names**
```
✓ /api/users
✓ /api/orders
✗ /api/user
✗ /api/order
```

**4. Nested Resources**
```
GET /api/users/{userId}/orders
GET /api/users/{userId}/orders/{orderId}
```

**5. Filtering, Sorting, Pagination**
```
GET /api/users?status=active&sort=name&page=0&size=10
```

---

## 8. Multithreading & Concurrency

### 8.1 Thread Creation

**Q: What are the ways to create threads in Java?**

```java
// 1. Extending Thread class
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running");
    }
}
new MyThread().start();

// 2. Implementing Runnable
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Runnable running");
    }
}
new Thread(new MyRunnable()).start();

// 3. Lambda expression
new Thread(() -> {
    System.out.println("Lambda thread");
}).start();

// 4. Callable (returns result)
Callable<Integer> task = () -> {
    return 42;
};
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);
```

### 8.2 Thread Lifecycle

**States:**
1. **NEW**: Thread created but not started
2. **RUNNABLE**: Ready to run or running
3. **BLOCKED**: Waiting for monitor lock
4. **WAITING**: Waiting indefinitely
5. **TIMED_WAITING**: Waiting for specified time
6. **TERMINATED**: Thread completed

### 8.3 Synchronization

**Q: How do you ensure thread safety?**

**Synchronized Method:**
```java
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

**Synchronized Block:**
```java
public class Counter {
    private int count = 0;
    private final Object lock = new Object();
    
    public void increment() {
        synchronized(lock) {
            count++;
        }
    }
}
```

**Volatile Keyword:**
```java
public class SharedResource {
    private volatile boolean flag = false;
    
    public void setFlag() {
        flag = true;  // Visible to all threads immediately
    }
}
```

### 8.4 Concurrent Collections

```java
// Thread-safe alternatives
Map<String, String> map = new ConcurrentHashMap<>();
List<String> list = new CopyOnWriteArrayList<>();
Queue<String> queue = new ConcurrentLinkedQueue<>();
BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>();

// Example usage
BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>(100);

// Producer
taskQueue.put(new Task());  // Blocks if full

// Consumer
Task task = taskQueue.take();  // Blocks if empty
```

### 8.5 ExecutorService

**Q: Why use ExecutorService over Thread?**

**Benefits:**
- Thread pooling (reuse threads)
- Better resource management
- Task queuing
- Lifecycle management

```java
// Fixed thread pool
ExecutorService executor = Executors.newFixedThreadPool(10);

// Submit tasks
executor.submit(() -> {
    // Task code
});

// Callable with Future
Future<String> future = executor.submit(() -> {
    return "Result";
});

String result = future.get();  // Blocking call

// Shutdown
executor.shutdown();
try {
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
}
```

**Types of Thread Pools:**
```java
// Fixed size
Executors.newFixedThreadPool(10);

// Single thread
Executors.newSingleThreadExecutor();

// Cached (creates as needed)
Executors.newCachedThreadPool();

// Scheduled
ScheduledExecutorService scheduler = 
    Executors.newScheduledThreadPool(5);
scheduler.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);
```

### 8.6 CompletableFuture (Java 8+)

```java
// Async computation
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // Long running task
    return "Result";
});

// Chain operations
future
    .thenApply(result -> result.toUpperCase())
    .thenAccept(System.out::println)
    .exceptionally(ex -> {
        System.err.println("Error: " + ex);
        return null;
    });

// Combine multiple futures
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = future1.thenCombine(future2,
    (s1, s2) -> s1 + " " + s2);
```

### 8.7 Common Concurrency Problems

**1. Deadlock:**
```java
// Problem
synchronized(lock1) {
    synchronized(lock2) {
        // Code
    }
}

// Solution: Lock ordering
```

**2. Race Condition:**
```java
// Problem
if (count < MAX) {
    count++;  // Not atomic
}

// Solution
synchronized(this) {
    if (count < MAX) {
        count++;
    }
}

// Or use AtomicInteger
AtomicInteger count = new AtomicInteger(0);
if (count.get() < MAX) {
    count.incrementAndGet();
}
```

---

## 9. Design Patterns

### 9.1 Creational Patterns

**Singleton:**
```java
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;
    
    private DatabaseConnection() { }
    
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}

// Enum Singleton (Best approach)
public enum DatabaseConnection {
    INSTANCE;
    
    public void connect() { }
}
```

**Factory:**
```java
public interface Vehicle {
    void drive();
}

public class VehicleFactory {
    public static Vehicle createVehicle(String type) {
        return switch(type) {
            case "CAR" -> new Car();
            case "BIKE" -> new Bike();
            case "TRUCK" -> new Truck();
            default -> throw new IllegalArgumentException();
        };
    }
}
```

**Builder:**
```java
public class User {
    private String name;
    private String email;
    private int age;
    private String address;
    
    private User(UserBuilder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
        this.address = builder.address;
    }
    
    public static class UserBuilder {
        private String name;
        private String email;
        private int age;
        private String address;
        
        public UserBuilder name(String name) {
            this.name = name;
            return this;
        }
        
        public UserBuilder email(String email) {
            this.email = email;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = new User.UserBuilder()
    .name("John")
    .email("john@example.com")
    .age(30)
    .build();
```

### 9.2 Structural Patterns

**Adapter:**
```java
// Legacy interface
interface OldPayment {
    void makePayment(double amount);
}

// New interface
interface NewPayment {
    void processPayment(PaymentRequest request);
}

// Adapter
class PaymentAdapter implements NewPayment {
    private OldPayment oldPayment;
    
    public void processPayment(PaymentRequest request) {
        oldPayment.makePayment(request.getAmount());
    }
}
```

**Decorator:**
```java
interface Coffee {
    double cost();
    String description();
}

class SimpleCoffee implements Coffee {
    public double cost() { return 5.0; }
    public String description() { return "Simple Coffee"; }
}

abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
}

class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    public double cost() {
        return coffee.cost() + 1.5;
    }
    
    public String description() {
        return coffee.description() + ", Milk";
    }
}

// Usage
Coffee coffee = new MilkDecorator(new SimpleCoffee());
```

### 9.3 Behavioral Patterns

**Strategy:**
```java
interface PaymentStrategy {
    void pay(double amount);
}

class CreditCardStrategy implements PaymentStrategy {
    public void pay(double amount) {
        System.out.println("Paid " + amount + " using Credit Card");
    }
}

class PayPalStrategy implements PaymentStrategy {
    public void pay(double amount) {
        System.out.println("Paid " + amount + " using PayPal");
    }
}

class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout(double amount) {
        paymentStrategy.pay(amount);
    }
}
```

**Observer:**
```java
interface Observer {
    void update(String message);
}

class Subject {
    private List<Observer> observers = new ArrayList<>();
    
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    public void notifyAllObservers(String message) {
        observers.forEach(o -> o.update(message));
    }
}

// Usage in Spring
@Component
public class OrderEventPublisher {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public void publishOrderCreated(Order order) {
        publisher.publishEvent(new OrderCreatedEvent(order));
    }
}

@Component
public class EmailNotificationListener {
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        sendEmail(event.getOrder());
    }
}
```

---

## 10. System Design for Senior Engineers

### 10.1 System Design Approach

**Steps:**
1. **Clarify Requirements** (5 min)
   - Functional requirements
   - Non-functional requirements
   - Scale (users, data, traffic)
   
2. **High-Level Design** (15 min)
   - Draw main components
   - APIs
   - Database schema
   
3. **Deep Dive** (15 min)
   - Bottlenecks
   - Scaling strategies
   - Trade-offs
   
4. **Wrap Up** (5 min)
   - Monitoring
   - Security
   - Future improvements

### 10.2 Scalability Patterns

**Horizontal vs Vertical Scaling:**
- **Vertical**: Add more resources (CPU, RAM) to single machine
- **Horizontal**: Add more machines

**Load Balancing:**
```
Client → Load Balancer → [Server1, Server2, Server3]
```

**Algorithms:**
- Round Robin
- Least Connections
- IP Hash
- Weighted Round Robin

**Caching:**
```
Client → CDN → Load Balancer → App Servers → Cache → Database
```

**Layers:**
1. CDN (Content Delivery Network)
2. Application cache (Redis, Memcached)
3. Database cache

**Database Scaling:**
- **Replication**: Master-Slave
- **Sharding**: Horizontal partitioning
- **Partitioning**: Vertical/Horizontal

### 10.3 Design: URL Shortener

**Requirements:**
- Shorten long URLs
- Redirect short URL to original
- 100M URLs per month
- Low latency

**API Design:**
```
POST /api/shorten
Request: { "longUrl": "https://example.com/very/long/url" }
Response: { "shortUrl": "https://short.ly/abc123" }

GET /abc123
Response: 302 Redirect to original URL
```

**Database Schema:**
```sql
CREATE TABLE urls (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    click_count INT DEFAULT 0,
    INDEX idx_short_code (short_code)
);
```

**Algorithm for Short Code:**
```java
public class URLShortener {
    private static final String ALPHABET = 
        "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    private static final int BASE = ALPHABET.length();
    
    public String encode(long id) {
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            sb.append(ALPHABET.charAt((int)(id % BASE)));
            id /= BASE;
        }
        return sb.reverse().toString();
    }
    
    public long decode(String shortUrl) {
        long id = 0;
        for (char c : shortUrl.toCharArray()) {
            id = id * BASE + ALPHABET.indexOf(c);
        }
        return id;
    }
}
```

**Architecture:**
```
Client
  ↓
Load Balancer
  ↓
App Servers (Spring Boot)
  ↓
Cache (Redis) ← → Database (MySQL)
```

### 10.4 CAP Theorem

**Q: Explain CAP Theorem.**

CAP states that a distributed system can provide only 2 of 3 guarantees:
- **Consistency**: All nodes see same data
- **Availability**: Every request gets response
- **Partition Tolerance**: System works despite network failures

**Examples:**
- **CA**: Traditional RDBMS (MySQL, PostgreSQL)
- **CP**: MongoDB, HBase, Redis
- **AP**: Cassandra, DynamoDB, CouchDB

### 10.5 Database Selection

**SQL vs NoSQL:**

| Use SQL When | Use NoSQL When |
|--------------|----------------|
| ACID transactions needed | High scalability needed |
| Complex queries | Flexible schema |
| Structured data | Unstructured data |
| Relationships important | High write throughput |

**NoSQL Types:**
1. **Document**: MongoDB, CouchDB
2. **Key-Value**: Redis, DynamoDB
3. **Column-Family**: Cassandra, HBase
4. **Graph**: Neo4j, Amazon Neptune

---

## 11. Security

### 11.1 Spring Security Basics

**Configuration:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) 
            throws Exception {
        http
            .csrf().disable()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .httpBasic();
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 11.2 JWT Authentication

**Q: How does JWT authentication work?**

**Flow:**
1. Client sends credentials
2. Server validates and generates JWT
3. Client stores JWT (localStorage/cookie)
4. Client sends JWT in Authorization header
5. Server validates JWT and processes request

**Implementation:**
```java
@Service
public class JwtService {
    private static final String SECRET = "your-secret-key";
    private static final long EXPIRATION = 86400000; // 24 hours
    
    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION))
            .signWith(SignatureAlgorithm.HS256, SECRET)
            .compact();
    }
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public boolean validateToken(String token, String username) {
        return username.equals(extractUsername(token)) 
            && !isTokenExpired(token);
    }
    
    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
}

// JWT Filter
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        
        String header = request.getHeader("Authorization");
        
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            String username = jwtService.extractUsername(token);
            
            if (username != null && SecurityContextHolder
                    .getContext().getAuthentication() == null) {
                
                if (jwtService.validateToken(token, username)) {
                    UsernamePasswordAuthenticationToken auth = 
                        new UsernamePasswordAuthenticationToken(
                            username, null, authorities);
                    SecurityContextHolder.getContext()
                        .setAuthentication(auth);
                }
            }
        }
        
        chain.doFilter(request, response);
    }
}
```

### 11.3 OAuth2 Flow

**Authorization Code Flow:**
1. Client redirects to Authorization Server
2. User logs in and grants permission
3. Auth Server sends authorization code
4. Client exchanges code for access token
5. Client uses access token to access resources

**Spring Security OAuth2:**
```java
@Configuration
public class OAuth2Config {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        http
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/oauth2/authorization/google")
                .defaultSuccessUrl("/home")
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt()
            );
        return http.build();
    }
}
```

### 11.4 Security Best Practices

1. **Password Security**
   - Use BCrypt/Argon2
   - Minimum length requirements
   - No plain text storage

2. **Input Validation**
   ```java
   @Valid @RequestBody UserRequest request
   ```

3. **CSRF Protection**
   - Enable for state-changing operations
   - Use CSRF tokens

4. **HTTPS Only**
   - Encrypt data in transit

5. **Rate Limiting**
   ```java
   @RateLimiter(name = "api")
   public ResponseEntity<?> apiEndpoint() { }
   ```

6. **SQL Injection Prevention**
   - Use PreparedStatement
   - Use JPA/Hibernate

---

## 12. Messaging & Event-Driven Architecture

### 12.1 Apache Kafka Basics

**Q: Explain Kafka architecture.**

**Components:**
- **Producer**: Sends messages to topics
- **Consumer**: Reads messages from topics
- **Topic**: Category of messages
- **Partition**: Topic divided into partitions
- **Broker**: Kafka server
- **ZooKeeper**: Maintains cluster state (not needed in Kafka 3.0+)

**Producer:**
```java
@Configuration
public class KafkaProducerConfig {
    @Bean
    public ProducerFactory<String, Order> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public KafkaTemplate<String, Order> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

@Service
public class OrderProducer {
    @Autowired
    private KafkaTemplate<String, Order> kafkaTemplate;
    
    public void sendOrder(Order order) {
        kafkaTemplate.send("orders", order.getId().toString(), order);
    }
}
```

**Consumer:**
```java
@Service
public class OrderConsumer {
    
    @KafkaListener(topics = "orders", groupId = "order-processing")
    public void consumeOrder(Order order) {
        System.out.println("Received order: " + order.getId());
        processOrder(order);
    }
}
```

### 12.2 RabbitMQ

**Q: Kafka vs RabbitMQ - when to use which?**

| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| **Type** | Log-based, append-only | Traditional message queue |
| **Message Retention** | Configurable (days) | Deleted after consumption |
| **Throughput** | Very high | Moderate |
| **Use Case** | Event streaming, logs | Task distribution, RPC |
| **Order Guarantee** | Within partition | Per queue |

**RabbitMQ Example:**
```java
@Configuration
public class RabbitMQConfig {
    @Bean
    public Queue orderQueue() {
        return new Queue("order-queue");
    }
    
    @Bean
    public Exchange orderExchange() {
        return new DirectExchange("order-exchange");
    }
    
    @Bean
    public Binding binding(Queue queue, Exchange exchange) {
        return BindingBuilder.bind(queue)
            .to(exchange)
            .with("order.created")
            .noargs();
    }
}

// Publisher
@Service
public class OrderPublisher {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void publishOrder(Order order) {
        rabbitTemplate.convertAndSend("order-exchange", 
            "order.created", order);
    }
}

// Consumer
@Service
public class OrderConsumer {
    @RabbitListener(queues = "order-queue")
    public void receiveOrder(Order order) {
        processOrder(order);
    }
}
```

### 12.3 Event-Driven Architecture

**Benefits:**
- Loose coupling
- Scalability
- Asynchronous processing
- Real-time processing

**Patterns:**
1. **Event Notification**: Simple notification
2. **Event-Carried State Transfer**: Event contains full state
3. **Event Sourcing**: Store events, not current state
4. **CQRS**: Separate read and write models

---

## 13. DevOps, CI/CD & Containerization

### 13.1 Docker Basics

**Q: What is Docker and why use it?**

**Benefits:**
- Consistent environments
- Isolation
- Portability
- Easy scaling

**Dockerfile for Java App:**
```dockerfile
# Multi-stage build
FROM maven:3.8.4-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM openjdk:17-jdk-slim
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Docker Compose:**
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_HOST=mysql
    depends_on:
      - mysql
      - redis
  
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: myapp
    volumes:
      - mysql-data:/var/lib/mysql
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mysql-data:
```

### 13.2 Kubernetes Basics

**Q: Explain Kubernetes architecture.**

**Components:**
- **Pod**: Smallest unit, contains container(s)
- **Service**: Exposes pods
- **Deployment**: Manages pod replicas
- **ConfigMap/Secret**: Configuration
- **Ingress**: HTTP(S) routing

**Deployment YAML:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: mysql
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### 13.3 CI/CD with Jenkins

**Jenkinsfile:**
```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8'
        jdk 'JDK 17'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/user/repo.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
                sh 'docker tag myapp:${BUILD_NUMBER} myapp:latest'
            }
        }
        
        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh 'docker login -u $USER -p $PASS'
                    sh 'docker push myapp:${BUILD_NUMBER}'
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl set image deployment/myapp myapp=myapp:${BUILD_NUMBER}'
                sh 'kubectl rollout status deployment/myapp'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend color: 'good', 
                      message: "Build ${BUILD_NUMBER} succeeded"
        }
        failure {
            slackSend color: 'danger', 
                      message: "Build ${BUILD_NUMBER} failed"
        }
    }
}
```

### 13.4 GitHub Actions

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    
    - name: Build with Maven
      run: mvn clean package
    
    - name: Run tests
      run: mvn test
    
    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Push to Docker Hub
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push myapp:${{ github.sha }}
```

---

## 14. Testing

### 14.1 JUnit 5

**Basic Test:**
```java
@SpringBootTest
class UserServiceTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    @DisplayName("Should create user successfully")
    void testCreateUser() {
        // Arrange
        UserRequest request = new UserRequest("John", "john@example.com");
        
        // Act
        UserDTO created = userService.createUser(request);
        
        // Assert
        assertNotNull(created.getId());
        assertEquals("John", created.getName());
        assertEquals("john@example.com", created.getEmail());
    }
    
    @Test
    void testGetUserById_NotFound() {
        assertThrows(ResourceNotFoundException.class, () -> {
            userService.getUserById(999L);
        });
    }
    
    @ParameterizedTest
    @ValueSource(strings = {"", "   ", "invalid-email"})
    void testInvalidEmail(String email) {
        UserRequest request = new UserRequest("John", email);
        assertThrows(ValidationException.class, () -> {
            userService.createUser(request);
        });
    }
}
```

### 14.2 Mockito

**Q: How do you use Mockito for unit testing?**

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private ProductService productService;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void testCreateOrder() {
        // Arrange
        Order order = new Order(1L, "Item", 100.0);
        when(orderRepository.save(any(Order.class)))
            .thenReturn(order);
        when(productService.getProduct(1L))
            .thenReturn(new Product(1L, "Item", 100.0));
        
        // Act
        Order created = orderService.createOrder(order);
        
        // Assert
        assertNotNull(created);
        verify(orderRepository, times(1)).save(any(Order.class));
        verify(productService, times(1)).getProduct(1L);
    }
    
    @Test
    void testGetOrder_CallsRepository() {
        Long orderId = 1L;
        when(orderRepository.findById(orderId))
            .thenReturn(Optional.of(new Order()));
        
        orderService.getOrderById(orderId);
        
        verify(orderRepository).findById(orderId);
        verifyNoMoreInteractions(orderRepository);
    }
}
```

**Argument Captors:**
```java
@Test
void testSaveOrder() {
    ArgumentCaptor<Order> orderCaptor = 
        ArgumentCaptor.forClass(Order.class);
    
    orderService.createOrder(new OrderRequest());
    
    verify(orderRepository).save(orderCaptor.capture());
    Order capturedOrder = orderCaptor.getValue();
    
    assertEquals(OrderStatus.PENDING, capturedOrder.getStatus());
}
```

### 14.3 Integration Testing

```java
@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(locations = "classpath:application-test.properties")
class UserControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void testCreateUser_Success() throws Exception {
        UserRequest request = new UserRequest("John", "john@example.com");
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.name").value("John"))
                .andExpect(jsonPath("$.email").value("john@example.com"))
                .andDo(print());
    }
    
    @Test
    void testGetUser_NotFound() throws Exception {
        mockMvc.perform(get("/api/users/999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.message")
                    .value("User not found"));
    }
}
```

### 14.4 Test Coverage

**Best Practices:**
- Aim for 80%+ code coverage
- Focus on business logic
- Don't just test for coverage
- Test edge cases and error conditions

---

## 15. Performance Optimization & Memory Management

### 15.1 Java Memory Model

**Q: Explain JVM memory structure.**

**Memory Areas:**
1. **Heap**: Objects and arrays
   - Young Generation (Eden + Survivor)
   - Old Generation
2. **Stack**: Method calls and local variables
3. **Method Area**: Class metadata
4. **PC Register**: Current instruction
5. **Native Method Stack**: Native methods

### 15.2 Garbage Collection

**Q: Explain garbage collection in Java.**

**GC Types:**
1. **Serial GC**: Single-threaded
2. **Parallel GC**: Multi-threaded
3. **G1 GC** (default Java 9+): Balanced
4. **ZGC**: Ultra-low latency

**Tuning:**
```bash
# Heap size
-Xms2g -Xmx4g

# G1 GC
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# GC Logging
-Xlog:gc*:file=gc.log
```

**Avoid Memory Leaks:**
- Close resources (use try-with-resources)
- Clear collections when done
- Avoid static collections
- Weak references for caches

### 15.3 Performance Optimization

**1. Database Optimization:**
```java
// Bad - N+1 problem
users.forEach(user -> {
    user.getOrders().size();
});

// Good - Fetch join
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

**2. Caching:**
```java
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }
    
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
    
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
}
```

**3. Connection Pooling:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
```

**4. Async Processing:**
```java
@Service
public class EmailService {
    
    @Async
    public CompletableFuture<Void> sendEmail(String to, String body) {
        // Send email
        return CompletableFuture.completedFuture(null);
    }
}
```

**5. Use Appropriate Data Structures:**
```java
// Fast lookup
Map<String, User> userMap = new HashMap<>();

// Sorted
TreeMap<String, User> sortedUsers = new TreeMap<>();

// Thread-safe
ConcurrentHashMap<String, User> concurrentMap = new ConcurrentHashMap<>();
```

### 15.4 Profiling Tools

- **JProfiler**: Memory and CPU profiling
- **VisualVM**: Monitor JVM
- **YourKit**: Performance profiling
- **JConsole**: JMX monitoring
- **Spring Boot Actuator**: Production metrics

---

## 16. Cloud Services (AWS Basics)

### 16.1 Core AWS Services

**Q: What AWS services would you use for a Java backend application?**

**Compute:**
- **EC2**: Virtual servers
- **ECS/EKS**: Container orchestration
- **Lambda**: Serverless functions
- **Elastic Beanstalk**: PaaS for Java apps

**Storage:**
- **S3**: Object storage
- **EBS**: Block storage for EC2
- **RDS**: Managed relational databases
- **DynamoDB**: NoSQL database

**Networking:**
- **VPC**: Virtual private cloud
- **ALB/NLB**: Load balancers
- **Route 53**: DNS
- **CloudFront**: CDN

**Others:**
- **SQS**: Message queue
- **SNS**: Pub/sub notifications
- **ElastiCache**: Redis/Memcached
- **CloudWatch**: Monitoring

### 16.2 Deploying Spring Boot to AWS

**Elastic Beanstalk:**
```bash
# Install EB CLI
pip install awsebcli

# Initialize
eb init -p java-17 my-app

# Create environment
eb create prod-env

# Deploy
eb deploy
```

**ECS with Docker:**
```json
// Task definition
{
  "family": "myapp",
  "containerDefinitions": [{
    "name": "myapp",
    "image": "myapp:latest",
    "memory": 512,
    "cpu": 256,
    "portMappings": [{
      "containerPort": 8080,
      "protocol": "tcp"
    }],
    "environment": [{
      "name": "SPRING_PROFILES_ACTIVE",
      "value": "prod"
    }]
  }]
}
```

---

## 17. Behavioral Interview Questions

### 17.1 STAR Method

**Structure:**
- **S**ituation: Context
- **T**ask: What needed to be done
- **A**ction: What you did
- **R**esult: Outcome and learning

### 17.2 Common Questions

**Q: Tell me about a challenging bug you resolved.**

**Example Answer:**
"In my previous role, we had a production issue where our API response time increased from 100ms to 5 seconds during peak hours.

**Situation**: The service was handling payment processing for an e-commerce platform, affecting thousands of transactions.

**Task**: I needed to identify the root cause and fix it without causing downtime.

**Action**: 
1. First, I analyzed the logs and metrics in CloudWatch
2. Discovered database connection pool exhaustion
3. Identified a missing index on a frequently queried table
4. Implemented connection pool optimization (increased max connections from 10 to 30)
5. Added the missing database index
6. Implemented caching for frequently accessed data

**Result**: Response times returned to under 200ms. We processed 50% more transactions without issues. I also created monitoring alerts to prevent similar issues."

**Q: Describe a time you disagreed with a technical decision.**

**Example Answer:**
"During a microservices migration, the team wanted to use synchronous REST calls between all services.

**Situation**: We were breaking a monolith into microservices.

**Task**: Ensure reliable inter-service communication.

**Action**:
1. Presented data showing REST calls created tight coupling
2. Demonstrated how async messaging with Kafka would improve reliability
3. Created a POC showing both approaches
4. Highlighted benefits: loose coupling, better scalability, fault tolerance

**Result**: Team agreed to use Kafka for non-critical operations and REST for synchronous needs. System handled 3x more load with fewer failures."

**Q: How do you handle conflicting priorities?**

**Example Answer:**
"I use a combination of business impact analysis and communication:

1. **Assess Impact**: Evaluate each task's business value
2. **Communicate**: Discuss with stakeholders
3. **Negotiate**: Be transparent about trade-offs
4. **Document**: Keep decisions recorded
5. **Deliver**: Focus on highest priority first

Example: When asked to build a new feature while fixing critical bugs, I communicated that bugs affect existing users (higher priority) while new feature affects potential users. We agreed to fix bugs first, then feature in next sprint."

### 17.3 Leadership Questions

**Q: How do you mentor junior developers?**

**Example Answer:**
1. **Pair Programming**: Work together on tasks
2. **Code Reviews**: Provide constructive feedback
3. **Knowledge Sharing**: Weekly tech talks
4. **Encourage Questions**: Create safe environment
5. **Gradual Complexity**: Start simple, increase difficulty
6. **Documentation**: Maintain good docs and examples

### 17.4 Questions to Ask Interviewer

**Technical:**
- What's your tech stack and why?
- How do you handle production incidents?
- What's your deployment frequency?
- How do you ensure code quality?

**Team:**
- What does a typical day look like?
- How is the team structured?
- What's the code review process?

**Growth:**
- What are opportunities for learning?
- How do you support professional development?
- What's the career path for this role?

**Product:**
- What's the biggest technical challenge?
- What's the product roadmap?
- How do you prioritize features?

---

## 18. Coding Challenges & Algorithms

### 18.1 Common Patterns

**1. Two Pointers:**
```java
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) {
            return false;
        }
        left++;
        right--;
    }
    return true;
}
```

**2. Sliding Window:**
```java
public int maxSubarraySum(int[] arr, int k) {
    int maxSum = 0, windowSum = 0;
    
    // First window
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }
    maxSum = windowSum;
    
    // Slide window
    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k];
        maxSum = Math.max(maxSum, windowSum);
    }
    
    return maxSum;
}
```

**3. HashMap for Frequency:**
```java
public char firstNonRepeatingChar(String s) {
    Map<Character, Integer> freq = new HashMap<>();
    
    for (char c : s.toCharArray()) {
        freq.put(c, freq.getOrDefault(c, 0) + 1);
    }
    
    for (char c : s.toCharArray()) {
        if (freq.get(c) == 1) {
            return c;
        }
    }
    
    return '\0';
}
```

### 18.2 Data Structure Problems

**Implement LRU Cache:**
```java
class LRUCache {
    private final int capacity;
    private final Map<Integer, Node> cache;
    private final Node head, tail;
    
    class Node {
        int key, value;
        Node prev, next;
        Node(int k, int v) { key = k; value = v; }
    }
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        cache = new HashMap<>();
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }
    
    public int get(int key) {
        if (!cache.containsKey(key)) return -1;
        Node node = cache.get(key);
        remove(node);
        insertAtHead(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        if (cache.containsKey(key)) {
            remove(cache.get(key));
        }
        
        if (cache.size() == capacity) {
            cache.remove(tail.prev.key);
            remove(tail.prev);
        }
        
        Node node = new Node(key, value);
        cache.put(key, node);
        insertAtHead(node);
    }
    
    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    private void insertAtHead(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}
```

### 18.3 Algorithm Problems

**Binary Search:**
```java
public int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return -1;
}
```

**Merge Sort:**
```java
public void mergeSort(int[] arr, int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;
        mergeSort(arr, left, mid);
        mergeSort(arr, mid + 1, right);
        merge(arr, left, mid, right);
    }
}

private void merge(int[] arr, int left, int mid, int right) {
    int[] temp = new int[right - left + 1];
    int i = left, j = mid + 1, k = 0;
    
    while (i <= mid && j <= right) {
        if (arr[i] <= arr[j]) {
            temp[k++] = arr[i++];
        } else {
            temp[k++] = arr[j++];
        }
    }
    
    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];
    
    System.arraycopy(temp, 0, arr, left, temp.length);
}
```

### 18.4 Practice Resources

**Platforms:**
- LeetCode (focus on Medium/Hard)
- HackerRank
- CodeSignal
- AlgoExpert

**Must-Do Problems:**
1. Two Sum
2. Reverse Linked List
3. Valid Parentheses
4. Merge Intervals
5. Binary Tree Level Order Traversal
6. LRU Cache
7. Design URL Shortener
8. Implement HashMap
9. Find Kth Largest Element
10. Serialize/Deserialize Binary Tree

---

## 19. Interview Day Preparation Checklist

### 19.1 One Week Before

- [ ] Review all sections of this guide
- [ ] Practice 5 coding problems daily
- [ ] Review your recent projects
- [ ] Prepare STAR stories for behavioral questions
- [ ] Review company's tech stack and products
- [ ] Prepare questions to ask interviewer

### 19.2 Day Before

- [ ] Light review (don't cram)
- [ ] Prepare your workspace (if remote)
- [ ] Test camera and microphone
- [ ] Print resume copies (if in-person)
- [ ] Plan your route (if in-person)
- [ ] Get good sleep

### 19.3 Interview Day

**Morning:**
- [ ] Light breakfast
- [ ] Review key concepts (30 min)
- [ ] Arrive 10-15 min early

**During Interview:**
- [ ] Listen carefully to questions
- [ ] Think before answering
- [ ] Ask clarifying questions
- [ ] Explain your thought process
- [ ] Admit if you don't know something
- [ ] Stay calm and confident

**Technical Round:**
1. **Clarify requirements** (5 min)
2. **Discuss approach** (5 min)
3. **Write code** (20 min)
4. **Test with examples** (5 min)
5. **Optimize if time** (5 min)

### 19.4 Mock Interview Practice

**With Friend/Peer:**
- Practice explaining concepts
- Time yourself
- Get feedback on communication

**Online Platforms:**
- Pramp
- interviewing.io
- LeetCode Mock Interviews

---

## Conclusion

This comprehensive guide covers all essential topics for senior Java backend developer interviews in 2025. Remember:

### Key Success Factors:

1. **Depth Over Breadth**: Better to know fewer topics deeply than many superficially
2. **Real Experience**: Always tie answers to your actual experience
3. **System Thinking**: Show you can design scalable, maintainable systems
4. **Communication**: Explain your thought process clearly
5. **Continuous Learning**: Show enthusiasm for learning new technologies

### Study Plan:

**Week 1-2**: Core Java, OOPs, Collections
**Week 3-4**: Spring Boot, REST APIs, Databases
**Week 5-6**: Microservices, System Design
**Week 7-8**: Coding practice, Mock interviews

### Final Tips:

- Practice coding on whiteboard/paper
- Time yourself on problems
- Review your mistakes
- Stay updated with latest Java features
- Read tech blogs and articles
- Contribute to open source if possible

**Good luck with your interviews! 🚀**

---

## Quick Reference - Must Know Topics

### Top 20 Most Important Topics:
1. Java 8+ Features (Streams, Lambda, Optional)
2. Spring Boot & Spring Framework
3. REST API Design
4. Microservices Architecture
5. Database Design & Hibernate/JPA
6. Multithreading & Concurrency
7. Design Patterns (Singleton, Factory, Strategy, Observer)
8. System Design Basics
9. SOLID Principles
10. Exception Handling
11. Collections Framework
12. Spring Security & JWT
13. Kafka/RabbitMQ
14. Docker & Kubernetes Basics
15. CI/CD Pipelines
16. Testing (JUnit, Mockito)
17. Performance Optimization
18. Transaction Management
19. Caching Strategies
20. Cloud Services (AWS Basics)

### Time Management in Interview:
- **Introduction**: 5 min
- **Technical Questions**: 30-40 min
- **Coding Round**: 30-45 min
- **System Design**: 45-60 min
- **Behavioral**: 15-20 min
- **Your Questions**: 5-10 min

### Interview Red Flags to Avoid:
- ❌ Not asking clarifying questions
- ❌ Jumping to code without planning
- ❌ Poor communication
- ❌ No error handling in code
- ❌ Not testing code with examples
- ❌ Saying "I don't know" without trying
- ❌ Criticizing previous employers
- ❌ No questions for interviewer

**Remember: Interviews test not just your knowledge, but your problem-solving approach, communication skills, and how you handle pressure. Stay confident and be yourself!**

---

**Document Version:** 1.0  
**Last Updated:** November 13, 2025  
**Target Interview Date:** December 2025

*This guide is based on current industry trends and interview patterns from top companies in 2025. Keep updating your knowledge as technology evolves.*