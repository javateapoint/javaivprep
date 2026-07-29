# Top Core Java Interview Questions & Answers

Comprehensive core Java interview questions and answers covering JVM architecture, OOP concepts, data types, classloaders, and method behavior.

---

## Q1. What is JVM? Why is Java called platform-independent?

**JVM (Java Virtual Machine)**: It is the abstract computing machine/runtime engine that executes Java bytecode (.class files). JVM converts bytecode into native machine instructions specific to the underlying operating system.

**Platform Independence**:
1. Java source code (`.java`) is compiled by `javac` into intermediate bytecode (`.class`).
2. Bytecode is CPU-independent and platform-neutral.
3. Any operating system with a platform-specific JVM can run the same bytecode.
4. Hence, Java achieves **"Write Once, Run Anywhere" (WORA)**.

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

---

## Q2. What is the difference between JDK, JRE, and JVM?

| Component | Description | Included Tools |
| :--- | :--- | :--- |
| **JDK (Java Development Kit)** | Complete software development environment required to develop and execute Java applications. | JRE + Development tools (`javac`, `jar`, `javadoc`, `jcmd`). |
| **JRE (Java Runtime Environment)** | Runtime environment containing core libraries and JVM to execute Java applications. | JVM + Core runtime libraries (`rt.jar` / Modules). |
| **JVM (Java Virtual Machine)** | Runtime engine that executes compiled bytecode line-by-line or via JIT compilation. | JIT Compiler, Garbage Collector, Memory Manager. |

---

## Q3. What are the key features of Java?

- **Object-Oriented**: Everything in Java (except primitives) is centered around objects and classes.
- **Platform Independent**: Compiled into platform-agnostic bytecode.
- **Robust and Secure**: Automatic memory management (GC), strong type checking, explicit exception handling, and no raw pointers.
- **Multithreaded**: Built-in support for concurrent execution via threads and Virtual Threads (Java 21).
- **Distributed**: Supports network programming, RMI/REST/gRPC protocols.
- **Dynamic and Extensible**: Loads classes on demand at runtime and supports reflection.

---

## Q4. What is the difference between PATH and CLASSPATH environment variables?

- **PATH**: Tells the Operating System where to locate executable binary commands (e.g., `java`, `javac`, `mvn`).
- **CLASSPATH**: Tells the JVM and Java compiler where to find user-defined `.class` files, packages, and third-party JAR dependencies.

```bash
# Example Setup
export PATH=/usr/lib/jvm/java-21-openjdk/bin:$PATH
export CLASSPATH=.:/home/user/app/lib/*
```

---

## Q5. What are primitive data types in Java?

Java has 8 primitive data types categorized as:

| Type | Size | Default Value | Range / Value |
| :--- | :--- | :--- | :--- |
| `byte` | 1 byte (8 bits) | `0` | -128 to 127 |
| `short` | 2 bytes (16 bits) | `0` | -32,768 to 32,767 |
| `int` | 4 bytes (32 bits) | `0` | -2^31 to 2^31-1 |
| `long` | 8 bytes (64 bits) | `0L` | -2^63 to 2^63-1 |
| `float` | 4 bytes (32 bits) | `0.0f` | IEEE 754 floating-point |
| `double` | 8 bytes (64 bits) | `0.0d` | IEEE 754 floating-point |
| `char` | 2 bytes (16 bits) | `'\u0000'` | Unicode character (0 to 65,535) |
| `boolean` | ~1 bit / 1 byte | `false` | `true` or `false` |

---

## Q6. What are Wrapper Classes?

Wrapper classes box primitive data types into object representations.

- `byte` → `Byte`
- `short` → `Short`
- `int` → `Integer`
- `long` → `Long`
- `float` → `Float`
- `double` → `Double`
- `char` → `Character`
- `boolean` → `Boolean`

They enable primitives to be used in Java Collections (`List<Integer>`), generics, and stream operations.

---

## Q7. What is Autoboxing and Unboxing?

- **Autoboxing**: Automatic conversion of a primitive type into its corresponding wrapper class object (`int` to `Integer`).
- **Unboxing**: Automatic conversion of a wrapper class object into its primitive type (`Integer` to `int`).

```java
Integer wrapper = 10; // Autoboxing (Integer.valueOf(10))
int primitive = wrapper; // Unboxing (wrapper.intValue())
```

---

## Q8. What is a Constructor?

A constructor is a special block of code used to initialize newly created objects. It has the exact same name as its class and has no explicit return type.

```java
class Person {
    private String name;

    // No-arg constructor
    public Person() {
        this.name = "Unknown";
    }

    // Parameterized constructor
    public Person(String name) {
        this.name = name;
    }
}
```

---

## Q9. What is Constructor Overloading?

Defining multiple constructors within the same class, each having a different parameter list (number, type, or order of parameters).

```java
class Person {
    private String name;
    private int age;

    public Person() {}
    public Person(String name) { this.name = name; }
    public Person(String name, int age) { this.name = name; this.age = age; }
}
```

---

## Q10. What is a static block in Java?

A `static` block is executed **once** when the JVM loads the class into memory, prior to object instantiation or main method execution. It is primarily used for static member initialization.

```java
class Config {
    static String dbUrl;
    
    static {
        dbUrl = System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://localhost:5432/db");
        System.out.println("Static block executed: Config initialized.");
    }
}
```

---

## Q11. Can we execute a Java program without a `main` method?

Prior to Java 7, a static initializer block could execute code before the missing `main` error was raised. From Java 7 onwards, the JVM explicitly checks for `public static void main(String[] args)` before initializing the class, so standalone programs require a main entry point (or modern Java 21 instance main methods).

---

## Q12. Difference between static and non-static methods?

- **Static Methods**: Associated with the class itself. Can be invoked using `ClassName.methodName()` without instantiating an object. Cannot directly access instance variables or `this`/`super`.
- **Non-Static Methods**: Associated with a specific object instance. Can access both static and instance fields.

---

## Q13. Difference between Method Overloading and Method Overriding?

| Aspect | Method Overloading | Method Overriding |
| :--- | :--- | :--- |
| **Scope** | Same class. | Subclass inheriting parent class. |
| **Parameters** | Must be different (count, types, or order). | Must be identical. |
| **Return Type** | Can be different. | Must be same or covariant subclass type. |
| **Binding Time** | Compile-time (Static polymorphism). | Runtime (Dynamic polymorphism). |

---

## Q14. What is the `final` keyword used for in Java?

- **Final Variable**: Creates a constant whose value cannot be reassigned once initialized.
- **Final Method**: Prevents subclasses from overriding the method.
- **Final Class**: Prevents other classes from extending/inheriting the class.

---

## Q15. Difference between `==` and `equals()`?

- **`==` Operator**: Compares reference memory addresses for object types (checks if both references point to identical memory location). For primitives, it compares primitive values.
- **`equals()` Method**: Compares logical values/content equality of objects. Must be overridden in custom classes alongside `hashCode()`.

```java
String s1 = "hello";
String s2 = new String("hello");

System.out.println(s1 == s2);       // false (different objects in heap)
System.out.println(s1.equals(s2));  // true (same string content)
```

---

## Q16. Purpose of `equals()` and `hashCode()` contract?

When using objects in hash-based collections (`HashMap`, `HashSet`, `ConcurrentHashMap`):
1. If `a.equals(b)` is `true`, then `a.hashCode()` **MUST** equal `b.hashCode()`.
2. If two objects have identical hash codes, they are **NOT required** to be equal (hash collision).
3. Overriding `equals()` without overriding `hashCode()` breaks `HashMap` retrieval logic.

---

## Q17. Difference between Abstract Class and Interface (Java 8+ to 21)

| Feature | Interface | Abstract Class |
| :--- | :--- | :--- |
| **Inheritance** | A class can implement multiple interfaces. | A class can extend only one abstract class. |
| **Fields** | Only `public static final` constants. | Can have instance fields with any access modifier. |
| **Methods** | Abstract, `default`, `static`, and `private` methods. | Abstract methods, concrete methods, static methods. |
| **Constructor** | Cannot have constructors. | Can have constructors called during subclass instantiation. |
