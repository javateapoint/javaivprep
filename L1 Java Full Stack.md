# L1 Java Full Stack

# 1. Square and Print in Order Using Java 8
Answer: Since the list is an ordered collection and you are using a sequential stream, the encounter order is preserved. For example:

```java
import java.util.List;

public class SquarePrinter {
    public static void main(String[] args) {
        List<Integer> data = List.of(1, 4, 5, 6, 3, 8);
        data.stream()
            .map(i -> i * i)
            .forEach(System.out::println);
    }
}
```
Explanation: Here, List.of(...) creates an immutable list with a defined order. The stream pipeline uses map (an intermediate operation) to square each number. When the terminal operation (forEach) is called, it processes the elements in the list’s order. If you switched to a parallel stream, you’d need to use forEachOrdered() to ensure order preservation.

# 2. Write Code to Find the First Non-Repeated Character from an Array of Strings Using Java 8
Answer: Assuming you want to find the first non-repeated character from a single String, you can leverage streams and a LinkedHashMap (which preserves insertion order):

```java
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public class FirstNonRepeatedChar {
    public static void main(String[] args) {
        String input = "swiss";
        Character result = input.chars()
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()))
            .entrySet().stream()
            .filter(e -> e.getValue() == 1)
            .map(Map.Entry::getKey)
            .findFirst()
            .orElse(null);
        System.out.println("First non-repeated character: " + result);
    }
}
```
Explanation:

Step 1: Convert the string to a stream of characters.

Step 2: Group by character into a LinkedHashMap to preserve order while counting occurrences.

Step 3: Stream the entries, filter to get the one with count 1, and pick the first one.

If the question meant “an array of strings” (i.e. process each string or all strings combined), you’d first adjust by flattening the streams accordingly.

# 3. Java 8 Features
Answer: Java 8 introduced several transformative features:

* Lambda Expressions:  Allow concise inline implementations of interfaces with a single abstract method.

* Streams API: Enable functional-style operations (filter, map, reduce) on collections.

* Default and Static Methods in Interfaces: Let you add methods to interfaces without breaking existing implementations.

* Functional Interfaces: Annotated with @FunctionalInterface (like Runnable, Callable), serving as targets for lambda expressions.

* Optional Class: Helps avoid explicit null checks.

* New Date/Time API (java.time): Provides a modern approach to handling dates and times.

* Method References: Simplify lambda expressions further by referring directly to methods.

* Parallel Streams: Provide an easy way to process data concurrently.

Explanation: With these features, Java 8 shifted much of the language’s idioms toward functional programming, improving expressiveness and potentially performance.

# 4. Why Were Default Methods Introduced in Interfaces?
Answer: Default methods were introduced primarily to evolve interfaces without breaking backward compatibility. When Java needed to add new functionality (for example, methods to support lambda expressions and stream operations), default methods allowed interface designers to provide a base implementation that host classes could inherit without forcing all implementers to change.

Explanation: They also help in providing “mix-in” types, enabling multiple inheritance of behavior even though Java does not support multiple inheritance of state.

# 5. What Is a Functional Interface and Why?
Answer: A functional interface is an interface that contains exactly one abstract method. This single abstract method makes it possible to use lambda expressions or method references as an implementation of that interface.

Example: The Runnable interface is functional because it has only one method: void run(). Explanation: This design pattern enables more concise and expressive code when working with callbacks, event handlers, or any scenario where you want to pass behavior as a parameter.

# 6. Can We Extend One Functional Interface from Another?
Answer: Yes, one functional interface can extend another as long as the resulting interface still has just one abstract method. Extra abstract methods or ambiguity in method signatures would violate the single abstract method requirement.

Explanation: This allows you to build more specific variants of standard functional interfaces without losing the benefits that let you easily use lambda expressions.

# 7. What Is ClassNotFoundException and ClassDefFoundError?
Answer:

ClassNotFoundException: A checked exception thrown when an application tries to load a class by name (e.g., using Class.forName()) and the class cannot be found.

NoClassDefFoundError: An error (not an exception) thrown when the JVM or a class loader tries to load a class that was present during compile time but is missing at runtime.

Explanation: This distinction is important because while you can catch a ClassNotFoundException, a NoClassDefFoundError indicates a fundamental problem in the classpath or deployment that generally cannot be recovered from at runtime.

# 8. What Is a ClassLoader?
Answer: A ClassLoader is an integral part of the Java runtime system responsible for dynamically loading Java classes into the JVM. It translates the binary representation of a class from various sources (files, JARs, network) into a runtime class.

Explanation: Class loaders also implement namespaces, can perform security checks, and allow multiple versions of classes (in some cases) to coexist.

# 9. How Many Types of ClassLoaders Are There?
Answer: Typically, there are three primary class loaders:

Bootstrap ClassLoader: Loads core Java classes from the rt.jar or similar.

Extension (Platform) ClassLoader: Loads classes from the extensions directory.

Application (System) ClassLoader: Loads classes from the application’s classpath.

Explanation: You can also create your own custom class loaders if needed. This layered approach keeps the Java runtime flexible and secure.

# 10. What Is a Heap Dump?
Answer: A heap dump is a snapshot of all the objects and classes in the JVM’s memory at a particular moment. It is useful for diagnosing memory leaks, understanding object retention, and profiling the memory usage of an application.

Explanation: Tools like Eclipse MAT or VisualVM can analyze heap dumps to pinpoint problematic memory consumption.

# 11. What Changes Happened in Java 8 as Part of Memory?
Answer: While Java 8 did not radically change the mechanics of memory management, one of the most significant improvements was the removal of the Permanent Generation (PermGen) and the introduction of Metaspace. In Metaspace, class metadata is stored in native memory and grows dynamically, reducing the need for manual tuning in many scenarios.

Explanation: This change alleviated many of the typical out-of-memory issues related to class metadata and improved the overall stability of long-running Java applications.

# 12. PermGen vs. Metaspace
Answer:

PermGen (Permanent Generation): A fixed-size memory space in pre–Java 8 JVMs used to store class metadata, which often required manual tuning and could lead to OutOfMemoryError if it filled up.

Metaspace: Introduced in Java 8, it stores class metadata in native memory. By default, it grows as needed (subject to an optional maximum), reducing the configuration issues that PermGen presented.

Explanation: This switch was a major improvement in how the JVM managed class-level memory and enhanced application stability.

# 13. How Does HashMap Work?
Answer: A HashMap is a hash table–based implementation of the Map interface. It:

Uses an array of nodes (buckets) where each node is stored based on its hash value.

When inserting, it computes the hash code of the key to determine the bucket.

In the event of collisions (multiple keys mapping to the same bucket), it stores entries in a linked list (or, for Java 8 and later, in a balanced tree if the bucket size exceeds a threshold).

Explanation: This design offers O(1) average-time complexity for read and write operations, though the worst-case can be O(n) if many keys collide.

# 14. Fail-fast vs. Fail-safe
Answer:

Fail-fast Iterators: Immediately throw a ConcurrentModificationException if the underlying collection is modified during iteration (e.g., ArrayList’s iterator).

Fail-safe Iterators: Operate on a clone of the underlying collection (e.g., CopyOnWriteArrayList), so they do not throw exceptions if the collection is modified but may not reflect real-time changes.

Explanation: Fail-fast iterators help catch concurrent modification errors early, while fail-safe iterators emphasize stability at the expense of real-time accuracy.

# 15. Write a Singleton Class
Answer: A common, thread-safe implementation is the double-checked locking pattern:

```java
public class Singleton {
    // volatile to ensure proper visibility of changes across threads
    private static volatile Singleton instance;

    private Singleton() {
        // Prevent instantiation via reflection (optional safeguard)
        if (instance != null) {
            throw new IllegalStateException("Already initialized");
        }
    }

    public static Singleton getInstance() {
        if (instance == null) {               // First check (no locking)
            synchronized (Singleton.class) {
                if (instance == null) {       // Second check (with locking)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
Explanation: This implementation minimizes synchronization overhead while ensuring that only one instance is created. The use of volatile guarantees visibility among threads.

# 16. How to Break the Singleton Pattern and the Best Way to Create a Singleton
Answer: Singletons can be broken by:

Reflection: Calling the constructor directly.

Serialization: Deserializing an instance can create a new object.

Cloning: Invoking clone() (if not handled properly).

Multiple ClassLoaders: Leading to different copies in the same JVM.

Best Practice: Using an enum singleton is the most robust solution. For example:

```java
public enum SingletonEnum {
    INSTANCE;
    // you can add methods if needed
}
```
Explanation: Enum singletons are inherently thread-safe, protect against reflection and serialization attacks, and are succinct.

# 17. What Is an Enum and Can We Have a Constructor for Enums?
Answer: An enum in Java is a special type that represents a group of constants. Yes, enums can have constructors—but they are implicitly private. This allows each constant to be associated with specific data.

Explanation: Enum constructors are executed once per constant and cannot be called elsewhere, making them ideal for simple and safe instantiation patterns.

# 18. Spring vs. Spring Boot
Answer:

Spring Framework: A comprehensive ecosystem providing dependency injection, AOP, data access, and more—but requires significant configuration.

Spring Boot: Built on top of Spring, it offers auto-configuration, embedded servers, and opinionated defaults to simplify development and deployment.

Explanation: Spring Boot significantly reduces boilerplate and configuration overhead while still leveraging the powerful Spring ecosystem.

# 19. What Are Spring Actuators?
Answer: Spring Boot Actuators add production-ready features such as monitoring, health checks, and metrics endpoints (e.g., /actuator/health, /actuator/metrics) to your application. They offer insight into your application’s runtime behavior.

Explanation: They help in proactive monitoring and managing application states, making it easier to integrate with external monitoring tools.

# 20. How to Check Metrics
Answer: Metrics can be checked using:

Actuator Endpoints: For example, accessing /actuator/metrics provides a comprehensive list of application metrics.

JMX Integration: Java Management Extensions allow you to monitor properties through tools like JConsole.

External Monitoring Tools: Integrations with Prometheus, Grafana, or APM tools (New Relic, AppDynamics) also provide dashboarded insights.

Explanation: The choice depends on your environment and whether you need simple internal checks or comprehensive external monitoring.

# 21. Use of DevTools
Answer: Spring Boot DevTools enhance the development experience by providing:

Automatic application restarts on code changes.

LiveReload support for web resources.

Enhanced error messages and debugging tools.

Explanation: These features speed up development cycles by reducing manual rebuilds and providing immediate feedback during development.

# 22. How Does an API Gateway Work?
Answer: An API Gateway serves as a single entry point to a microservices architecture. It:

Routes client requests to the appropriate microservice.

Performs cross-cutting concerns such as authentication, request throttling, logging, and load balancing.

May transform protocols and aggregate multiple responses.

Explanation: It abstracts the microservice topology from external clients, simplifying API consumption and enhancing security.

# 23. How Does a Circuit Breaker Work?
Answer: The circuit breaker pattern monitors remote calls. When the failure rate exceeds a threshold, it “opens” the circuit to prevent further calls, thereby giving a failing service time to recover. Once conditions improve, the circuit may move to a “half-open” state and eventually “close” if the service stabilizes. Tools: Libraries like Hystrix or Resilience4j implement this pattern.

Explanation: This pattern protects your system from cascading failures and improves resilience in distributed architectures.

# 24. How Do Microservices (MS) Communicate with Each Other?
Answer: Microservices communicate via:

Synchronous Protocols: RESTful HTTP/HTTPS, gRPC.

Asynchronous Messaging: Message brokers such as Kafka, RabbitMQ, or AMQP.

API Gateways: Acting as intermediaries.

Event-Driven Architecture: Using events to decouple services.

Explanation: The choice of communication pattern depends on latency, scalability, and reliability requirements.

# 25. How to Check the Performance, CPU Utilization, and Memory Consumption of a Microservice?
Answer: You can use profiling and monitoring tools such as:

JMX-based Tools: (JConsole, VisualVM).

Java Flight Recorder & Mission Control: For detailed profiling.

Spring Boot Actuator Metrics: Combined with monitoring systems like Prometheus and Grafana.

APM Tools: New Relic, AppDynamics, or Dynatrace for end-to-end monitoring.

Explanation: These tools help pinpoint performance bottlenecks and resource usage anomalies in production.

# 26. What Will Happen When 500 Requests Hit Your API?
Answer: The behavior depends on your system design:

If properly sized: Your thread pool, connection management, and load balancer should handle the load by queuing or processing concurrently.

If overwhelmed: You might see increased latency, thread exhaustion, or even request rejection (HTTP 429). Mitigations: Load balancing, rate limiting, asynchronous processing, and auto-scaling.

Explanation: It’s crucial to design with concurrency in mind, with proper monitoring and fallback mechanisms (like circuit breakers).

# 27. How to Increase the Performance of SPA Applications
Answer: For Single Page Applications (SPAs):

Optimize Asset Loading: Use lazy loading, code splitting, and bundling.

Caching: Utilize browser caching, service workers, and Content Delivery Networks (CDNs).

Efficient State Management: Minimize unnecessary re-renders.

Backend Optimizations: Ensure responsive REST APIs or GraphQL endpoints.

Explanation: A well-optimized SPA and accompanying backend work together to provide a snappy user experience.

# 28. How Do Parent and Child Components Communicate and Pass Data?
Answer: In modern front-end frameworks:

Angular: Uses @Input() for parent-to-child data binding and @Output() with EventEmitters for child-to-parent communication.

React: Parent components pass data through props and may use callback functions to receive data from children.

Explanation: Choosing the right communication method helps maintain component encapsulation and promotes a unidirectional data flow.

# 29. What Is a Pipe?
Answer: In Angular, a pipe is a mechanism to transform data in templates. For example, the built-in date, currency, or uppercase pipes take input values and return transformed output.

Explanation: Pipes encapsulate transformation logic so that view templates remain clean and declarative.

# 30. How Is Data Fetched from the Backend to the UI?
Answer: Data is typically fetched using:

HTTP Clients: Such as Angular’s HttpClient, the Fetch API, or Axios in React.

GraphQL: For efficient data querying.

WebSockets: For real-time data.

Explanation: The choice depends on the application’s real-time needs, the volume of data, and whether a request/response or streaming model fits best.

# 31. What Is RxJS?
Answer: RxJS (Reactive Extensions for JavaScript) is a library for reactive programming using Observables, which makes it easier to handle asynchronous data streams (like events, HTTP requests, etc.) with operators such as map, filter, and reduce.

Explanation: RxJS enables you to compose asynchronous operations in a declarative fashion, leading to cleaner and more scalable UI logic.

# 32. What Is ExecutorService?
Answer: ExecutorService is a part of Java’s concurrency framework that provides a higher-level API for managing threads. It allows you to:

Submit asynchronous tasks.

Manage thread pools.

Schedule tasks for future execution.

Explanation: It abstracts raw thread management so you can focus on task execution and proper shutdown of resources.

# 33. What Is Immutability in Java?
Answer: Immutability refers to objects whose state cannot be changed after their creation. Immutable objects are inherently thread-safe and reduce side effects.

Explanation: They simplify reasoning about code since the object’s state remains consistent throughout its lifecycle.

# 34. How to Create an Immutable Class in Java
Answer: A typical pattern for creating an immutable class:

Declare the class as final to prevent subclassing.

Mark all fields as private and final.

Do not provide setters.

Initialize all properties via the constructor.

If a field holds a reference to a mutable object, return a defensive copy in any getter.

Example:

```java
public final class ImmutablePerson {
    private final String name;
    private final int age;
    
    public ImmutablePerson(String name, int age) {
        // For mutable fields, create defensive copies
        this.name = name;
        this.age = age;
    }
    
    public String getName() {
        return name;
    }
    
    public int getAge() {
        return age;
    }
}
```
# 35. How to Check if a Memory Leak Has Occurred in an Application
Answer: You can diagnose memory leaks by:

Monitoring Tools: Using JMX-based tools (JConsole, VisualVM) to observe heap usage.

Profilers: Java Flight Recorder or Mission Control for detailed runtime metrics.

Heap Dumps: Analyzing heap dumps with tools like Eclipse MAT to pinpoint objects that are not being released.

Logging & Metrics: Integrating with APM tools (New Relic, AppDynamics) for automated anomaly detection.

Explanation: Regular monitoring and profiling are key. Analyzing trends over time and comparing heap dumps can reveal unexpected object retention indicating potential leaks.

