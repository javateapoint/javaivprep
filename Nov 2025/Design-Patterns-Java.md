# Design Patterns in Java — A Practical, Interview-Ready Deep Dive

> Goal: Make design patterns easy to understand, hard to forget, and practical to apply in real Java systems.

---

## Table of Contents

- [1. Introduction](#1-introduction-to-design-patterns)
- [2. Classification](#2-classification-of-design-patterns)
- [3. Creational Patterns](#3-creational-design-patterns-java)
  - [Singleton](#singleton)
  - [Factory Method](#factory-method)
  - [Abstract Factory](#abstract-factory)
  - [Builder](#builder)
  - [Prototype](#prototype)
- [4. Structural Patterns](#4-structural-design-patterns-java)
  - [Adapter](#adapter)
  - [Decorator](#decorator)
  - [Facade](#facade)
  - [Proxy](#proxy)
  - [Composite](#composite)
  - [Bridge](#bridge)
- [5. Behavioral Patterns](#5-behavioral-design-patterns-java)
  - [Strategy](#strategy)
  - [Observer](#observer)
  - [Command](#command)
  - [State](#state)
  - [Template Method](#template-method)
  - [Chain of Responsibility](#chain-of-responsibility)
  - [Iterator](#iterator)
  - [Mediator](#mediator)
- [6. Patterns vs SOLID](#6-design-patterns-vs-solid-principles)
- [7. Patterns in Real Java Frameworks](#7-design-patterns-in-real-java-frameworks)
- [8. Interview Questions](#8-most-asked-design-pattern-interview-questions)
- [9. Pattern Selection Guide](#9-pattern-selection-guide)
- [10. Common Mistakes & Over-Engineering](#10-common-mistakes--over-engineering)
- [11. Final Cheat Sheet](#11-final-cheat-sheet-one-screen-revision)

---

## 1. Introduction to Design Patterns

### What design patterns are

**Design patterns are reusable solutions to recurring design problems** in software. They are not code you copy-paste; they are **shapes of solutions** you recognize and adapt.

A useful way to remember:
- **Algorithm**: solves a computational problem (sorting, shortest path).
- **Design Pattern**: solves a *design* problem (how objects collaborate, how to add behavior safely, how to decouple).

### Why they exist (the “pain” they fix)

Most code starts simple. Then requirements grow:
- “Add one more payment provider”
- “Add caching”
- “Add retries”
- “Add metrics”
- “Support multiple DBs”
- “Support multiple UI clients”

Without good design, changes cause:
- Touching too many files
- Fragile conditionals (`if/else` trees)
- Tight coupling
- Untestable code
- “One change breaks everything”

Patterns exist to make **change cheaper and safer**.

### Problems they solve

Patterns typically solve one or more of:
- **Creation complexity** (how to build objects safely and flexibly)
- **Structural composition** (how to assemble objects without coupling)
- **Behavior variation** (how to switch logic without `if/else` explosions)
- **Integration** (how to connect incompatible APIs)
- **Control and indirection** (caching, lazy loading, security, remote calls)

### Design patterns vs algorithms vs frameworks

| Concept | What it is | Example | Key difference |
|---|---|---|---|
| Algorithm | Step-by-step computation | Binary search | Focused on performance/correctness of computation |
| Design Pattern | Reusable design solution | Strategy, Decorator | Focused on object collaboration and extensibility |
| Framework | Inversion-of-control platform | Spring, Quarkus | Framework calls your code; patterns guide your code design within it |

### When **not** to use design patterns

Patterns are tools, not trophies. Avoid patterns when:
- The code is small and stable (YAGNI: “You Aren’t Gonna Need It”)
- The pattern adds indirection without real change pressure
- A simple function or class would do
- The team can’t maintain the abstraction

**Rule of thumb:**  
Use patterns when they **reduce total cost of change**, not when they increase cleverness.

---

## 2. Classification of Design Patterns

### The three categories

- **Creational**: How objects are created
- **Structural**: How objects are composed/wired
- **Behavioral**: How objects communicate and share responsibilities

### Summary table (pattern → category → intent)

| Pattern | Category | Primary intent |
|---|---|---|
| Singleton | Creational | One shared instance, globally accessible |
| Factory Method | Creational | Let subclasses decide what to instantiate |
| Abstract Factory | Creational | Create families of related objects |
| Builder | Creational | Construct complex objects step-by-step |
| Prototype | Creational | Clone existing objects efficiently |
| Adapter | Structural | Make incompatible interfaces work together |
| Decorator | Structural | Add behavior without subclassing |
| Facade | Structural | Provide a simpler API over complex subsystems |
| Proxy | Structural | Control access to an object (lazy/caching/security/remote) |
| Composite | Structural | Treat part and whole uniformly (tree structures) |
| Bridge | Structural | Decouple abstraction from implementation |
| Strategy | Behavioral | Swap algorithms at runtime |
| Observer | Behavioral | Publish/subscribe state changes |
| Command | Behavioral | Encapsulate a request as an object |
| State | Behavioral | Change behavior when internal state changes |
| Template Method | Behavioral | Define algorithm skeleton; subclasses fill steps |
| Chain of Responsibility | Behavioral | Pass request along handlers until one handles |
| Iterator | Behavioral | Traverse collections without exposing internals |
| Mediator | Behavioral | Centralize complex communications between objects |

---

## 3. Creational Design Patterns (Java)

### Singleton

#### Problem statement
You want **exactly one** instance of something:
- Configuration holder
- Connection pool manager
- Feature flags registry
- In-memory cache

#### Simple explanation
A Singleton is a controlled global instance. The trick is: **global access without uncontrolled global state**.

#### Real-world analogy
A building has **one main electrical panel**. Everyone uses it, but you don’t want 10 different panels created randomly.

#### Bad design example (multiple instances, inconsistent state)
```java
class AppConfig {
    // Loaded repeatedly, possibly with different sources
    public AppConfig() {
        System.out.println("Loading config from disk...");
    }
}

class Main {
    public static void main(String[] args) {
        AppConfig c1 = new AppConfig();
        AppConfig c2 = new AppConfig(); // loaded twice
    }
}
```

#### Correct solution (Singleton with enum — simplest and safest)
```java
public enum AppConfig {
    INSTANCE;

    private final String environment = "prod";

    public String env() {
        return environment;
    }
}

// Usage:
// AppConfig.INSTANCE.env()
```

#### Step-by-step explanation
1. `enum` in Java guarantees one instance per enum constant.
2. Serialization safety is built in.
3. Reflection attacks are effectively prevented compared to many naive implementations.

#### When to use
- The object truly must be unique
- The object is stateless or carefully managed
- You can justify global access

#### When to avoid
- When it becomes a hidden dependency (hurts testability)
- When multiple configurations per test/request are needed
- When it becomes “global mutable state”

#### Time & space considerations
- Access is O(1)
- Memory is constant (one instance)

#### Interview tips
- Prefer `enum` Singleton in Java.
- Mention testability concerns and alternatives (DI container-managed singletons in Spring).

---

### Factory Method

#### Problem statement
You need to create objects, but:
- The exact type depends on runtime inputs
- You want to avoid sprinkling `new` and `if/else` across the codebase

#### Simple explanation
Factory Method delegates creation to subclasses or dedicated creators so that **creation logic is centralized and extensible**.

#### Real-world analogy
A restaurant doesn’t make you build your own sandwich. You place an order; the kitchen creates the right sandwich.

#### Bad design example (if/else explosion)
```java
interface Notification {
    void send(String to, String msg);
}

class EmailNotification implements Notification {
    public void send(String to, String msg) { System.out.println("EMAIL to " + to); }
}

class SmsNotification implements Notification {
    public void send(String to, String msg) { System.out.println("SMS to " + to); }
}

class NotificationService {
    public Notification create(String type) {
        if ("email".equalsIgnoreCase(type)) return new EmailNotification();
        if ("sms".equalsIgnoreCase(type)) return new SmsNotification();
        throw new IllegalArgumentException("Unknown type: " + type);
    }
}
```

#### Correct pattern-based solution (Factory Method via Creator hierarchy)
```java
interface Notification {
    void send(String to, String msg);
}

class EmailNotification implements Notification {
    public void send(String to, String msg) { System.out.println("EMAIL to " + to + ": " + msg); }
}
class SmsNotification implements Notification {
    public void send(String to, String msg) { System.out.println("SMS to " + to + ": " + msg); }
}

abstract class NotificationCreator {
    // Factory Method
    public abstract Notification create();

    // Business method uses product
    public void notify(String to, String msg) {
        Notification n = create();
        n.send(to, msg);
    }
}

class EmailNotificationCreator extends NotificationCreator {
    public Notification create() { return new EmailNotification(); }
}

class SmsNotificationCreator extends NotificationCreator {
    public Notification create() { return new SmsNotification(); }
}
```

#### Step-by-step explanation
1. `NotificationCreator.notify(...)` contains stable business flow.
2. `create()` varies and is delegated to subclasses.
3. Adding a new channel = create a new `NotificationCreator` subclass.

#### When to use
- When creation varies by context
- When you want to keep business logic stable and open for extension

#### When to avoid
- When you only ever need one concrete type
- When it introduces too many tiny classes without real variation

#### Time & space considerations
- Creation cost depends on product; factory overhead is negligible.

#### Interview tips
- Explain “separating object creation from object usage”.
- Mention OCP: add new product without changing existing logic.

---

### Abstract Factory

#### Problem statement
You need to create **families of related objects** that must work together:
- UI components (Windows vs Mac)
- Cloud providers (AWS vs GCP)
- Persistence stacks (SQL vs NoSQL)

#### Simple explanation
Abstract Factory creates a *set* of related products without specifying concrete classes.

#### Real-world analogy
When you buy a “bedroom set”, you get a bed + wardrobe + side table that match. You don’t mix random furniture styles.

#### Bad design example (mixing incompatible families)
```java
class AwsS3Client {}
class GcpPubSubClient {}

class App {
    AwsS3Client storage = new AwsS3Client();
    GcpPubSubClient messaging = new GcpPubSubClient(); // mismatched stack
}
```

#### Correct solution (Abstract Factory for cloud stack)
```java
// Products
interface StorageClient { void put(String key, byte[] data); }
interface MessagingClient { void publish(String topic, String msg); }

// AWS products
class AwsS3Storage implements StorageClient {
    public void put(String key, byte[] data) { System.out.println("AWS S3 put " + key); }
}
class AwsSnsMessaging implements MessagingClient {
    public void publish(String topic, String msg) { System.out.println("AWS SNS publish " + topic); }
}

// GCP products
class GcpGcsStorage implements StorageClient {
    public void put(String key, byte[] data) { System.out.println("GCP GCS put " + key); }
}
class GcpPubSubMessaging implements MessagingClient {
    public void publish(String topic, String msg) { System.out.println("GCP PubSub publish " + topic); }
}

// Abstract Factory
interface CloudFactory {
    StorageClient storage();
    MessagingClient messaging();
}

// Concrete factories
class AwsCloudFactory implements CloudFactory {
    public StorageClient storage() { return new AwsS3Storage(); }
    public MessagingClient messaging() { return new AwsSnsMessaging(); }
}
class GcpCloudFactory implements CloudFactory {
    public StorageClient storage() { return new GcpGcsStorage(); }
    public MessagingClient messaging() { return new GcpPubSubMessaging(); }
}

// Client
class CloudApp {
    private final StorageClient storage;
    private final MessagingClient messaging;

    public CloudApp(CloudFactory factory) {
        this.storage = factory.storage();
        this.messaging = factory.messaging();
    }

    public void run() {
        storage.put("k1", new byte[]{1,2});
        messaging.publish("t1", "hello");
    }
}
```

#### Step-by-step explanation
1. `CloudApp` depends only on interfaces.
2. The factory supplies a consistent family (AWS or GCP).
3. Switching provider becomes configuration, not refactor.

#### When to use
- You must ensure compatible families of objects
- You expect multiple “platforms” (cloud providers, UI themes, environments)

#### When to avoid
- Only one family exists and likely won’t change
- The number of factories becomes unmanageable

#### Time & space considerations
- Factory overhead is minimal; dominant cost is the concrete components.

#### Interview tips
- Compare with Factory Method: Abstract Factory returns **multiple related products**.
- Mention “ensures consistency across product families.”

---

### Builder

#### Problem statement
Constructing objects gets messy when:
- Many optional fields exist
- Constructors become unreadable and error-prone
- You want immutability

#### Simple explanation
Builder separates construction from representation: **build step-by-step, then produce the final object**.

#### Real-world analogy
Ordering a custom pizza: choose size, crust, toppings. The pizza is assembled after options are selected.

#### Bad design example (telescoping constructors)
```java
class HttpRequest {
    private final String url;
    private final String method;
    private final int timeoutMs;
    private final boolean followRedirects;

    public HttpRequest(String url) { this(url, "GET"); }
    public HttpRequest(String url, String method) { this(url, method, 1000); }
    public HttpRequest(String url, String method, int timeoutMs) { this(url, method, timeoutMs, true); }

    public HttpRequest(String url, String method, int timeoutMs, boolean followRedirects) {
        this.url = url;
        this.method = method;
        this.timeoutMs = timeoutMs;
        this.followRedirects = followRedirects;
    }
}
```

#### Correct solution (Builder)
```java
class HttpRequest {
    private final String url;
    private final String method;
    private final int timeoutMs;
    private final boolean followRedirects;

    private HttpRequest(Builder b) {
        this.url = b.url;
        this.method = b.method;
        this.timeoutMs = b.timeoutMs;
        this.followRedirects = b.followRedirects;
    }

    public static class Builder {
        private final String url;              // required
        private String method = "GET";         // optional defaults
        private int timeoutMs = 1000;
        private boolean followRedirects = true;

        public Builder(String url) {
            this.url = url;
        }

        public Builder method(String method) {
            this.method = method;
            return this;
        }

        public Builder timeoutMs(int timeoutMs) {
            this.timeoutMs = timeoutMs;
            return this;
        }

        public Builder followRedirects(boolean follow) {
            this.followRedirects = follow;
            return this;
        }

        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }
}

// Usage:
// HttpRequest req = new HttpRequest.Builder("https://example.com")
//     .method("POST")
//     .timeoutMs(2000)
//     .followRedirects(false)
//     .build();
```

#### Step-by-step explanation
1. Required values go into `Builder` constructor.
2. Optional values are fluent setters.
3. `build()` creates an immutable final object.

#### When to use
- Many optional fields
- Want immutability and readable construction
- Want validation at build time

#### When to avoid
- Tiny objects with 1–2 fields
- Overuse where it adds boilerplate without benefit

#### Time & space considerations
- One extra builder object allocated; typically negligible.

#### Interview tips
- Mention `StringBuilder` as a canonical Builder-like API in JDK.
- Explain how it improves readability and reduces constructor ambiguity.

---

### Prototype

#### Problem statement
Creating objects is expensive, or you need many similar objects:
- Deep configuration objects
- Pre-initialized templates
- Cloning cached templates

#### Simple explanation
Prototype creates new objects by cloning an existing “prototype”.

#### Real-world analogy
Photocopying a filled form template instead of writing from scratch.

#### Bad design example (rebuilding heavy objects repeatedly)
```java
class ReportTemplate {
    private final String header;
    private final String footer;

    public ReportTemplate() {
        // Imagine expensive I/O or heavy initialization
        this.header = "HEADER...";
        this.footer = "FOOTER...";
    }
}
```

#### Correct solution (Prototype with `clone()` or copy constructor)
> In modern Java, prefer **copy constructors** or explicit `copy()` methods over `Cloneable` unless you really need cloning semantics.

```java
class ReportTemplate {
    private final String header;
    private final String footer;

    public ReportTemplate(String header, String footer) {
        this.header = header;
        this.footer = footer;
    }

    // Prototype copy
    public ReportTemplate copy() {
        return new ReportTemplate(this.header, this.footer);
    }

    public ReportTemplate withHeader(String newHeader) {
        return new ReportTemplate(newHeader, this.footer);
    }
}

// Usage:
// ReportTemplate base = new ReportTemplate("H1", "F1");
// ReportTemplate copy = base.copy();
// ReportTemplate variant = base.withHeader("H2");
```

#### Step-by-step explanation
1. Create one “prototype” object (base template).
2. Create copies quickly via `copy()` or with slight modifications via `withX`.

#### When to use
- Object creation is expensive
- Need lots of similar objects
- Want to avoid complex initialization logic repeatedly

#### When to avoid
- Object graphs are complex and copying becomes error-prone
- Immutability makes copying unnecessary or trivial

#### Time & space considerations
- Copy cost depends on deep vs shallow copy; be explicit about copy depth.

#### Interview tips
- Discuss deep vs shallow copies.
- Mention why `Cloneable` is often avoided in modern Java design.

---

## 4. Structural Design Patterns (Java)

### Adapter

#### Problem statement
You need to integrate with an API that doesn’t match your expected interface.

#### Simple explanation
Adapter converts one interface into another the client expects.

#### Real-world analogy
A travel plug adapter: your device stays the same; the adapter translates the socket.

#### Bad design example (leaking third-party API everywhere)
```java
class LegacyPaymentSdk {
    public void makePayment(int amountInPaise) {
        System.out.println("Paid " + amountInPaise + " paise");
    }
}

class Checkout {
    private final LegacyPaymentSdk sdk = new LegacyPaymentSdk();

    public void pay(double amountInRupees) {
        sdk.makePayment((int) (amountInRupees * 100)); // conversion logic scattered
    }
}
```

#### Correct solution (Adapter)
```java
// Target interface
interface PaymentGateway {
    void pay(double amountInRupees);
}

// Adaptee
class LegacyPaymentSdk {
    public void makePayment(int amountInPaise) {
        System.out.println("Paid " + amountInPaise + " paise");
    }
}

// Adapter
class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentSdk sdk;

    public LegacyPaymentAdapter(LegacyPaymentSdk sdk) {
        this.sdk = sdk;
    }

    public void pay(double amountInRupees) {
        int paise = (int) Math.round(amountInRupees * 100);
        sdk.makePayment(paise);
    }
}

// Client depends only on PaymentGateway
class Checkout {
    private final PaymentGateway gateway;

    public Checkout(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public void pay(double amount) {
        gateway.pay(amount);
    }
}
```

#### When to use
- Integrating legacy or third-party APIs
- Migrating between APIs gradually

#### When to avoid
- When you control both interfaces (just refactor instead)

#### Interview tips
- Mention adapters in JDK: `InputStreamReader` adapts `InputStream` to `Reader`.

---

### Decorator

#### Problem statement
You want to add responsibilities dynamically without creating many subclasses:
- Logging
- Metrics
- Caching
- Compression/encryption

#### Simple explanation
Decorator wraps an object and adds behavior before/after delegating.

#### Real-world analogy
A gift wrap: the gift remains the gift; wrap adds presentation without changing the gift itself.

#### Bad design example (subclass explosion)
```java
// If you need combinations: Logged + Cached + Retried...
// you end up with many subclasses: LoggedService, CachedService, LoggedCachedService, ...
```

#### Correct solution (Decorator)
```java
interface DataService {
    String fetch(String key);
}

class RealDataService implements DataService {
    public String fetch(String key) {
        return "value-for-" + key;
    }
}

// Decorator base (optional)
abstract class DataServiceDecorator implements DataService {
    protected final DataService delegate;

    protected DataServiceDecorator(DataService delegate) {
        this.delegate = delegate;
    }
}

class LoggingDecorator extends DataServiceDecorator {
    public LoggingDecorator(DataService delegate) { super(delegate); }

    public String fetch(String key) {
        System.out.println("[LOG] fetch " + key);
        String v = delegate.fetch(key);
        System.out.println("[LOG] result " + v);
        return v;
    }
}

class CachingDecorator extends DataServiceDecorator {
    private final java.util.Map<String, String> cache = new java.util.HashMap<>();

    public CachingDecorator(DataService delegate) { super(delegate); }

    public String fetch(String key) {
        return cache.computeIfAbsent(key, delegate::fetch);
    }
}

// Usage:
// DataService svc = new CachingDecorator(new LoggingDecorator(new RealDataService()));
```

#### When to use
- Cross-cutting behaviors (logging/metrics/caching) without changing core service
- Need combinations of behaviors

#### When to avoid
- If decorators become too deep and hard to trace
- If order of decoration becomes confusing (cache before log vs log before cache)

#### Time & space considerations
- Each decorator adds an extra call layer; often negligible.
- Some decorators (cache) add memory usage intentionally.

#### Interview tips
- Mention Java I/O streams: `BufferedInputStream`, `GZIPInputStream` are classic decorators.

---

### Facade

#### Problem statement
A subsystem is complex; you want a simplified entry point:
- Multiple classes to start an operation
- Difficult orchestration
- Too many dependencies for clients

#### Simple explanation
Facade provides a simple API over a complex subsystem.

#### Real-world analogy
A hotel concierge: you ask once, they coordinate taxi + reservation + tickets.

#### Bad design example (client orchestrates everything)
```java
class OrderController {
    // talks directly to Payment, Inventory, Shipping, Email, ...
    // becomes a fragile orchestration layer
}
```

#### Correct solution (Facade)
```java
class PaymentService {
    boolean charge(String userId, double amount) { return true; }
}
class InventoryService {
    boolean reserve(String sku, int qty) { return true; }
}
class ShippingService {
    void ship(String userId, String sku) { System.out.println("Shipping " + sku); }
}

class OrderFacade {
    private final PaymentService payment = new PaymentService();
    private final InventoryService inventory = new InventoryService();
    private final ShippingService shipping = new ShippingService();

    public void placeOrder(String userId, String sku, int qty, double amount) {
        if (!inventory.reserve(sku, qty)) throw new RuntimeException("Out of stock");
        if (!payment.charge(userId, amount)) throw new RuntimeException("Payment failed");
        shipping.ship(userId, sku);
    }
}

// Client uses one entry point:
class OrderController {
    private final OrderFacade facade = new OrderFacade();

    public void checkout() {
        facade.placeOrder("u1", "sku-1", 1, 499.0);
    }
}
```

#### When to use
- To simplify usage of a subsystem
- To reduce coupling of clients with internal modules

#### When to avoid
- If it becomes a “god object” doing too much business logic (SRP risk)

#### Interview tips
- Mention Spring: `JdbcTemplate` is a facade over JDBC plumbing.

---

### Proxy

#### Problem statement
You need to control access to an object:
- Lazy initialization
- Caching
- Security checks
- Remote calls

#### Simple explanation
Proxy looks like the real object but adds access control or extra behavior.

#### Real-world analogy
A security guard at a building entrance: you don’t talk to every employee; you pass through a gatekeeper.

#### Bad design example (clients manage caching or lazy loading manually)
```java
class ImageService {
    public byte[] load(String id) {
        // expensive load
        return new byte[10_000_000];
    }
}

class Client {
    private final ImageService svc = new ImageService();

    public void display(String id) {
        // might load repeatedly with no cache
        byte[] img = svc.load(id);
    }
}
```

#### Correct solution (Caching proxy)
```java
interface ImageLoader {
    byte[] load(String id);
}

class RealImageLoader implements ImageLoader {
    public byte[] load(String id) {
        System.out.println("Loading image " + id + " from disk/network...");
        return new byte[10_000_000];
    }
}

class CachingImageProxy implements ImageLoader {
    private final ImageLoader real;
    private final java.util.Map<String, byte[]> cache = new java.util.HashMap<>();

    public CachingImageProxy(ImageLoader real) {
        this.real = real;
    }

    public byte[] load(String id) {
        return cache.computeIfAbsent(id, real::load);
    }
}
```

#### When to use
- Lazy load heavy objects
- Cache expensive calls
- Security checks before delegating
- Remote proxies (RPC stubs)

#### When to avoid
- If caching/indirection hides too much and makes debugging difficult

#### Time & space considerations
- Proxy adds small overhead; caching trades memory for speed.

#### Interview tips
- Mention Spring AOP proxies: transactional and security aspects often use proxies.

---

### Composite

#### Problem statement
You want to treat individual objects and groups uniformly:
- File systems (file vs folder)
- UI components (button vs panel of widgets)
- Organization charts

#### Simple explanation
Composite builds a tree where both leaf and container share the same interface.

#### Real-world analogy
A folder can contain files and other folders. You “delete” both using the same operation.

#### Bad design example (different APIs for leaf vs container)
```java
// Client code has to do instanceof checks: if file => do X, if folder => iterate children...
```

#### Correct solution (Composite)
```java
interface Node {
    int size();
}

class FileNode implements Node {
    private final int bytes;
    public FileNode(int bytes) { this.bytes = bytes; }
    public int size() { return bytes; }
}

class DirectoryNode implements Node {
    private final java.util.List<Node> children = new java.util.ArrayList<>();

    public void add(Node n) { children.add(n); }

    public int size() {
        int total = 0;
        for (Node c : children) total += c.size();
        return total;
    }
}

// Usage:
// DirectoryNode root = new DirectoryNode();
// root.add(new FileNode(10));
// DirectoryNode sub = new DirectoryNode();
// sub.add(new FileNode(5));
// root.add(sub);
// root.size() => 15
```

#### When to use
- Hierarchical structures
- Need uniform operations on leaf and composite

#### When to avoid
- If hierarchy is not natural or operations differ too much

#### Interview tips
- Emphasize “uniform treatment” and avoiding type checks.

---

### Bridge

#### Problem statement
You have two dimensions of variation and don’t want a subclass explosion:
- Shapes (circle, square) × renderers (raster, vector)
- Notifications (email, SMS) × delivery providers (Twilio, AWS SNS)

#### Simple explanation
Bridge splits an abstraction from its implementation so both can vary independently.

#### Real-world analogy
A TV remote (abstraction) works with many brands of TVs (implementation). You don’t create a new remote class for every brand × feature.

#### Bad design example (subclass explosion)
```java
// RasterCircle, RasterSquare, VectorCircle, VectorSquare, ...
```

#### Correct solution (Bridge)
```java
// Implementation hierarchy
interface Renderer {
    void drawCircle(int radius);
}

class RasterRenderer implements Renderer {
    public void drawCircle(int radius) { System.out.println("Raster circle r=" + radius); }
}

class VectorRenderer implements Renderer {
    public void drawCircle(int radius) { System.out.println("Vector circle r=" + radius); }
}

// Abstraction hierarchy
abstract class Shape {
    protected final Renderer renderer;
    protected Shape(Renderer renderer) { this.renderer = renderer; }
    public abstract void draw();
}

class Circle extends Shape {
    private final int radius;
    public Circle(Renderer renderer, int radius) {
        super(renderer);
        this.radius = radius;
    }
    public void draw() {
        renderer.drawCircle(radius);
    }
}
```

#### When to use
- Two (or more) independent axes of change
- Want to avoid class explosion from combinations

#### When to avoid
- If only one axis exists (don’t over-abstract)

#### Interview tips
- Compare with Adapter: Adapter makes incompatible APIs work; Bridge is designed upfront to separate variation axes.

---

## 5. Behavioral Design Patterns (Java)

### Strategy

#### Problem statement
You have multiple ways to do something (algorithms) and want to switch without `if/else` growth.

#### Simple explanation
Strategy encapsulates an algorithm behind an interface and allows switching at runtime.

#### Real-world analogy
Route planning: fastest route vs shortest route vs avoiding tolls.

#### Bad design example
```java
class ShippingCostCalculator {
    double cost(String mode, double weight) {
        if ("air".equals(mode)) return weight * 10;
        if ("ground".equals(mode)) return weight * 5;
        if ("sea".equals(mode)) return weight * 2;
        throw new IllegalArgumentException();
    }
}
```

#### Correct solution (Strategy)
```java
interface ShippingStrategy {
    double cost(double weight);
}

class AirShipping implements ShippingStrategy {
    public double cost(double weight) { return weight * 10; }
}
class GroundShipping implements ShippingStrategy {
    public double cost(double weight) { return weight * 5; }
}
class SeaShipping implements ShippingStrategy {
    public double cost(double weight) { return weight * 2; }
}

class ShippingService {
    private ShippingStrategy strategy;

    public ShippingService(ShippingStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(ShippingStrategy strategy) {
        this.strategy = strategy;
    }

    public double calculate(double weight) {
        return strategy.cost(weight);
    }
}
```

#### When to use
- Multiple interchangeable algorithms
- Avoiding complex conditionals
- A/B testing and feature flags

#### When to avoid
- Only one algorithm exists and is stable

#### Interview tips
- Strategy is a direct friend of OCP and DIP.

---

### Observer

#### Problem statement
Many components need to react when something changes:
- Price updates
- Order status changes
- Cache invalidation
- UI updates

#### Simple explanation
Observer defines a publish/subscribe relationship: subject notifies observers on changes.

#### Real-world analogy
You subscribe to a YouTube channel; when a new video arrives, subscribers are notified.

#### Bad design example (hard-coded callbacks)
```java
// OrderService directly calls EmailService, SmsService, AnalyticsService...
// Tight coupling and constant changes when adding a new listener.
```

#### Correct solution (Observer)
```java
interface OrderListener {
    void onOrderPlaced(String orderId);
}

class EmailListener implements OrderListener {
    public void onOrderPlaced(String orderId) {
        System.out.println("Email sent for " + orderId);
    }
}

class AnalyticsListener implements OrderListener {
    public void onOrderPlaced(String orderId) {
        System.out.println("Analytics tracked " + orderId);
    }
}

class OrderService {
    private final java.util.List<OrderListener> listeners = new java.util.ArrayList<>();

    public void addListener(OrderListener l) { listeners.add(l); }

    public void placeOrder(String orderId) {
        System.out.println("Order placed: " + orderId);
        for (OrderListener l : listeners) l.onOrderPlaced(orderId);
    }
}
```

#### When to use
- Multiple independent reactions to events
- Event-driven architecture

#### When to avoid
- If update order matters a lot and becomes implicit/buggy
- If it causes hidden side effects that are hard to trace

#### Interview tips
- Mention Java built-ins: `PropertyChangeListener` (JavaBeans), reactive streams.

---

### Command

#### Problem statement
You want to represent actions as objects:
- Undo/redo
- Queue and execute later
- Audit actions
- Macro commands

#### Simple explanation
Command wraps a request (what to do) into an object (command).

#### Real-world analogy
A restaurant order ticket: it represents an action to be executed by the kitchen.

#### Bad design example (hard-coded actions)
```java
// Button click does if/else: if save => doSave, if delete => doDelete...
```

#### Correct solution (Command)
```java
interface Command {
    void execute();
}

class SaveCommand implements Command {
    public void execute() { System.out.println("Saved"); }
}
class DeleteCommand implements Command {
    public void execute() { System.out.println("Deleted"); }
}

class Button {
    private final Command command;
    public Button(Command command) { this.command = command; }
    public void click() { command.execute(); }
}

// Usage:
// new Button(new SaveCommand()).click();
```

#### When to use
- Undo/redo
- Task scheduling
- Decouple UI/actions from business operations

#### When to avoid
- Simple direct calls with no need for action objects

#### Interview tips
- Relate to `Runnable` in Java: it’s effectively a command (an executable action).

---

### State

#### Problem statement
Behavior changes based on internal state:
- Order lifecycle (NEW → PAID → SHIPPED → DELIVERED)
- Circuit breaker (CLOSED → OPEN → HALF_OPEN)
- Authentication flow

#### Simple explanation
State moves state-specific behavior into separate classes, avoiding giant conditional blocks.

#### Real-world analogy
A traffic light behaves differently based on its current color.

#### Bad design example (stateful if/else everywhere)
```java
class Order {
    String status; // "NEW", "PAID", ...

    void next() {
        if ("NEW".equals(status)) status = "PAID";
        else if ("PAID".equals(status)) status = "SHIPPED";
        else if ("SHIPPED".equals(status)) status = "DELIVERED";
    }
}
```

#### Correct solution (State)
```java
interface OrderState {
    void next(OrderContext ctx);
    String name();
}

class OrderContext {
    private OrderState state = new NewState();

    public void next() { state.next(this); }
    void setState(OrderState state) { this.state = state; }
    public String status() { return state.name(); }
}

class NewState implements OrderState {
    public void next(OrderContext ctx) { ctx.setState(new PaidState()); }
    public String name() { return "NEW"; }
}
class PaidState implements OrderState {
    public void next(OrderContext ctx) { ctx.setState(new ShippedState()); }
    public String name() { return "PAID"; }
}
class ShippedState implements OrderState {
    public void next(OrderContext ctx) { ctx.setState(new DeliveredState()); }
    public String name() { return "SHIPPED"; }
}
class DeliveredState implements OrderState {
    public void next(OrderContext ctx) { /* terminal */ }
    public String name() { return "DELIVERED"; }
}
```

#### When to use
- Many state-specific behaviors
- Want explicit modeling of transitions

#### When to avoid
- Only 2 simple states and transitions are trivial

#### Interview tips
- Mention how State helps remove conditionals and improves OCP.

---

### Template Method

#### Problem statement
You have an algorithm with fixed structure, but some steps vary:
- Import workflows (CSV/JSON/XML)
- Authentication flows
- ETL pipelines

#### Simple explanation
Template Method defines the skeleton in base class; subclasses override specific steps.

#### Real-world analogy
A checklist: step order is fixed, but details differ (different types of inspections).

#### Bad design example (copy-pasted workflows)
```java
// CsvImporter and JsonImporter have 80% duplicate code with minor differences.
```

#### Correct solution (Template Method)
```java
abstract class DataImporter {
    // Template method: fixed algorithm
    public final void importData(String source) {
        String raw = read(source);
        String parsed = parse(raw);
        validate(parsed);
        save(parsed);
    }

    protected abstract String read(String source);
    protected abstract String parse(String raw);

    protected void validate(String parsed) {
        // default validation
        if (parsed == null || parsed.isEmpty()) throw new IllegalArgumentException("Invalid data");
    }

    protected void save(String parsed) {
        System.out.println("Saved: " + parsed);
    }
}

class CsvImporter extends DataImporter {
    protected String read(String source) { return "a,b,c"; }
    protected String parse(String raw) { return "CSV(" + raw + ")"; }
}

class JsonImporter extends DataImporter {
    protected String read(String source) { return "{\"k\":1}"; }
    protected String parse(String raw) { return "JSON(" + raw + ")"; }
}
```

#### When to use
- Common workflow with small varying steps
- Want consistency and reuse

#### When to avoid
- If variations are huge; Strategy may be better

#### Interview tips
- Know difference: Template Method uses inheritance; Strategy uses composition.

---

### Chain of Responsibility

#### Problem statement
A request may be handled by one of many handlers:
- Validation pipeline
- Middleware filters
- Authentication/authorization checks
- Logging + retry + circuit breaker chain

#### Simple explanation
Pass a request along a chain until someone handles it.

#### Real-world analogy
Customer support: level 1 → level 2 → specialist.

#### Bad design example (monolithic validator)
```java
// One huge validate() method with many ifs and early returns.
```

#### Correct solution (Chain)
```java
class Request {
    final String user;
    Request(String user) { this.user = user; }
}

abstract class Handler {
    private Handler next;

    public Handler setNext(Handler next) {
        this.next = next;
        return next;
    }

    public final void handle(Request r) {
        if (process(r) && next != null) {
            next.handle(r);
        }
    }

    // return true to continue chain
    protected abstract boolean process(Request r);
}

class AuthHandler extends Handler {
    protected boolean process(Request r) {
        System.out.println("Auth OK for " + r.user);
        return true;
    }
}

class RateLimitHandler extends Handler {
    protected boolean process(Request r) {
        System.out.println("Rate limit OK for " + r.user);
        return true;
    }
}

class BusinessHandler extends Handler {
    protected boolean process(Request r) {
        System.out.println("Business processed for " + r.user);
        return true;
    }
}

// Usage:
// Handler chain = new AuthHandler();
// chain.setNext(new RateLimitHandler()).setNext(new BusinessHandler());
// chain.handle(new Request("u1"));
```

#### When to use
- Pipelines / middleware
- You want to add/remove handlers independently

#### When to avoid
- If debugging chain order becomes hard
- If request must always be handled in one place

#### Interview tips
- Mention Servlet Filter chain in Java web apps as a real example.

---

### Iterator

#### Problem statement
You want to traverse a structure without exposing internal representation.

#### Simple explanation
Iterator provides a standard way to traverse elements.

#### Real-world analogy
A TV remote “next channel” button: you don’t need to know channel storage.

#### Bad design example
```java
// Client directly accesses internal arrays/lists and breaks encapsulation.
```

#### Correct solution (use Java’s `Iterator`)
```java
import java.util.Iterator;
import java.util.List;

class Demo {
    public static void main(String[] args) {
        List<String> items = List.of("A", "B", "C");
        Iterator<String> it = items.iterator();

        while (it.hasNext()) {
            System.out.println(it.next());
        }
    }
}
```

#### When to use
- Always: prefer iterator/foreach rather than manual indexing when possible
- When underlying structure may change but traversal shouldn’t

#### Interview tips
- Know `Iterable` vs `Iterator`.
- Mention fail-fast iterators (ConcurrentModificationException).

---

### Mediator

#### Problem statement
Many objects talk to each other directly, creating a “spiderweb” of dependencies:
- UI components coordination
- Chat rooms
- Complex workflows

#### Simple explanation
Mediator centralizes communication so colleagues don’t depend on each other directly.

#### Real-world analogy
Air traffic control: planes don’t coordinate with each other directly; they coordinate via the tower.

#### Bad design example (many-to-many coupling)
```java
// Button talks to TextBox, TextBox talks to Label, Label talks to Button...
// changes ripple everywhere
```

#### Correct solution (Mediator)
```java
interface Mediator {
    void notify(Component sender, String event);
}

abstract class Component {
    protected final Mediator mediator;
    protected Component(Mediator mediator) { this.mediator = mediator; }
}

class TextBox extends Component {
    private String text = "";
    public TextBox(Mediator m) { super(m); }
    public void setText(String t) {
        this.text = t;
        mediator.notify(this, "textChanged");
    }
    public String getText() { return text; }
}

class Button extends Component {
    private boolean enabled = false;
    public Button(Mediator m) { super(m); }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public boolean isEnabled() { return enabled; }
}

class FormMediator implements Mediator {
    private TextBox textBox;
    private Button submit;

    public void setTextBox(TextBox t) { this.textBox = t; }
    public void setSubmit(Button b) { this.submit = b; }

    public void notify(Component sender, String event) {
        if (sender == textBox && "textChanged".equals(event)) {
            submit.setEnabled(!textBox.getText().isBlank());
        }
    }
}
```

#### When to use
- Complex object interaction rules
- Need to reduce coupling between components

#### When to avoid
- If mediator becomes a “god object” with too much logic (SRP risk)

#### Interview tips
- Explain “reduces many-to-many dependencies to many-to-one”.

---

## 6. Design Patterns vs SOLID Principles

### How patterns support SOLID

- **SRP**: Facade can simplify clients; Decorators keep core classes focused; Command separates “what” from “when/how executed”.
- **OCP**: Strategy, Factory Method, Decorator, State are common OCP enablers (add new implementations without modifying existing flow).
- **LSP**: Patterns using polymorphism (Strategy/State) rely on substitutability; broken inheritance breaks patterns.
- **ISP**: Many patterns prefer small interfaces: Strategy, Command, Observer.
- **DIP**: Most patterns become clean only when clients depend on abstractions (interfaces), not concrete classes.

### Patterns that can violate SOLID when misused

| Pattern | Typical misuse | SOLID principle at risk | Example failure mode |
|---|---|---|---|
| Singleton | global mutable state | SRP, DIP | hidden dependencies, hard-to-test code |
| Factory (overused) | factories for everything | KISS / SRP | too many layers for no gain |
| Facade | “god facade” that does business logic | SRP | monolithic coordinator |
| Mediator | “god mediator” controlling everything | SRP | central brain becomes unmaintainable |
| Inheritance-heavy Template Method | deep inheritance trees | LSP | fragile base class problems |

### Practical guideline
Patterns are most valuable when they:
- **Reduce coupling**
- **Localize change**
- **Improve test seams**
- **Avoid conditionals and duplication**

---

## 7. Patterns in Real Java Frameworks

### 7.1 Spring Boot production applications (microservices)

Common pattern usage:
- **Singleton**: Spring beans are singleton by default (container-managed).
- **Factory Method / Abstract Factory**: Auto-configuration creates beans based on environment; `FactoryBean` is a formal factory hook.
- **Proxy**: Spring AOP uses proxies for `@Transactional`, security, caching.
- **Facade**: `JdbcTemplate`, `RestTemplate` simplify complex APIs.
- **Decorator**: Filters/interceptors often wrap requests/responses; `ClientHttpRequestInterceptor`.
- **Strategy**: `AuthenticationProvider`, `HttpMessageConverter`, `HandlerMethodArgumentResolver`.
- **Chain of Responsibility**: Servlet filters, Spring Security filter chain.
- **Observer**: `ApplicationEventPublisher` and listeners.

**Practical tip:** In Spring, many patterns are implemented by the framework—your job is to plug in the right interfaces cleanly.

### 7.2 JDK patterns (Collections, Executors, Streams)

- **Iterator**: `Iterable` / `Iterator` define traversal.
- **Decorator**: Java I/O streams (`BufferedInputStream`, `DataInputStream`, `GZIPInputStream`).
- **Adapter**: `InputStreamReader` adapts byte streams to character streams.
- **Factory**:
  - `Executors.newFixedThreadPool(...)` (factory methods)
  - `Collections.unmodifiableList(...)` (wraps list—also proxy/decorator-like)
- **Strategy**: `Comparator` passed into sorting methods.
- **Template Method**: Many abstract classes define skeleton behavior (e.g., `AbstractList`).

### 7.3 Logging frameworks (SLF4J, Logback, Log4j2)

- **Facade**: SLF4J is a logging facade (API) over implementations.
- **Strategy**: appenders/layouts/encoders chosen by configuration.
- **Chain of Responsibility**: log filtering (turbo filters, marker filters).
- **Decorator**: async appenders or wrappers adding buffering and dispatch.

---

## 8. Most Asked Design Pattern Interview Questions

### Conceptual

1. **What problem do design patterns solve?**  
   They reduce the cost of change by providing proven structures for collaboration, creation, and extension.

2. **Pattern vs framework?**  
   Pattern is a design concept; framework is a runtime platform that calls your code (IoC).

3. **Factory Method vs Abstract Factory?**  
   Factory Method creates one product type via inheritance/creator; Abstract Factory creates families of related products.

4. **Strategy vs Template Method?**  
   Strategy uses composition to swap algorithms; Template Method uses inheritance and fixed skeleton.

5. **Decorator vs Proxy?**  
   Decorator adds responsibilities; Proxy controls access (lazy/caching/security/remote). Both wrap, but intent differs.

### Code-based / identification

6. **You see a lot of `if (type == ...) new X()` — what pattern helps?**  
   Factory Method or Strategy (depending on whether you’re selecting creation or behavior).

7. **You need to add logging + caching + retries without touching core service — what pattern?**  
   Decorator (or Proxy if the main goal is access control/caching).

8. **You want one API entry point over multiple subsystems — what pattern?**  
   Facade.

9. **You need to integrate a legacy API that doesn’t match your interface — what pattern?**  
   Adapter.

10. **You need undo/redo — what pattern?**  
   Command.

### Design scenario questions

11. **Design a payment system supporting Stripe/PayPal, plus new providers later**  
   - Use `PaymentGateway` interface (DIP)
   - Use Factory/DI to select implementation
   - Use Strategy if selection can change per request

12. **Implement order lifecycle with state-specific behavior**  
   Use State pattern (avoid huge conditional logic).

---

## 9. Pattern Selection Guide

### Quick decision table

| Situation | Likely pattern | Why |
|---|---|---|
| Many optional constructor params | Builder | readable, safe construction |
| You want one global instance (carefully) | Singleton | single shared state |
| Create objects without hardcoding concrete types | Factory Method | centralized creation, OCP |
| Create compatible families of objects | Abstract Factory | platform stacks |
| Need to integrate incompatible APIs | Adapter | interface translation |
| Add behaviors like logging/caching | Decorator | stackable responsibilities |
| Simplify a complex subsystem | Facade | reduce coupling |
| Control access (lazy, caching, security, remote) | Proxy | guarded delegation |
| Tree structures (file/folder) | Composite | uniform treatment |
| Two axes of variation | Bridge | avoid subclass explosion |
| Swap algorithms at runtime | Strategy | behavior selection |
| Publish/subscribe | Observer | decoupled eventing |
| Undo/queue actions | Command | action objects |
| State-dependent behavior | State | remove conditionals |
| Fixed workflow with variable steps | Template Method | shared skeleton |
| Pipeline of handlers | Chain of Responsibility | modular processing |
| Reduce many-to-many coupling | Mediator | central coordination |

### Common anti-patterns (what to avoid)

- **Singleton-as-a-service-locator**: Hidden dependencies, hard tests
- **Factory-everywhere**: Abstraction tax without benefits
- **Overuse inheritance**: Brittle designs; prefer composition
- **Pattern obsession**: Forcing patterns into simple code

---

## 10. Common Mistakes & Over-Engineering

### Misusing Singleton
**Mistake:** using singleton for convenience (“I need it everywhere”).  
**Result:** global state + hidden dependencies.

**Better:**
- In Spring: inject a singleton bean
- In plain Java: pass dependency explicitly or via constructor

### Over-abstracting with Factory
**Mistake:** “every interface needs a factory”.  
**Result:** too many layers; slower development; harder debugging.

**Better:** use factories when creation logic is truly complex or variant.

### Inheritance abuse
**Mistake:** subclassing to reuse code.  
**Result:** LSP issues, fragile base classes.

**Better:** composition + delegation; use Decorator/Strategy.

### Pattern obsession
**Mistake:** rewriting simple code into “enterprise architecture” prematurely.  
**Result:** complexity without payoff.

**Better:** refactor *toward* patterns when pressure appears.

---

## 11. Final Cheat Sheet (One-Screen Revision)

| Pattern | Category | Purpose | Key benefit | Memory trick |
|---|---|---|---|---|
| Singleton | Creational | One instance | shared consistent state | “One ruler for the whole kingdom” |
| Factory Method | Creational | Delegate creation | add products without touching clients | “Order from menu; kitchen decides” |
| Abstract Factory | Creational | Create families | consistent platform stack | “Buy the whole furniture set” |
| Builder | Creational | Step-by-step build | readable, immutable objects | “Pizza order: toppings then bake” |
| Prototype | Creational | Clone templates | fast creation from baseline | “Photocopy a filled form” |
| Adapter | Structural | Translate interface | integrate legacy/3rd-party | “Travel plug converter” |
| Decorator | Structural | Add behavior by wrapping | combine features without subclass explosion | “Gift wrap adds value” |
| Facade | Structural | Simplify subsystem | fewer dependencies for clients | “Concierge handles complexity” |
| Proxy | Structural | Control access | lazy/caching/security/remote | “Security guard at the door” |
| Composite | Structural | Tree uniform ops | treat part/whole same | “Folder contains files/folders” |
| Bridge | Structural | Split variation axes | avoid subclass explosion | “Remote and TV vary independently” |
| Strategy | Behavioral | Swap algorithms | remove if/else complexity | “Choose route mode” |
| Observer | Behavioral | Publish/subscribe | decouple event reactions | “Subscribers get notified” |
| Command | Behavioral | Encapsulate action | undo/queue/log actions | “Order ticket to kitchen” |
| State | Behavioral | State-based behavior | remove state conditionals | “Traffic light behavior changes” |
| Template Method | Behavioral | Skeleton algorithm | reuse workflow, vary steps | “Recipe with customizable step” |
| Chain of Responsibility | Behavioral | Pipeline handlers | flexible request processing | “Support escalations” |
| Iterator | Behavioral | Standard traversal | hide internal representation | “Next channel button” |
| Mediator | Behavioral | Centralize communication | reduce many-to-many coupling | “Air traffic control tower” |

---

### Final guidance (practical)
Design patterns are best seen as **names for proven structures**. In interviews and real projects, the strongest signal is not “knowing pattern definitions” but:
- Recognizing the *problem* first
- Choosing the *simplest* structure that reduces future change cost
- Implementing it in Java with clean interfaces and test seams

This document is suitable for direct use in a GitHub repository as `DESIGN_PATTERNS.md` or as a technical reference guide.