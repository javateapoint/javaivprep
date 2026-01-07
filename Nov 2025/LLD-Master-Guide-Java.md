# Low-Level Design (LLD) Master Guide: Java & Microservices Edition

> **Role:** Principal Java Architect & Interview Expert  
> **Goal:** Bridge the gap between theory and production-grade systems. This guide teaches you how to *think* like an architect during LLD interviews.

---

## 1. Introduction to Low-Level Design (LLD)

### What is LLD?
**High-Level Design (HLD)** focuses on the "macro" view: architecture, databases, load balancers, and how services talk to each other.
**Low-Level Design (LLD)** focuses on the "micro" view: class diagrams, object-oriented modeling, database schemas, APIs, and algorithms within a single component or service.

It answers: *“How do I implement this specific feature using clean, maintainable code?”*

### Why LLD Matters
*   **Production:** Bad LLD leads to "spaghetti code," rigid systems that break when features are added, and concurrency bugs.
*   **Java Backend Roles:** Java is verbose and object-oriented; your ability to use interfaces, abstraction, and patterns effectively is critical.
*   **Microservices:** Each service is a mini-application. LLD determines how clean the internal logic of that service is.

### What Interviewers Evaluate
1.  **Object Modeling:** Can you identify the right entities (Classes/Enums)?
2.  **SOLID Principles:** Do you write decoupled, extensible code?
3.  **Extensibility:** Can I add a new requirement (e.g., a new payment method) without rewriting the core logic?
4.  **Edge Cases:** Do you handle nulls, concurrency, and errors?
5.  **Simplicity:** Do you over-engineer, or do you solve the problem simply first?

---

## 2. LLD Fundamentals (Must-Know Concepts)

### Core Concepts

| Concept | Explanation | Java Example |
| :--- | :--- | :--- |
| **Encapsulation** | Hiding internal state. | `private` fields, public getters/setters. |
| **Polymorphism** | One interface, multiple implementations. | `PaymentStrategy` -> `CreditCard`, `PayPal`. |
| **Composition** | "Has-a" relationship. Prefer over inheritance. | `Order` has a `List<OrderItem>`. |
| **Immutability** | Objects that cannot change after creation. | `final` class fields, no setters. Java `Records`. |
| **Thread Safety** | Safe for concurrent use. | `ConcurrentHashMap`, `AtomicInteger`, `synchronized`. |

### Design Thinking Process
1.  **Clarify Requirements:** Ask questions. "Is this a parking lot for cars only, or trucks too?"
2.  **Identify Entities:** Noun analysis. "Parking Lot," "Ticket," "Gate."
3.  **Define Relationships:** "A Parking Lot *has* Floors. A Floor *has* Spots."
4.  **Assign Responsibilities:** "The `Gate` scans tickets. The `PaymentService` calculates fees."
5.  **Refine & Patternize:** "I need different parking fees for weekends. Use Strategy Pattern."

---

## 3. UML for LLD (Interview-Friendly)

**Tip:** In interviews, don't waste time drawing perfect boxes. Use simple text or rough sketches.

### Text-Based UML Example (Parking Lot)

```text
ParkingLot
  - id: String
  - floors: List<ParkingFloor>
  + addFloor(floor)

ParkingFloor
  - floorNumber: int
  - spots: Map<SpotType, List<ParkingSpot>>
  + parkVehicle(vehicle): Ticket

ParkingSpot (Abstract)
  - id: String
  - isFree: boolean
  + assignVehicle(vehicle)
  + removeVehicle()

Ticket
  - id: String
  - entryTime: long
  - spot: ParkingSpot
```

---

## 4. Core Design Principles in LLD

### SOLID Principles (The "Bible" of LLD)

*   **S - Single Responsibility:** A class does ONE thing.
    *   *Bad:* `OrderService` calculates tax, saves to DB, and sends email.
    *   *Good:* `OrderService` delegates to `TaxCalculator`, `OrderRepository`, `EmailService`.
*   **O - Open/Closed:** Open for extension, closed for modification.
    *   *Example:* Adding a new `NotificationChannel` (SMS) shouldn't change the `NotificationService` code.
*   **L - Liskov Substitution:** Subtypes must replace parents without breaking logic.
    *   *Example:* `Square` extending `Rectangle` often violates this if you change width/height independently.
*   **I - Interface Segregation:** Small, specific interfaces > one huge interface.
    *   *Example:* `Printer` interface vs `Printable` and `Scannable` interfaces.
*   **D - Dependency Inversion:** Depend on abstractions, not concrete classes.
    *   *Example:* Inject `List` (interface), not `ArrayList` (implementation).

---

## 🧱 LLD Case Study 1: Parking Lot System

### Problem Statement
Design a parking lot with multiple floors, spot types (Compact, Large), and entry/exit gates. The system assigns a spot near the entrance and calculates fees upon exit.

### Key Requirements
1.  Floors have different spot capacities.
2.  Vehicles: Car, Truck, Motorcycle.
3.  Ticket issued at entry; payment at exit.
4.  Strategy: "Park at the first available spot."

### Class Design (Java)

```java
// 1. Enums for fixed types
public enum VehicleType { CAR, TRUCK, BIKE }
public enum SpotType { COMPACT, LARGE, MOTORCYCLE }

// 2. Core Entities
public class Vehicle {
    private String licensePlate;
    private VehicleType type;
    // Constructor & Getters
}

public class ParkingSpot {
    private String id;
    private SpotType type;
    private boolean isFree;
    private Vehicle currentVehicle;

    public void park(Vehicle v) {
        this.currentVehicle = v;
        this.isFree = false;
    }

    public void unpark() {
        this.currentVehicle = null;
        this.isFree = true;
    }
}

// 3. Strategy Pattern for Finding Spots
public interface ParkingStrategy {
    ParkingSpot findSpot(List<ParkingFloor> floors, VehicleType type);
}

public class NaturalOrderStrategy implements ParkingStrategy {
    @Override
    public ParkingSpot findSpot(List<ParkingFloor> floors, VehicleType type) {
        // Simple logic: iterate floors, finding first free spot matching type
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.findFreeSpot(type);
            if (spot != null) return spot;
        }
        return null; // Lot full
    }
}

// 4. Entry Gate Controller
public class EntryGate {
    private ParkingStrategy strategy;
    private ParkingLot lot;

    public Ticket processEntry(Vehicle vehicle) {
        ParkingSpot spot = strategy.findSpot(lot.getFloors(), vehicle.getType());
        if (spot == null) throw new RuntimeException("Full");
        
        spot.park(vehicle);
        return new Ticket(vehicle, spot, System.currentTimeMillis());
    }
}
```

---

## 🧱 LLD Case Study 2: Rate Limiter (Production Favorite)

### Problem Statement
Design a component that limits the number of requests a user can send within a time window (e.g., 10 req / sec).

### Approaches
1.  **Fixed Window:** Simple but has "burst" edge cases at window boundaries.
2.  **Sliding Window Log:** Accurate but memory intensive.
3.  **Token Bucket:** Best balance. Standard industry algorithm.

### Java Implementation (Token Bucket - Thread Safe)

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

public class TokenBucketRateLimiter {
    private final int capacity;
    private final int refillRate; // tokens per second
    private final AtomicInteger tokens;
    private final AtomicLong lastRefillTimestamp;

    public TokenBucketRateLimiter(int capacity, int refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = new AtomicInteger(capacity);
        this.lastRefillTimestamp = new AtomicLong(System.currentTimeMillis());
    }

    public synchronized boolean allowRequest() {
        refill();
        if (tokens.get() > 0) {
            tokens.decrementAndGet();
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long last = lastRefillTimestamp.get();
        long elapsedSeconds = (now - last) / 1000;

        if (elapsedSeconds > 0) {
            int newTokens = (int) (elapsedSeconds * refillRate);
            int current = tokens.get();
            // Cap at capacity
            int updated = Math.min(capacity, current + newTokens);
            tokens.set(updated);
            lastRefillTimestamp.set(now);
        }
    }
}
```

**Interview Discussion Points:**
*   **Thread Safety:** Use `synchronized` or atomic variables. `synchronized` is simpler for interviews; Atomics are faster but harder to implement complex logic correctly.
*   **Distributed Rate Limiting:** This code works for *one* server. For distributed systems, use **Redis** (with Lua scripts) to store tokens shared across instances.

---

## 🧱 LLD Case Study 3: Notification System

### Design Goals
*   Support multiple channels (Email, SMS, Push).
*   Extensible (adding Slack/WhatsApp later).
*   Resilient (retry if failed).

### Pattern: Factory + Strategy

```java
// 1. Interface (Strategy)
public interface NotificationChannel {
    void send(String to, String message);
}

// 2. Concrete Strategies
public class EmailChannel implements NotificationChannel {
    public void send(String to, String message) {
        System.out.println("Emailing " + to + ": " + message);
    }
}

public class SmsChannel implements NotificationChannel {
    public void send(String to, String message) {
        System.out.println("SMS to " + to + ": " + message);
    }
}

// 3. Factory to create channels
public class ChannelFactory {
    public static NotificationChannel getChannel(String type) {
        return switch (type.toLowerCase()) {
            case "email" -> new EmailChannel();
            case "sms" -> new SmsChannel();
            default -> throw new IllegalArgumentException("Unknown channel");
        };
    }
}

// 4. Service Layer
public class NotificationService {
    public void notifyUser(String type, String userId, String msg) {
        NotificationChannel channel = ChannelFactory.getChannel(type);
        try {
            channel.send(userId, msg);
        } catch (Exception e) {
            // LLD logic: Retry queue or Dead Letter Queue (DLQ)
            System.err.println("Failed to send. Pushing to DLQ...");
        }
    }
}
```

---

## ⚙️ Microservices-Focused LLD

When doing LLD for a Microservice (e.g., `OrderService`), the structure changes slightly:

1.  **Controller Layer:** Accepts HTTP requests, validates DTOs.
2.  **Service Layer:** Business logic. Don't put logic in Controllers.
3.  **Repository Layer:** DB access.
4.  **Domain Models:** Internal logic entities (POJOs).
5.  **DTOs (Data Transfer Objects):** Contracts for the outside world. Never expose your DB entities directly in APIs.

**Example: Idempotency in LLD**
Handling duplicate requests (e.g., user clicks "Pay" twice).
*   *Solution:* Use an `idempotencyKey` in the request header.
*   *Implementation:* Check Redis/DB if key exists. If yes, return previous response. If no, process and save key.

---

## 🧠 Most Asked LLD Interview Problems

1.  **Design a Cache (LRU):**
    *   *Core:* HashMap + Doubly Linked List.
    *   *Key:* O(1) get and put.
2.  **Design an Elevator System:**
    *   *State Pattern:* MovingUp, MovingDown, Idle.
    *   *Scheduling Algorithm:* SCAN or LOOK algorithm (not FCFS).
3.  **Design Tic-Tac-Toe / Chess:**
    *   *Core:* Board class, Piece abstract class, RuleEngine for validating moves.
4.  **Design a File System:**
    *   *Composite Pattern:* Directory contains Files or other Directories.

---

## 🚫 Common LLD Mistakes

*   **God Classes:** Creating one `System` class that manages everything.
*   **Getters/Setters Abuse:** Using them for everything instead of business methods (e.g., `account.setBalance(x)` vs `account.deposit(x)`).
*   **Ignoring Concurrency:** Assuming code runs in a single thread.
*   **Database Driven Design:** Starting with "Table Schema" instead of "Object Behavior." Start with Objects!

---

## 🧾 Final LLD Cheat Sheet (One-Screen Revision)

| Category | Shortcut / Tip |
| :--- | :--- |
| **Patterns** | **Strategy** (algorithms), **Factory** (creation), **Observer** (events), **Singleton** (config). |
| **SOLID** | **S**ingle purpose, **O**pen for extension, **D**epend on interfaces. |
| **Concurrency** | `AtomicInteger`, `ConcurrentHashMap`, `synchronized`, `ReentrantLock`. |
| **Microservices** | Controller -> Service -> Repository. Use DTOs. Handle retries. |
| **Interview** | Clarify reqs -> Define Entities -> Define API -> Code Core Logic -> Handle Edge Cases. |

---

**Document Version:** 1.0  
**Target Audience:** Senior Java Developers & Architects  
**Usage:** Interview Prep, Design References