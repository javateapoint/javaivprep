# Java 17 LTS Features Guide

A comprehensive guide to key Java 17 Long-Term Support (LTS) features, complete with code examples and production benefits.

---

## 1. Records (`java.lang.Record`)

Records provide a compact syntax for declaring transparent immutable data-carrier classes.

```java
// Automatically generates: private final fields, constructor, getters, equals(), hashCode(), and toString()
public record UserDto(Long id, String username, String email) {

    // Compact Constructor for Validation
    public UserDto {
        if (username == null || username.isBlank()) {
            throw new IllegalArgumentException("Username cannot be empty");
        }
    }

    // Custom instance method
    public String domainName() {
        return email.substring(email.indexOf("@") + 1);
    }
}
```

---

## 2. Sealed Classes and Interfaces (JEP 409)

Sealed classes give developers explicit control over class hierarchy inheritance by declaring which subclasses are permitted to extend or implement them.

```java
// Abstract sealed parent class permitting exact subclasses
public abstract sealed class Shape permits Circle, Rectangle, Triangle {
    public abstract double area();
}

// Subclasses MUST be declared as final, sealed, or non-sealed
public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    @Override public double area() { return Math.PI * radius * radius; }
}

public final class Rectangle extends Shape {
    private final double width, height;
    public Rectangle(double width, double height) { this.width = width; this.height = height; }
    @Override public double area() { return width * height; }
}

// Non-sealed allows un-restricted subclassing downstream
public non-sealed class Triangle extends Shape {
    private final double base, height;
    public Triangle(double base, double height) { this.base = base; this.height = height; }
    @Override public double area() { return 0.5 * base * height; }
}
```

### Benefits
- **Exhaustiveness checking** in switch expressions without requiring default cases.
- **Stronger domain modeling** and encapsulation.

---

## 3. Pattern Matching for `switch` (JEP 406 / 427)

Extends pattern matching to `switch` expressions and statements, permitting type-safe dispatch with pattern guards.

```java
public class SwitchPatternMatching {

    public static String formatValue(Object obj) {
        return switch (obj) {
            case Integer i -> String.format("Integer value: %d", i);
            case Long l    -> String.format("Long value: %d", l);
            case Double d  -> String.format("Double value: %.2f", d);
            case String s && s.length() > 10 -> "Long String: " + s.substring(0, 10) + "...";
            case String s  -> "Short String: " + s;
            case Circle c  -> String.format("Circle with radius %.2f", c.area());
            case null      -> "Null value provided";
            default        -> obj.toString();
        };
    }
}
```

---

## 4. Pattern Matching for `instanceof` (JEP 394)

Combines type checking and casting into a single atomic operation.

```java
// Traditional Approach
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.toLowerCase());
}

// Modern Pattern Matching Approach
if (obj instanceof String s && s.length() > 5) {
    // 's' is automatically cast and scoped to the true branch
    System.out.println(s.toLowerCase());
}
```

---

## 5. Text Blocks (JEP 378)

Multi-line string literals that avoid escape sequence clutter and preserve formatting.

```java
String jsonPayload = """
        {
            "service": "order-service",
            "status": "ACTIVE",
            "retryAttempts": 3
        }
        """;
```

---

## 6. Enhanced Pseudo-Random Number Generators (JEP 356)

Introduced `java.util.random.RandomGenerator` providing jumpable, splittable PRNG algorithms.

```java
import java.util.random.RandomGenerator;
import java.util.random.RandomGeneratorFactory;

public class RandomGeneratorExample {
    public static void main(String[] args) {
        RandomGenerator rng = RandomGenerator.of("L64X128MixRandom");
        int randomInt = rng.nextInt(1, 100);
        System.out.println("Generated Random Number: " + randomInt);
    }
}
```
