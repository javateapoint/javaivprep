# SOLID Principles: A Complete Java Deep Dive

## Introduction - Why SOLID Exists

### What Are SOLID Principles?

SOLID is an acronym representing five fundamental principles of object-oriented design introduced by Robert C. Martin (Uncle Bob). These principles guide developers in writing software that is easier to maintain, extend, test, and understand.

SOLID is not a framework or library. It's a mindset—a set of design guidelines that help prevent code from becoming a tangled, unmaintainable mess as systems grow.

### Why SOLID Matters in Real-World Java Applications

Consider a typical enterprise Java application that starts with 10 classes and grows to 1000 classes over 3 years. Without SOLID principles:

- **Changing one feature breaks five others** because everything is tightly coupled
- **Adding new features requires modifying existing code**, introducing bugs
- **Testing becomes impossible** because classes depend on databases, external APIs, and each other
- **New team members take weeks** to understand "spaghetti code" with unclear responsibilities

SOLID principles address these issues directly.

### Problems in Codebases Without SOLID

**Without SRP (Single Responsibility):**
- "God classes" that do everything from validation to database access to sending emails
- Changing email logic requires understanding database code
- One bug fix creates three new bugs in unrelated features

**Without OCP (Open/Closed):**
- Every new feature requires modifying existing, tested code
- Long chains of if-else statements checking object types
- High regression risk with each change

**Without LSP (Liskov Substitution):**
- Subtypes that throw exceptions when calling parent methods
- Runtime errors that should have been caught at design time
- Fragile inheritance hierarchies

**Without ISP (Interface Segregation):**
- Classes implementing 20 methods when they only need 3
- Empty method implementations or methods throwing UnsupportedOperationException
- Unnecessary coupling to unrelated functionality

**Without DIP (Dependency Inversion):**
- Business logic hardcoded to specific database implementations
- Impossible to unit test without hitting real databases
- Changing one low-level component requires changes throughout the system

### How SOLID Improves Your Code

#### Maintainability
Code organized by SOLID principles is easier to modify because changes are localized. Fixing a bug in email sending doesn't touch payment processing code.

#### Readability
Smaller, focused classes with clear names are self-documenting. A class named `InvoiceEmailSender` is immediately understandable.

#### Testability
Loose coupling through interfaces allows easy mocking and stubbing. You can test business logic without databases, networks, or external dependencies.

#### Scalability
New features can be added by creating new classes rather than modifying existing ones. This reduces risk and allows parallel development by multiple team members.

### How Interviewers Evaluate SOLID Knowledge

**Junior Level**: Can you explain what each principle means? Can you recognize violations in simple code examples?

**Mid-Level**: Can you refactor violating code? Can you explain trade-offs? Can you apply SOLID to real scenarios?

**Senior Level**: Do you know when NOT to apply SOLID? Can you architect systems using these principles? Can you explain how frameworks like Spring implement these principles?

**Red Flags for Interviewers**:
- Reciting definitions without understanding practical application
- Applying SOLID dogmatically without considering simplicity
- Unable to recognize violations in code
- Confusing principles with patterns (e.g., DIP with Dependency Injection)

---

## Overview of SOLID - The Big Picture

### The SOLID Acronym Explained

| Letter | Principle | One-Line Meaning |
|--------|-----------|------------------|
| **S** | Single Responsibility Principle | A class should have one, and only one, reason to change |
| **O** | Open/Closed Principle | Open for extension, closed for modification |
| **L** | Liskov Substitution Principle | Derived classes must be substitutable for their base classes |
| **I** | Interface Segregation Principle | Many client-specific interfaces are better than one general-purpose interface |
| **D** | Dependency Inversion Principle | Depend upon abstractions, not concretions |

### How SOLID Principles Work Together

SOLID principles are not isolated rules—they complement and reinforce each other:

**SRP + DIP**: When a class has a single responsibility, it's easier to inject its dependencies through interfaces.

**OCP + DIP**: Depending on abstractions makes it easy to add new implementations without modifying existing code.

**LSP + ISP**: Focused interfaces make it easier to create proper inheritance hierarchies that don't violate substitutability.

**All Together**: A well-designed system has small, focused classes (SRP) that depend on abstractions (DIP), can be extended without modification (OCP), have substitutable hierarchies (LSP), and expose only necessary interfaces (ISP).

---

## S - Single Responsibility Principle (SRP)

### Explanation

**Definition**: "A class should have one, and only one, reason to change."

This principle is often misunderstood. It doesn't mean a class should only have one method or do one thing. It means a class should be responsible to **one actor** or **one business concern**.

Ask yourself: "If I change the database structure, should I also have to modify the email sending logic?" If yes, you've violated SRP.

### Real-World Analogy

Think of a restaurant. The chef cooks, the waiter serves, and the cashier handles payments. Each has a single responsibility. If the chef had to cook, serve, AND handle payments, chaos would ensue when one responsibility changes (e.g., accepting credit cards).

Similarly, in code, separating responsibilities means changing how you send emails doesn't affect how you save data.

### Bad Design Example

```java
/**
 * VIOLATION: This class has multiple reasons to change:
 * 1. Changes to user validation logic
 * 2. Changes to database schema or persistence technology
 * 3. Changes to email service provider or email templates
 */
public class UserManager {
    
    public void createUser(String username, String email, String password) {
        // Responsibility 1: Validation logic
        if (username == null || username.isEmpty()) {
            throw new IllegalArgumentException("Username cannot be empty");
        }
        
        if (password.length() < 8) {
            throw new IllegalArgumentException("Password must be at least 8 characters");
        }
        
        if (!email.contains("@")) {
            throw new IllegalArgumentException("Invalid email format");
        }
        
        // Responsibility 2: Database operations
        String sql = "INSERT INTO users (username, email, password) VALUES (?, ?, ?)";
        // Execute SQL directly
        System.out.println("Executing SQL: " + sql);
        System.out.println("User saved to database");
        
        // Responsibility 3: Email notifications
        String emailBody = "Welcome " + username + "! Thanks for registering.";
        // Send email directly
        System.out.println("Connecting to SMTP server...");
        System.out.println("Sending email to: " + email);
        System.out.println("Email sent successfully");
        
        // Responsibility 4: Logging
        System.out.println("User created: " + username + " at " + System.currentTimeMillis());
    }
}
```

**Why This Is a Problem:**

1. **Tight Coupling**: The class is coupled to database technology, email infrastructure, validation rules, and logging mechanisms
2. **Hard to Test**: You can't test validation without triggering database and email code
3. **Difficult to Maintain**: A change in email format affects a class that should only care about user management
4. **Multiple Reasons to Change**: 
   - Marketing team wants different email templates → Must modify UserManager
   - DBA changes database schema → Must modify UserManager
   - Security team updates password rules → Must modify UserManager
   - Operations team changes logging format → Must modify UserManager

### Good Design Example

```java
// Responsibility 1: Validation
public class UserValidator {
    public void validate(String username, String email, String password) {
        if (username == null || username.isEmpty()) {
            throw new IllegalArgumentException("Username cannot be empty");
        }
        
        if (password.length() < 8) {
            throw new IllegalArgumentException("Password must be at least 8 characters");
        }
        
        if (!email.contains("@")) {
            throw new IllegalArgumentException("Invalid email format");
        }
    }
}

// Responsibility 2: Data persistence
public class UserRepository {
    public void save(User user) {
        String sql = "INSERT INTO users (username, email, password) VALUES (?, ?, ?)";
        System.out.println("Executing SQL: " + sql);
        System.out.println("User saved to database");
    }
}

// Responsibility 3: Email notifications
public class EmailService {
    public void sendWelcomeEmail(String email, String username) {
        String emailBody = "Welcome " + username + "! Thanks for registering.";
        System.out.println("Connecting to SMTP server...");
        System.out.println("Sending email to: " + email);
        System.out.println("Email sent successfully");
    }
}

// Responsibility 4: Logging
public class UserActivityLogger {
    public void logUserCreation(String username) {
        System.out.println("User created: " + username + " at " + System.currentTimeMillis());
    }
}

// Coordinator: Uses specialized classes
public class UserService {
    private final UserValidator validator;
    private final UserRepository repository;
    private final EmailService emailService;
    private final UserActivityLogger logger;
    
    // Constructor injection (Dependency Inversion)
    public UserService(UserValidator validator, 
                       UserRepository repository,
                       EmailService emailService,
                       UserActivityLogger logger) {
        this.validator = validator;
        this.repository = repository;
        this.emailService = emailService;
        this.logger = logger;
    }
    
    public void createUser(String username, String email, String password) {
        // Delegate to specialists
        validator.validate(username, email, password);
        
        User user = new User(username, email, password);
        repository.save(user);
        
        emailService.sendWelcomeEmail(email, username);
        logger.logUserCreation(username);
    }
}

// Simple data class
class User {
    private String username;
    private String email;
    private String password;
    
    public User(String username, String email, String password) {
        this.username = username;
        this.email = email;
        this.password = password;
    }
    
    // Getters omitted for brevity
}
```

**Why This Is Better:**

1. **Single Responsibility**: Each class has one reason to change
2. **Easy to Test**: Test `UserValidator` without databases or email servers
3. **Easy to Maintain**: Change email logic in `EmailService` without touching validation or persistence
4. **Easy to Extend**: Add SMS notifications by creating `SmsService` without modifying existing classes
5. **Clear Naming**: Class names describe their purpose unambiguously

### Key Takeaways

**How to Identify SRP Violations:**

1. **Class name contains "And" or "Manager"**: `UserAndEmailManager`, `DataManagerService`
2. **Class has many dependencies**: If constructor has 5+ parameters, it's doing too much
3. **Multiple reasons to change**: Ask "What would cause this class to change?" If you list more than one business reason, it violates SRP
4. **Hard to name**: If you can't give a class a clear, concise name, it probably has multiple responsibilities
5. **Giant classes**: 500+ line classes are almost always doing too much

**When to Apply SRP:**

- During initial design when defining classes
- When a class becomes hard to understand or test
- When changes to one feature keep breaking unrelated features
- When onboarding new developers who struggle to understand a class

**When NOT to Overdo SRP:**

- Very simple classes or utilities (e.g., a `MathUtils` class with multiple unrelated math functions is acceptable)
- Early prototypes where flexibility matters more than structure
- When splitting creates more complexity than it removes

---

## O - Open/Closed Principle (OCP)

### Explanation

**Definition**: "Software entities (classes, modules, functions) should be open for extension, but closed for modification."

**In Plain English**: You should be able to add new functionality to your system without changing existing code. Instead of modifying a class to add behavior, you extend the system by adding new classes.

**Why It Matters**: Every time you modify existing code, you risk introducing bugs. Code that's been tested and deployed should remain untouched. New features should come from new code.

### Real-World Analogy

Think of a power outlet in your home. The outlet (interface) never changes, but you can plug in different devices (implementations)—a lamp, phone charger, laptop, or fan. You extend functionality without modifying the outlet.

Similarly, your code should have stable interfaces that allow new implementations to be plugged in.

### Bad Design Example

```java
/**
 * VIOLATION: Every time we add a new discount type,
 * we must modify this class.
 */
public class PriceCalculator {
    
    public double calculatePrice(double basePrice, String customerType) {
        double finalPrice = basePrice;
        
        // Must modify this code for every new customer type
        if (customerType.equals("REGULAR")) {
            finalPrice = basePrice;
        } else if (customerType.equals("SILVER")) {
            finalPrice = basePrice * 0.95; // 5% discount
        } else if (customerType.equals("GOLD")) {
            finalPrice = basePrice * 0.90; // 10% discount
        } else if (customerType.equals("PLATINUM")) {
            finalPrice = basePrice * 0.80; // 20% discount
        }
        // What happens when marketing wants a "DIAMOND" tier?
        // We have to modify this tested code!
        
        return finalPrice;
    }
}

// Usage
public class ShoppingCart {
    private PriceCalculator calculator = new PriceCalculator();
    
    public void checkout(double price, String customerType) {
        double finalPrice = calculator.calculatePrice(price, customerType);
        System.out.println("Final price: $" + finalPrice);
    }
}
```

**Why This Is a Problem:**

1. **Modification Required**: Every new customer type requires editing `PriceCalculator`
2. **Testing Overhead**: Each change requires retesting all existing discount types
3. **Merge Conflicts**: Multiple developers adding different discount types will have conflicts
4. **Fragile**: Easy to accidentally break existing discount logic
5. **Violation of OCP**: The class is not closed for modification

### Good Design Example - Using Strategy Pattern

```java
/**
 * GOOD: Define an interface for discount strategies.
 * This interface is CLOSED for modification but OPEN for extension.
 */
public interface DiscountStrategy {
    double applyDiscount(double basePrice);
}

// Extension 1: Regular customers (no discount)
public class RegularCustomerDiscount implements DiscountStrategy {
    @Override
    public double applyDiscount(double basePrice) {
        return basePrice; // No discount
    }
}

// Extension 2: Silver customers
public class SilverCustomerDiscount implements DiscountStrategy {
    @Override
    public double applyDiscount(double basePrice) {
        return basePrice * 0.95; // 5% off
    }
}

// Extension 3: Gold customers
public class GoldCustomerDiscount implements DiscountStrategy {
    @Override
    public double applyDiscount(double basePrice) {
        return basePrice * 0.90; // 10% off
    }
}

// Extension 4: Platinum customers
public class PlatinumCustomerDiscount implements DiscountStrategy {
    @Override
    public double applyDiscount(double basePrice) {
        return basePrice * 0.80; // 20% off
    }
}

// NEW: Diamond customers - added WITHOUT modifying existing code!
public class DiamondCustomerDiscount implements DiscountStrategy {
    @Override
    public double applyDiscount(double basePrice) {
        return basePrice * 0.70; // 30% off
    }
}

/**
 * This class is now CLOSED for modification.
 * It will never change as we add new discount types.
 */
public class PriceCalculator {
    public double calculatePrice(double basePrice, DiscountStrategy strategy) {
        return strategy.applyDiscount(basePrice);
    }
}

// Usage
public class ShoppingCart {
    private PriceCalculator calculator = new PriceCalculator();
    
    public void checkout(double price, DiscountStrategy discountStrategy) {
        double finalPrice = calculator.calculatePrice(price, discountStrategy);
        System.out.println("Final price: $" + finalPrice);
    }
    
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();
        
        // Different customers get different strategies
        cart.checkout(100, new RegularCustomerDiscount());
        cart.checkout(100, new SilverCustomerDiscount());
        cart.checkout(100, new GoldCustomerDiscount());
        cart.checkout(100, new PlatinumCustomerDiscount());
        cart.checkout(100, new DiamondCustomerDiscount()); // New tier!
    }
}
```

**Output:**
```
Final price: $100.0
Final price: $95.0
Final price: $90.0
Final price: $80.0
Final price: $70.0
```

**Why This Is Better:**

1. **Extension Without Modification**: Adding a new discount type just requires creating a new class
2. **Existing Code Untouched**: `PriceCalculator` never changes once written
3. **Easier Testing**: Each discount strategy is tested independently
4. **Parallel Development**: Multiple developers can add different strategies without conflicts
5. **Follows OCP**: The system is open for extension, closed for modification

### Alternative Good Design - Using Polymorphism with Inheritance

```java
// Abstract base class
public abstract class Customer {
    protected String name;
    
    public Customer(String name) {
        this.name = name;
    }
    
    // Template method - can be extended
    public abstract double getDiscount(double basePrice);
}

public class RegularCustomer extends Customer {
    public RegularCustomer(String name) {
        super(name);
    }
    
    @Override
    public double getDiscount(double basePrice) {
        return basePrice;
    }
}

public class VipCustomer extends Customer {
    public VipCustomer(String name) {
        super(name);
    }
    
    @Override
    public double getDiscount(double basePrice) {
        return basePrice * 0.85; // 15% off
    }
}

// New customer type - no modification to existing code
public class CorporateCustomer extends Customer {
    public CorporateCustomer(String name) {
        super(name);
    }
    
    @Override
    public double getDiscount(double basePrice) {
        return basePrice * 0.75; // 25% off
    }
}

// Usage remains clean
public class Billing {
    public void processPayment(Customer customer, double amount) {
        double finalAmount = customer.getDiscount(amount);
        System.out.println("Billing " + customer.name + ": $" + finalAmount);
    }
}
```

### Interview Insight - Why OCP Is Critical

**Interviewer**: "Why is the Open/Closed Principle important in large systems?"

**Strong Answer**: "In enterprise systems with hundreds of developers and millions of lines of code, modifying existing code is risky and expensive. Every change requires:

1. **Regression testing** of all existing functionality
2. **Code review** and approval cycles
3. **Risk of introducing bugs** in working features
4. **Deployment coordination** across teams

By following OCP and using abstractions, we can add features by creating new classes that implement existing interfaces. This means:

- New code doesn't affect existing code
- New features can be developed in parallel
- Testing is isolated to the new functionality
- Deployment risk is minimized

For example, in microservices, when we need a new payment gateway, we implement the `PaymentGateway` interface. The order service never changes—it just receives a new implementation via dependency injection."

---

## L - Liskov Substitution Principle (LSP)

### Explanation

**Definition**: "Objects of a superclass should be replaceable with objects of a subclass without affecting the correctness of the program."

**In Plain English**: If class `B` extends class `A`, then anywhere in your code where you use class `A`, you should be able to substitute it with class `B` without things breaking or behaving unexpectedly.

**The Duck Test**: If it looks like a duck and quacks like a duck, it better behave like a duck in all situations.

### Relationship with Inheritance

LSP is the principle that governs when inheritance is appropriate. Just because you CAN extend a class doesn't mean you SHOULD. The subclass must honor all behaviors of the parent class.

**Violations Signal**:
- Subclass throws exceptions that parent doesn't throw
- Subclass changes expected behavior of parent methods
- Subclass has stronger preconditions (requires more) or weaker postconditions (guarantees less)

### Bad Design Example - The Classic Bird Problem

```java
/**
 * VIOLATION: Not all birds can fly.
 * This hierarchy creates LSP violations.
 */
public class Bird {
    public void fly() {
        System.out.println("I am flying!");
    }
}

public class Sparrow extends Bird {
    // Sparrow is fine - it can fly
}

public class Ostrich extends Bird {
    @Override
    public void fly() {
        // VIOLATION: Ostrich cannot fly!
        // Breaking the contract of the parent class
        throw new UnsupportedOperationException("Ostrich cannot fly!");
    }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        // VIOLATION: Penguin cannot fly either!
        throw new UnsupportedOperationException("Penguin cannot fly!");
    }
}

// Code that expects all birds to fly
public class BirdWatcher {
    public void makeBirdsFly(List<Bird> birds) {
        for (Bird bird : birds) {
            bird.fly(); // CRASH when bird is Ostrich or Penguin!
        }
    }
}

// Usage
public static void main(String[] args) {
    List<Bird> birds = new ArrayList<>();
    birds.add(new Sparrow());
    birds.add(new Ostrich());  // Problem!
    birds.add(new Penguin());  // Problem!
    
    BirdWatcher watcher = new BirdWatcher();
    watcher.makeBirdsFly(birds); // Runtime exceptions!
}
```

**Why This Is a Problem:**

1. **Runtime Errors**: Code expecting all Birds to fly will crash
2. **Violates LSP**: Cannot substitute Ostrich for Bird
3. **Fragile Design**: Clients must check types before calling methods
4. **Poor Abstraction**: `Bird` abstraction is wrong—not all birds fly

### Good Design Example - Proper Hierarchy

```java
/**
 * GOOD: Create a proper hierarchy based on actual capabilities.
 */

// Base interface for all birds
public interface Bird {
    void eat();
    void makeSound();
}

// Separate capability for flying
public interface FlyingBird extends Bird {
    void fly();
}

// Flying birds
public class Sparrow implements FlyingBird {
    @Override
    public void eat() {
        System.out.println("Sparrow is eating seeds");
    }
    
    @Override
    public void makeSound() {
        System.out.println("Chirp chirp!");
    }
    
    @Override
    public void fly() {
        System.out.println("Sparrow is flying");
    }
}

public class Eagle implements FlyingBird {
    @Override
    public void eat() {
        System.out.println("Eagle is hunting");
    }
    
    @Override
    public void makeSound() {
        System.out.println("Screech!");
    }
    
    @Override
    public void fly() {
        System.out.println("Eagle is soaring");
    }
}

// Non-flying birds - don't implement FlyingBird
public class Ostrich implements Bird {
    @Override
    public void eat() {
        System.out.println("Ostrich is eating plants");
    }
    
    @Override
    public void makeSound() {
        System.out.println("Boom boom!");
    }
    
    // No fly() method - correctly represents reality
}

public class Penguin implements Bird {
    @Override
    public void eat() {
        System.out.println("Penguin is eating fish");
    }
    
    @Override
    public void makeSound() {
        System.out.println("Squawk!");
    }
    
    public void swim() {
        System.out.println("Penguin is swimming");
    }
}

// Correct usage
public class BirdSanctuary {
    
    // Works with all birds
    public void feedBirds(List<Bird> birds) {
        for (Bird bird : birds) {
            bird.eat(); // Safe - all birds eat
        }
    }
    
    // Works only with flying birds
    public void trainFlyingBirds(List<FlyingBird> flyingBirds) {
        for (FlyingBird bird : flyingBirds) {
            bird.fly(); // Safe - only flying birds passed here
        }
    }
}

// Usage
public static void main(String[] args) {
    BirdSanctuary sanctuary = new BirdSanctuary();
    
    // All birds can eat
    List<Bird> allBirds = List.of(
        new Sparrow(),
        new Eagle(),
        new Ostrich(),
        new Penguin()
    );
    sanctuary.feedBirds(allBirds); // Works perfectly
    
    // Only flying birds can fly
    List<FlyingBird> flyingBirds = List.of(
        new Sparrow(),
        new Eagle()
        // Cannot add Ostrich or Penguin - compile error!
    );
    sanctuary.trainFlyingBirds(flyingBirds); // Works perfectly
}
```

### Another Example - Rectangle/Square Problem

```java
/**
 * BAD: Classic LSP violation
 */
class Rectangle {
    protected int width;
    protected int height;
    
    public void setWidth(int width) {
        this.width = width;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
    
    public int getArea() {
        return width * height;
    }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width; // VIOLATION: Changing both
    }
    
    @Override
    public void setHeight(int height) {
        this.width = height; // VIOLATION: Changing both
        this.height = height;
    }
}

// This breaks with Square
public void testRectangle(Rectangle rect) {
    rect.setWidth(5);
    rect.setHeight(4);
    
    // Expected: 20
    // With Rectangle: 20 ✓
    // With Square: 16 ✗ (LSP violated!)
    assert rect.getArea() == 20; // Fails for Square!
}
```

**Better Design**:
```java
// Separate interfaces for different shapes
public interface Shape {
    int getArea();
}

public class Rectangle implements Shape {
    private int width;
    private int height;
    
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public int getArea() {
        return width * height;
    }
}

public class Square implements Shape {
    private int side;
    
    public Square(int side) {
        this.side = side;
    }
    
    @Override
    public int getArea() {
        return side * side;
    }
}

// No inheritance relationship - no LSP violation possible
```

### Red Flags - How to Detect LSP Violations

1. **Throwing UnsupportedOperationException** in overridden methods
2. **Empty implementations** of parent methods
3. **Type checking before calling methods**: `if (bird instanceof Ostrich)`
4. **Stronger preconditions**: Subclass requires more than parent
5. **Weaker postconditions**: Subclass guarantees less than parent
6. **Documentation that says "don't use this method"** in subclass

**Interview Red Flags**:
- Candidate says "just catch the exception"
- Candidate doesn't understand why Square extending Rectangle is problematic
- Candidate uses instanceof checks instead of proper polymorphism

---

## I - Interface Segregation Principle (ISP)

### Explanation

**Definition**: "No client should be forced to depend on methods it does not use."

**In Plain English**: Don't create "fat" interfaces with many unrelated methods. Instead, create multiple small, focused interfaces. Clients should only know about methods they actually need.

**Why "Fat Interfaces" Are Dangerous**:
1. **Unnecessary coupling**: Classes implement methods they never use
2. **Hard to maintain**: Changes to unused methods force recompilation
3. **Confusing**: Developers must implement irrelevant methods
4. **Testing overhead**: Must mock or stub unused methods

### Real-World Analogy

A Swiss Army knife has many tools, but most people only need a few. If every time you wanted to cut something, you had to carry scissors, screwdriver, bottle opener, and 10 other tools, it would be cumbersome.

Similarly, an interface should only expose what a specific client needs, not everything possible.

### Bad Design Example - Fat Interface

```java
/**
 * VIOLATION: Fat interface with many unrelated responsibilities.
 * Forces implementers to deal with methods they don't need.
 */
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeCode();
    void reviewCode();
    void deployApplication();
    void manageTeam();
}

/**
 * A Junior Developer shouldn't have to implement management methods!
 */
public class JuniorDeveloper implements Worker {
    @Override
    public void work() {
        System.out.println("Working on assigned tasks");
    }
    
    @Override
    public void eat() {
        System.out.println("Having lunch");
    }
    
    @Override
    public void sleep() {
        System.out.println("Sleeping");
    }
    
    @Override
    public void attendMeeting() {
        System.out.println("Attending daily standup");
    }
    
    @Override
    public void writeCode() {
        System.out.println("Writing code");
    }
    
    @Override
    public void reviewCode() {
        System.out.println("Reviewing peer's code");
    }
    
    @Override
    public void deployApplication() {
        // VIOLATION: Junior developer typically doesn't deploy
        throw new UnsupportedOperationException("Cannot deploy");
    }
    
    @Override
    public void manageTeam() {
        // VIOLATION: Junior developer doesn't manage teams
        throw new UnsupportedOperationException("Cannot manage team");
    }
}

/**
 * A Robot worker shouldn't need eat() or sleep()!
 */
public class RobotWorker implements Worker {
    @Override
    public void work() {
        System.out.println("Processing tasks 24/7");
    }
    
    @Override
    public void eat() {
        // VIOLATION: Robots don't eat
        throw new UnsupportedOperationException("Robots don't eat");
    }
    
    @Override
    public void sleep() {
        // VIOLATION: Robots don't sleep
        throw new UnsupportedOperationException("Robots don't sleep");
    }
    
    @Override
    public void attendMeeting() {
        throw new UnsupportedOperationException("Robots don't attend meetings");
    }
    
    @Override
    public void writeCode() {
        System.out.println("Generating code");
    }
    
    @Override
    public void reviewCode() {
        System.out.println("Analyzing code");
    }
    
    @Override
    public void deployApplication() {
        System.out.println("Auto-deploying");
    }
    
    @Override
    public void manageTeam() {
        throw new UnsupportedOperationException("Robots don't manage teams");
    }
}
```

**Why This Is a Problem:**

1. **Forced Implementation**: Classes implement methods they'll never use
2. **Runtime Exceptions**: `UnsupportedOperationException` scattered everywhere
3. **Confusing API**: Clients don't know which methods are safe to call
4. **Tight Coupling**: All implementers coupled to all method signatures
5. **Hard to Test**: Must handle exceptions for invalid methods

### Good Design Example - Segregated Interfaces

```java
/**
 * GOOD: Break down into small, focused interfaces.
 * Each interface represents a specific capability.
 */

// Core work capability
public interface Workable {
    void work();
}

// Biological needs
public interface Biological {
    void eat();
    void sleep();
}

// Professional activities
public interface Attendee {
    void attendMeeting();
}

// Development capabilities
public interface Coder {
    void writeCode();
    void reviewCode();
}

// Deployment capability
public interface Deployer {
    void deployApplication();
}

// Management capability
public interface Manager {
    void manageTeam();
}

/**
 * Junior Developer implements only what's relevant
 */
public class JuniorDeveloper implements Workable, Biological, Attendee, Coder {
    @Override
    public void work() {
        System.out.println("Working on assigned tasks");
    }
    
    @Override
    public void eat() {
        System.out.println("Having lunch");
    }
    
    @Override
    public void sleep() {
        System.out.println("Sleeping");
    }
    
    @Override
    public void attendMeeting() {
        System.out.println("Attending daily standup");
    }
    
    @Override
    public void writeCode() {
        System.out.println("Writing code");
    }
    
    @Override
    public void reviewCode() {
        System.out.println("Reviewing peer's code");
    }
    
    // No need to implement deploy() or manageTeam()!
}

/**
 * Senior Developer has more capabilities
 */
public class SeniorDeveloper implements Workable, Biological, Attendee, Coder, Deployer, Manager {
    @Override
    public void work() {
        System.out.println("Working on complex architecture");
    }
    
    @Override
    public void eat() {
        System.out.println("Having lunch");
    }
    
    @Override
    public void sleep() {
        System.out.println("Sleeping");
    }
    
    @Override
    public void attendMeeting() {
        System.out.println("Leading technical discussions");
    }
    
    @Override
    public void writeCode() {
        System.out.println("Writing code");
    }
    
    @Override
    public void reviewCode() {
        System.out.println("Reviewing code");
    }
    
    @Override
    public void deployApplication() {
        System.out.println("Deploying to production");
    }
    
    @Override
    public void manageTeam() {
        System.out.println("Mentoring junior developers");
    }
}

/**
 * Robot Worker implements only relevant interfaces
 */
public class RobotWorker implements Workable, Coder, Deployer {
    @Override
    public void work() {
        System.out.println("Processing tasks 24/7");
    }
    
    @Override
    public void writeCode() {
        System.out.println("Generating code");
    }
    
    @Override
    public void reviewCode() {
        System.out.println("Analyzing code");
    }
    
    @Override
    public void deployApplication() {
        System.out.println("Auto-deploying");
    }
    
    // No eat(), sleep(), attendMeeting(), or manageTeam() methods!
}

// Clean usage
public class ProjectManager {
    
    public void scheduleWork(List<Workable> workers) {
        for (Workable worker : workers) {
            worker.work(); // All workers can work
        }
    }
    
    public void organizeMeeting(List<Attendee> attendees) {
        for (Attendee attendee : attendees) {
            attendee.attendMeeting(); // Only meeting attendees
        }
    }
    
    public void performDeployment(Deployer deployer) {
        deployer.deployApplication(); // Only deployers
    }
}
```

### Practical Benefits

**Cleaner Code**:
- No empty methods
- No `UnsupportedOperationException`
- Clear intent through interface names

**Easier Testing**:
- Mock only what you need
- Smaller, focused test doubles
- No irrelevant method stubs

**Better Maintainability**:
- Changes to one interface don't affect unrelated implementers
- Clear separation of concerns
- Easy to understand what each class can do

**Flexibility**:
- Mix and match capabilities
- Add new interfaces without changing existing code
- Compose functionality through multiple interface implementation

### ISP vs SRP

**Question**: What's the difference between ISP and SRP?

**Answer**:
- **SRP** focuses on **class implementation**: A class should have one responsibility
- **ISP** focuses on **interface design**: An interface should represent one capability

**Example**:
```java
// SRP: UserService does one thing (user management)
public class UserService {
    // ... only user-related methods
}

// ISP: Break capabilities into focused interfaces
public interface Readable {
    void read();
}

public interface Writable {
    void write();
}

// Class implements only needed capabilities
public class ReadOnlyFile implements Readable {
    public void read() { /* implementation */ }
    // No write() method forced on us
}
```

---

## D - Dependency Inversion Principle (DIP)

### Explanation

**Definition**: 
1. "High-level modules should not depend on low-level modules. Both should depend on abstractions."
2. "Abstractions should not depend on details. Details should depend on abstractions."

**In Plain English**: Your business logic shouldn't care about the specific implementation details of things like databases, email services, or payment gateways. Everything should depend on interfaces (abstractions), and concrete implementations are "injected" at runtime.

### High-Level vs Low-Level Modules

**High-Level Module**: Contains business logic and use cases. Example: `OrderService`

**Low-Level Module**: Contains infrastructure details. Example: `MySQLDatabase`, `GmailSender`, `StripePaymentGateway`

**The Problem**: If `OrderService` directly creates a `MySQLDatabase` object, it's tightly coupled to MySQL. Switching to PostgreSQL requires changing `OrderService`.

**The Solution**: `OrderService` depends on a `Database` interface. We inject `MySQLDatabase` or `PostgreSQLDatabase` at runtime.

### Bad Design Example - Tight Coupling

```java
/**
 * VIOLATION: High-level module (NotificationService) depends on
 * low-level concrete class (EmailSender).
 */

// Low-level module
public class EmailSender {
    public void sendEmail(String recipient, String message) {
        System.out.println("Sending email to " + recipient);
        System.out.println("Message: " + message);
        // Actual SMTP logic here
    }
}

// High-level module - TIGHTLY COUPLED
public class NotificationService {
    private EmailSender emailSender; // Direct dependency on concrete class
    
    public NotificationService() {
        this.emailSender = new EmailSender(); // Creating dependency inside!
    }
    
    public void notifyUser(String userId, String message) {
        // Business logic
        System.out.println("Preparing notification for user: " + userId);
        
        // Coupled to EmailSender - cannot easily change to SMS or Push
        emailSender.sendEmail(userId + "@example.com", message);
    }
}

/**
 * Problems:
 * 1. Cannot test NotificationService without sending real emails
 * 2. Cannot switch to SMS without modifying NotificationService
 * 3. Cannot use both Email and SMS
 * 4. Hard-coded to one implementation
 */
public class Main {
    public static void main(String[] args) {
        NotificationService service = new NotificationService();
        service.notifyUser("john123", "Your order is confirmed");
        
        // What if marketing wants SMS notifications?
        // Must modify NotificationService!
    }
}
```

**Why This Is a Problem:**

1. **No Testability**: Can't unit test without sending real emails
2. **No Flexibility**: Stuck with email forever
3. **Violates OCP**: Must modify `NotificationService` to add SMS
4. **Tight Coupling**: High-level logic knows about low-level SMTP details

### Good Design Example - Dependency Inversion

```java
/**
 * GOOD: Introduce an abstraction (interface).
 * Both high-level and low-level modules depend on it.
 */

// Abstraction - Neither high-level nor low-level owns this
public interface MessageSender {
    void send(String recipient, String message);
}

// Low-level module 1: Email implementation
public class EmailSender implements MessageSender {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending EMAIL to " + recipient);
        System.out.println("Message: " + message);
        // SMTP logic
    }
}

// Low-level module 2: SMS implementation
public class SmsSender implements MessageSender {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending SMS to " + recipient);
        System.out.println("Message: " + message);
        // SMS API logic
    }
}

// Low-level module 3: Push notification implementation
public class PushNotificationSender implements MessageSender {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending PUSH notification to " + recipient);
        System.out.println("Message: " + message);
        // Push notification logic
    }
}

// High-level module - depends on abstraction
public class NotificationService {
    private final MessageSender messageSender; // Depends on interface
    
    // Dependency Injection via constructor
    public NotificationService(MessageSender messageSender) {
        this.messageSender = messageSender;
    }
    
    public void notifyUser(String userId, String message) {
        // Business logic - unchanged regardless of implementation
        System.out.println("Preparing notification for user: " + userId);
        
        // Polymorphism handles the details
        messageSender.send(userId, message);
    }
}

// Usage - wiring happens at composition root
public class Main {
    public static void main(String[] args) {
        // Easy to switch implementations
        
        // Using Email
        MessageSender emailSender = new EmailSender();
        NotificationService emailNotifications = new NotificationService(emailSender);
        emailNotifications.notifyUser("john123", "Your order is confirmed");
        
        System.out.println();
        
        // Using SMS - no changes to NotificationService!
        MessageSender smsSender = new SmsSender();
        NotificationService smsNotifications = new NotificationService(smsSender);
        smsNotifications.notifyUser("555-1234", "Your order is confirmed");
        
        System.out.println();
        
        // Using Push - still no changes!
        MessageSender pushSender = new PushNotificationSender();
        NotificationService pushNotifications = new NotificationService(pushSender);
        pushNotifications.notifyUser("device-token-123", "Your order is confirmed");
    }
}
```

**Output:**
```
Preparing notification for user: john123
Sending EMAIL to john123
Message: Your order is confirmed

Preparing notification for user: 555-1234
Sending SMS to 555-1234
Message: Your order is confirmed

Preparing notification for user: device-token-123
Sending PUSH notification to device-token-123
Message: Your order is confirmed
```

**Why This Is Better:**

1. **Testable**: Inject a `MockMessageSender` for unit tests
2. **Flexible**: Switch implementations without touching `NotificationService`
3. **Extensible**: Add new senders (Slack, WhatsApp) without modifications
4. **Loosely Coupled**: High-level and low-level modules are independent

### DIP in Testing

```java
// Mock implementation for testing
public class MockMessageSender implements MessageSender {
    private String lastRecipient;
    private String lastMessage;
    private int sendCount = 0;
    
    @Override
    public void send(String recipient, String message) {
        this.lastRecipient = recipient;
        this.lastMessage = message;
        this.sendCount++;
    }
    
    // Test helpers
    public String getLastRecipient() { return lastRecipient; }
    public String getLastMessage() { return lastMessage; }
    public int getSendCount() { return sendCount; }
}

// Unit test
public class NotificationServiceTest {
    public void testNotifyUser() {
        // Arrange
        MockMessageSender mockSender = new MockMessageSender();
        NotificationService service = new NotificationService(mockSender);
        
        // Act
        service.notifyUser("testUser", "Test message");
        
        // Assert
        assert mockSender.getSendCount() == 1;
        assert mockSender.getLastRecipient().equals("testUser");
        assert mockSender.getLastMessage().equals("Test message");
        
        System.out.println("Test passed!");
    }
}
```

### Real-World Usage - How Spring Framework Applies DIP

**Spring's Dependency Injection** is the implementation of Dependency Inversion Principle.

```java
// Interface
public interface PaymentGateway {
    boolean processPayment(double amount);
}

// Implementations
@Component
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public boolean processPayment(double amount) {
        System.out.println("Processing $" + amount + " via Stripe");
        return true;
    }
}

@Component
public class PayPalPaymentGateway implements PaymentGateway {
    @Override
    public boolean processPayment(double amount) {
        System.out.println("Processing $" + amount + " via PayPal");
        return true;
    }
}

// High-level service depends on abstraction
@Service
public class OrderService {
    
    private final PaymentGateway paymentGateway;
    
    // Spring automatically injects the implementation
    @Autowired
    public OrderService(@Qualifier("stripePaymentGateway") PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
    
    public void placeOrder(double amount) {
        System.out.println("Placing order for $" + amount);
        boolean paymentSuccess = paymentGateway.processPayment(amount);
        
        if (paymentSuccess) {
            System.out.println("Order placed successfully");
        }
    }
}
```

**Benefits in Spring:**
- Configuration determines which implementation is injected
- Can swap implementations via configuration without code changes
- Easy to inject mocks in integration tests
- Loose coupling throughout the application

### DIP vs Dependency Injection

**Question**: What's the difference between Dependency Inversion Principle and Dependency Injection?

**Answer**:

**Dependency Inversion Principle (DIP)**: A design principle stating that high-level and low-level modules should depend on abstractions.

**Dependency Injection (DI)**: A design pattern that implements DIP by providing dependencies from outside rather than creating them inside.

**Analogy**:
- **DIP** is the philosophy: "Don't create your own tools; accept tools from outside"
- **DI** is the practice: "Here's how you receive those tools (constructor, setter, etc.)"

**Example**:
```java
// DIP: Depend on interface
public class Service {
    private Repository repository; // Interface, not concrete class
}

// DI: Inject the dependency
public Service(Repository repository) {
    this.repository = repository; // Injected from outside
}
```

---

## SOLID Principles Side-by-Side Summary

| Principle | Purpose | Problem Solved | Key Benefit | Common Violation |
|-----------|---------|----------------|-------------|------------------|
| **Single Responsibility (SRP)** | A class should have one reason to change | "God classes" doing everything | High cohesion, easier maintenance | Class names with "Manager" or "And"; 1000+ line classes |
| **Open/Closed (OCP)** | Open for extension, closed for modification | Fragile code breaking with new features | Extensibility without risk | Long if-else chains checking types; frequent modifications to existing code |
| **Liskov Substitution (LSP)** | Subclasses must be substitutable for parents | Unexpected runtime errors with inheritance | Reliable polymorphism | Methods throwing `UnsupportedOperationException`; empty implementations |
| **Interface Segregation (ISP)** | Don't force clients to implement unused methods | "Fat interfaces" with unrelated methods | Decoupling, cleaner contracts | Interfaces with 10+ methods; forced empty implementations |
| **Dependency Inversion (DIP)** | Depend on abstractions, not concretions | Tight coupling; untestable code | Loose coupling, testability | Using `new` keyword in business logic; direct dependencies on concrete classes |

### Quick Decision Guide

**When reviewing code, ask:**

1. **SRP**: "If I change the database, do I also have to modify the email sender?" → If yes, violates SRP
2. **OCP**: "Can I add a new payment method without editing existing classes?" → If no, violates OCP
3. **LSP**: "Can I replace this parent class with any of its children without crashes?" → If no, violates LSP
4. **ISP**: "Is this class forced to implement methods it doesn't use?" → If yes, violates ISP
5. **DIP**: "Is this business logic class directly creating database connections?" → If yes, violates DIP

---

## SOLID in Real Java Projects

### Where SOLID Fits

#### REST APIs

```java
// SRP: Separate concerns
@RestController
public class UserController {
    // Only handles HTTP concerns
}

@Service
public class UserService {
    // Only handles business logic
}

@Repository
public class UserRepository {
    // Only handles data access
}

// DIP: Depend on interfaces
public class UserService {
    private final UserRepository repository; // Interface
    
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}

// OCP: Easy to add new endpoints without modifying existing ones
@GetMapping("/users")
@PostMapping("/users")
@DeleteMapping("/users/{id}")
```

#### Microservices

```java
// SRP at architectural level
UserService // Only user-related operations
OrderService // Only order-related operations
PaymentService // Only payment operations

// DIP: Services depend on contracts, not implementations
public interface EventPublisher {
    void publish(Event event);
}

@Service
public class OrderService {
    private final EventPublisher publisher; // Abstract
    
    public void placeOrder(Order order) {
        // Business logic
        publisher.publish(new OrderPlacedEvent(order));
    }
}
```

#### Enterprise Applications

```java
// ISP: Separate read and write operations (CQRS pattern)
public interface ReadRepository {
    List<User> findAll();
    User findById(Long id);
}

public interface WriteRepository {
    void save(User user);
    void delete(Long id);
}

// LSP: Proper inheritance in domain models
public abstract class Account {
    public abstract void calculateInterest();
}

public class SavingsAccount extends Account {
    @Override
    public void calculateInterest() {
        // Implementation that honors parent contract
    }
}
```

### How SOLID Reduces Bugs and Change Cost

**Without SOLID**:
```
Change request: "Add SMS notifications"
↓
Modify NotificationService (risk of breaking emails)
↓
Modify tests for NotificationService
↓
Full regression testing needed
↓
Deploy entire application
Cost: 5 days, high risk
```

**With SOLID**:
```
Change request: "Add SMS notifications"
↓
Create SmsSender implementing MessageSender
↓
Inject SmsSender where needed
↓
Test only SmsSender (isolated)
↓
Deploy new class (no changes to existing code)
Cost: 1 day, low risk
```

**Measurable Benefits**:
- **Fewer bugs**: Changes are localized
- **Faster development**: Parallel work on different implementations
- **Easier testing**: Mock dependencies easily
- **Lower technical debt**: Code remains clean over time
- **Reduced cost**: Less regression testing, fewer rewrites

---

## Common Mistakes & Misconceptions

### Mistake 1: Over-Engineering

**Wrong**:
```java
// Creating interfaces for everything, even simple utilities
public interface MathCalculator {
    int add(int a, int b);
}

public class BasicMathCalculator implements MathCalculator {
    @Override
    public int add(int a, int b) {
        return a + b;
    }
}
```

**Right**:
```java
// Simple utility doesn't need interface
public class MathUtils {
    public static int add(int a, int b) {
        return a + b;
    }
}
```

**When to apply SOLID**: When you anticipate change or need testability. Not for trivial utilities.

### Mistake 2: Blindly Applying SOLID

**Wrong Mindset**: "Every class must have an interface!"

**Right Mindset**: "Where do I anticipate change? Where do I need testability?"

**Example**:
```java
// Don't need an interface for DTOs or value objects
public class UserDTO {
    private String username;
    private String email;
    // Getters/setters
}

// DO need interface for services with business logic
public interface PaymentProcessor {
    boolean process(Payment payment);
}
```

### Mistake 3: Misusing Inheritance

**Wrong**:
```java
// Inheritance for code reuse
public class List {
    public void add(Object item) { /* */ }
}

public class Stack extends List {
    // Inheriting add() when we want push()
}
```

**Right**:
```java
// Composition over inheritance
public class Stack {
    private List list = new ArrayList();
    
    public void push(Object item) {
        list.add(item);
    }
}
```

**Rule**: Use inheritance for "is-a" relationships that follow LSP. Use composition for "has-a" or "uses-a".

### Mistake 4: Confusing SOLID with Design Patterns

**SOLID** = Principles (guidelines)
**Design Patterns** = Solutions (implementations)

**Example**:
- **OCP** is the principle
- **Strategy Pattern** is one way to implement OCP

```java
// OCP principle: Open for extension, closed for modification

// Strategy pattern implements OCP
public interface SortStrategy {
    void sort(int[] array);
}

public class QuickSort implements SortStrategy { /* */ }
public class MergeSort implements SortStrategy { /* */ }
```

### Mistake 5: Premature Abstraction

**Wrong**: Creating abstractions before you need them

```java
// Creating interfaces "just in case" for every class
public interface Calculator { }
public interface Logger { }
public interface Parser { }
// ... when there's only one implementation and no plans for more
```

**Right**: Apply abstractions when:
- You have multiple implementations
- You need to mock for testing
- You anticipate change

**YAGNI Principle**: You Aren't Gonna Need It. Don't write code for hypothetical future requirements.

---

## Interview Questions & Answers

### Q1: Explain Single Responsibility Principle with a code example

**Answer**:

"SRP states that a class should have only one reason to change. This means it should be responsible for one aspect of the functionality.

Bad example:
```java
public class User {
    public void save() {
        // Database logic - one responsibility
    }
    
    public void sendEmail() {
        // Email logic - another responsibility
    }
}
```

Good example:
```java
public class User {
    // Only data
}

public class UserRepository {
    public void save(User user) {
        // Database logic
    }
}

public class UserNotifier {
    public void sendEmail(User user) {
        // Email logic
    }
}
```

Now, changes to email logic don't affect data persistence."

---

### Q2: What's the difference between ISP and SRP?

**Answer**:

"Both promote focused design, but they apply to different levels:

**SRP** is about **class implementation**: A class should have one responsibility. It prevents 'God classes' that do everything.

**ISP** is about **interface design**: An interface should represent one capability. It prevents 'fat interfaces' that force clients to implement unused methods.

Example:
```java
// SRP: UserService has one responsibility
public class UserService {
    // Only user business logic
}

// ISP: Separate interfaces for different capabilities
public interface Readable {
    void read();
}

public interface Writable {
    void write();
}

// Class can be read-only without implementing write()
public class ReadOnlyFile implements Readable {
    public void read() { }
}
```

SRP makes implementation cohesive. ISP makes interfaces focused."

---

### Q3: How does Spring Framework use Dependency Inversion?

**Answer**:

"Spring is built on DIP. In Spring:

1. **Define abstractions** (interfaces):
```java
public interface PaymentService {
    void processPayment(double amount);
}
```

2. **Implement concrete classes**:
```java
@Component
public class StripePaymentService implements PaymentService {
    public void processPayment(double amount) {
        // Stripe-specific logic
    }
}
```

3. **Depend on abstractions in high-level code**:
```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    
    @Autowired // Spring injects the implementation
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

4. **Spring injects implementations** at runtime based on configuration.

This means:
- OrderService doesn't know about Stripe
- We can swap to PayPal without changing OrderService
- Easy to inject mocks in tests

Spring's IoC container is essentially a DIP enforcement tool."

---

### Q4: Identify SOLID violations in this code:

```java
public class Order {
    public void calculateTotal() { }
    public void saveToDatabase() { }
    public void sendConfirmationEmail() { }
    public void printInvoice() { }
}
```

**Answer**:

"This violates **Single Responsibility Principle**. The Order class has four reasons to change:

1. Business logic changes (calculateTotal)
2. Database changes (saveToDatabase)
3. Email provider changes (sendConfirmationEmail)
4. Printing format changes (printInvoice)

Correct design:
```java
public class Order {
    // Only data and domain logic
    public double calculateTotal() { }
}

public class OrderRepository {
    public void save(Order order) { }
}

public class OrderNotifier {
    public void sendEmail(Order order) { }
}

public class InvoicePrinter {
    public void print(Order order) { }
}
```"

---

### Q5: When should you NOT follow SOLID principles?

**Answer**:

"SOLID should be balanced with pragmatism:

**Don't apply when:**

1. **Simple utilities**: No need for interfaces on `Math.add()`
2. **DTOs or value objects**: Just hold data
3. **Prototypes**: Early-stage code that will be rewritten
4. **Performance-critical code**: Sometimes direct coupling is faster
5. **Over-abstraction**: Creating 10 classes for a simple feature

**Do apply when:**

1. Business logic that will evolve
2. Code that needs testing
3. Multiple implementations exist or are likely
4. Large team collaboration
5. Long-lived production systems

The goal is maintainable code, not maximum abstraction. Use YAGNI (You Aren't Gonna Need It) and KISS (Keep It Simple) alongside SOLID."

---

### Q6: Explain Liskov Substitution with a practical example

**Answer**:

"LSP says you should be able to replace a parent class with any child class without breaking the program.

Classic violation - Square/Rectangle:
```java
class Rectangle {
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
}

class Square extends Rectangle {
    public void setWidth(int w) {
        this.width = w;
        this.height = w; // Breaks expected behavior!
    }
}

// This breaks with Square
void testRectangle(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.getArea() == 20; // Fails for Square!
}
```

Correct approach - don't use inheritance:
```java
interface Shape {
    int getArea();
}

class Rectangle implements Shape { }
class Square implements Shape { }
```

In interviews, mention that LSP violations often appear as:
- Throwing `UnsupportedOperationException`
- Empty method implementations
- Type checking with `instanceof`
- Subclass strengthening preconditions"

---

### Q7: Design a notification system following SOLID principles

**Answer**:

```java
// OCP & DIP: Depend on abstraction
public interface NotificationChannel {
    void send(String recipient, String message);
}

// ISP: Focused interfaces for different capabilities
public interface SchedulableChannel {
    void scheduleNotification(String recipient, String message, LocalDateTime time);
}

// Implementations
public class EmailChannel implements NotificationChannel {
    public void send(String recipient, String message) {
        // Email logic
    }
}

public class SmsChannel implements NotificationChannel, SchedulableChannel {
    public void send(String recipient, String message) {
        // SMS logic
    }
    
    public void scheduleNotification(String recipient, String message, LocalDateTime time) {
        // Schedule SMS
    }
}

// SRP: Service only orchestrates
public class NotificationService {
    private final NotificationChannel channel;
    
    // DIP: Inject dependency
    public NotificationService(NotificationChannel channel) {
        this.channel = channel;
    }
    
    public void notify(String recipient, String message) {
        channel.send(recipient, message);
    }
}

// LSP: All channels can substitute each other
public void sendNotification(NotificationChannel channel, String user, String msg) {
    channel.send(user, msg); // Works with any implementation
}

// Usage
NotificationService emailService = new NotificationService(new EmailChannel());
NotificationService smsService = new NotificationService(new SmsChannel());
```

This design:
- **SRP**: Each class has one purpose
- **OCP**: Add new channels without modifying existing code
- **LSP**: All channels work interchangeably
- **ISP**: Schedulable is separate from basic sending
- **DIP**: Service depends on abstraction"

---

## Final Cheat Sheet - One-Screen Revision

### The SOLID Quick Reference

**S - Single Responsibility Principle**
- **Rule**: One class, one reason to change
- **Apply when**: Class has multiple unrelated responsibilities
- **Don't apply when**: Simple utilities or DTOs
- **Red flag**: Class name has "And" or "Manager"

**O - Open/Closed Principle**
- **Rule**: Open for extension, closed for modification
- **Apply when**: Anticipating new variations of behavior
- **Don't apply when**: Requirements are stable and simple
- **Red flag**: Long if-else chains checking types

**L - Liskov Substitution Principle**
- **Rule**: Subtypes must be substitutable for parent types
- **Apply when**: Using inheritance
- **Don't apply when**: Composition is better (usually is)
- **Red flag**: Throwing `UnsupportedOperationException`

**I - Interface Segregation Principle**
- **Rule**: Many small interfaces > one fat interface
- **Apply when**: Clients need different subsets of methods
- **Don't apply when**: All clients need all methods
- **Red flag**: Implementing methods you don't use

**D - Dependency Inversion Principle**
- **Rule**: Depend on abstractions, not concretions
- **Apply when**: Need testability or flexibility
- **Don't apply when**: Simple, stable utilities
- **Red flag**: Using `new` keyword in business logic

### Quick Violation Detection

```java
// Violates SRP
public class God {
    void doEverything() { }
}

// Violates OCP
public void process(String type) {
    if (type.equals("A")) { } // Modify for each new type
    else if (type.equals("B")) { }
}

// Violates LSP
public class Child extends Parent {
    @Override
    void method() {
        throw new UnsupportedOperationException();
    }
}

// Violates ISP
public interface Everything {
    void method1();
    void method2();
    void method3(); // Too many methods
}

// Violates DIP
public class Service {
    private Database db = new MySQL(); // Concrete dependency
}
```

### The 5-Second Test

**Ask yourself**:
1. If I change X, do I have to change Y? → **SRP issue**
2. Can I add behavior without editing code? → **OCP issue**
3. Can I swap parent with child? → **LSP issue**
4. Am I implementing unused methods? → **ISP issue**
5. Am I creating dependencies with `new`? → **DIP issue**

### Interview Shortcuts

**What interviewers listen for**:
- ✅ Practical examples over definitions
- ✅ Knowing when NOT to apply
- ✅ Understanding trade-offs
- ✅ Real project experience

**What to avoid**:
- ❌ Reciting definitions verbatim
- ❌ Dogmatic "always use interfaces"
- ❌ Confusing principles with patterns
- ❌ No practical code examples

### The SOLID Mindset

> "Write code that's easy to change, not just code that works."

**Remember**:
- SOLID prevents pain in **month 6**, not **day 1**
- Balance SOLID with KISS and YAGNI
- Refactor toward SOLID as complexity grows
- Code quality is about **maintainability**, not perfection

---

## Conclusion

SOLID principles are the foundation of professional object-oriented design. They're not academic theory—they're battle-tested guidelines from decades of software maintenance nightmares.

**Key Takeaways**:

1. **SRP** keeps classes focused and cohesive
2. **OCP** makes systems extensible without breaking existing code
3. **LSP** ensures inheritance hierarchies are reliable
4. **ISP** prevents interface bloat and unnecessary coupling
5. **DIP** enables testability and flexibility through abstractions

**In Practice**:
- Apply SOLID gradually as code complexity grows
- Balance principles with pragmatism (YAGNI, KISS)
- Use SOLID to guide refactoring decisions
- Recognize violations during code reviews

**For Interviews**:
- Explain principles with concrete examples
- Discuss trade-offs and when NOT to apply
- Demonstrate understanding of how frameworks use SOLID
- Show experience with real-world refactoring

Master SOLID, and you'll write code that your future self (and your team) will thank you for.

---

**Document Version**: 1.0  
**Last Updated**: January 2026  
**Target Audience**: Java developers, interview preparation, clean code enthusiasts  
**Prerequisites**: Basic Java and OOP knowledge