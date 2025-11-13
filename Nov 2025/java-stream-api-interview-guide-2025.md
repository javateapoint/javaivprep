# Java Stream API: Complete Hands-On Interview Guide 2025

## Advanced Stream API with Real-Time Examples, Tricky Questions & Detailed Answers

**Prepared on:** November 13, 2025  
**Target Audience:** Senior Java Developers & Above  
**Interview Focus:** 2025 Technical Rounds

---

## Table of Contents

1. [Stream API Fundamentals](#stream-api-fundamentals)
2. [Stream Pipeline Architecture](#stream-pipeline-architecture)
3. [Intermediate Operations (Part 1: Basic)](#intermediate-operations-basic)
4. [Intermediate Operations (Part 2: Advanced)](#intermediate-operations-advanced)
5. [Terminal Operations](#terminal-operations)
6. [Reduction Operations (reduce, collect)](#reduction-operations)
7. [Map vs FlatMap - Deep Dive](#map-vs-flatmap)
8. [Collectors - Advanced Usage](#collectors-advanced-usage)
9. [Parallel Streams - Performance & Pitfalls](#parallel-streams)
10. [Lazy Evaluation & Short-Circuiting](#lazy-evaluation)
11. [Stateless vs Stateful Operations](#stateless-vs-stateful)
12. [Non-Interfering & Immutability](#non-interfering)
13. [Encounter Order & Sequential vs Parallel](#encounter-order)
14. [Debugging Streams with peek()](#debugging-streams)
15. [Real-Time Interview Challenges](#real-time-challenges)
16. [Tricky Questions & Common Mistakes](#tricky-questions)
17. [Performance Optimization Best Practices](#performance-optimization)
18. [Interview Day Preparation Checklist](#interview-checklist)

---

## 1. Stream API Fundamentals

### What is Stream API?

**Definition:** Stream API is a functional programming interface introduced in Java 8 to process sequences of elements in a declarative way, allowing you to express complex data processing queries in a concise and readable manner.

**Key Characteristics:**
- **Declarative**: Focus on "what" to do, not "how"
- **Lazy Evaluation**: Operations don't execute until a terminal operation is called
- **Immutable**: Streams don't modify the original source
- **Composable**: Operations can be chained together
- **One-time Use**: Once consumed, a stream cannot be reused

### Streams vs Collections

```java
// Collections (Imperative - How)
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> evenNumbers = new ArrayList<>();
for (Integer n : numbers) {
    if (n % 2 == 0) {
        evenNumbers.add(n);
    }
}

// Streams (Declarative - What)
List<Integer> evenNumbers = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
```

**Detailed Comparison:**

| Aspect | Stream | Collection |
|--------|--------|-----------|
| **Nature** | Immutable | Mutable |
| **Storage** | No storage (processes data) | Stores elements in memory |
| **Data Flow** | Lazy evaluation | Eager evaluation |
| **Reusability** | One-time use | Multiple iterations |
| **External Iteration** | Internal (handled by stream) | External (we control) |
| **Performance** | Lazy: processes when needed | All elements processed upfront |

### Interview Question 1: Basic Understanding

**Q: Why would you prefer Stream API over traditional loops in Java?**

**Answer:**
1. **Readability**: More concise and expressive code
```java
// Traditional way
List<String> upperNames = new ArrayList<>();
for (String name : names) {
    if (!name.isEmpty()) {
        upperNames.add(name.toUpperCase());
    }
}

// Stream way
List<String> upperNames = names.stream()
    .filter(n -> !n.isEmpty())
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

2. **Functional Composition**: Chain operations naturally
3. **Parallel Processing**: Easy to parallelize
4. **Less Error-Prone**: Fewer mutable variables to track
5. **Better Performance**: In many scenarios due to lazy evaluation

---

## 2. Stream Pipeline Architecture

### The Three Components of a Stream Pipeline

```
SOURCE → INTERMEDIATE OPERATIONS → TERMINAL OPERATION
```

#### **1. Source**
Provides initial data to the stream.

```java
// From Collection
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> s1 = list.stream();

// From Array
String[] arr = {"a", "b", "c"};
Stream<String> s2 = Arrays.stream(arr);

// Using Stream.of()
Stream<String> s3 = Stream.of("a", "b", "c");

// Using Stream.builder()
Stream<String> s4 = Stream.<String>builder()
    .add("a").add("b").add("c")
    .build();

// From File (NIO)
Stream<String> s5 = Files.lines(Paths.get("data.txt"));

// Infinite Streams
Stream<Double> s6 = Stream.generate(Math::random).limit(5);
Stream<Integer> s7 = Stream.iterate(1, n -> n + 2).limit(5);

// From Primitive Arrays
IntStream s8 = Arrays.stream(new int[]{1, 2, 3});

// Primitive Stream Generators
IntStream s9 = IntStream.range(1, 5);        // 1,2,3,4
IntStream s10 = IntStream.rangeClosed(1, 5); // 1,2,3,4,5
```

#### **2. Intermediate Operations**
Transform the stream without consuming it. Always return a new stream.

```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5);

Stream<Integer> intermediate = nums.stream()
    .filter(n -> n > 2)              // Intermediate
    .map(n -> n * 2)                 // Intermediate
    .distinct()                       // Intermediate
    // Nothing executed yet!
```

**Key Point**: Intermediate operations are **lazy** - they don't do anything until a terminal operation is called.

#### **3. Terminal Operations**
Consume the stream and produce a result or side-effect.

```java
nums.stream()
    .filter(n -> n > 2)
    .map(n -> n * 2)
    .forEach(System.out::println);  // Terminal - Executes everything
```

### Real-World Example: E-Commerce Order Processing

```java
class Order {
    private int orderId;
    private double amount;
    private String status;
    private LocalDate date;
    
    // Constructor and getters
}

// Find total value of completed orders from last month
double totalAmount = orders.stream()
    // Source: List<Order>
    
    // Intermediate: Filter active orders from last month
    .filter(order -> order.getStatus().equals("COMPLETED"))
    .filter(order -> order.getDate().getYear() == 2025)
    .filter(order -> order.getDate().getMonthValue() == 11)
    
    // Intermediate: Transform to get only amount
    .map(Order::getAmount)
    
    // Terminal: Sum all amounts
    .reduce(0.0, Double::sum);
    // OR: .sum() if using DoubleStream

System.out.println("Total: " + totalAmount);
```

**Execution Flow:**
1. No operations execute when intermediate operations are added
2. Only when `reduce()` (terminal) is called, the pipeline activates
3. Each order flows through: filter status → filter year → filter month → map amount → reduce sum

---

## 3. Intermediate Operations (Part 1: Basic)

### 1. filter(Predicate)

Selects elements that match a condition.

```java
// Example 1: Filter even numbers
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6);
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
// Output: [2, 4, 6]

// Example 2: Filter employees by salary
List<Employee> employees = getEmployees();
List<Employee> highEarners = employees.stream()
    .filter(e -> e.getSalary() > 100000)
    .filter(e -> e.getYearsExperience() > 5)
    .collect(Collectors.toList());

// Example 3: Complex predicate
List<String> words = Arrays.asList("apple", "banana", "apricot", "cherry");
List<String> aWords = words.stream()
    .filter(word -> word.startsWith("a") && word.length() > 5)
    .collect(Collectors.toList());
// Output: ["apple", "apricot"]
```

**Interview Tip**: You can chain multiple `filter()` operations or combine conditions with &&/||.

### 2. map(Function)

Transforms each element to another form (1-to-1 mapping).

```java
// Example 1: Convert strings to uppercase
List<String> names = Arrays.asList("alice", "bob", "charlie");
List<String> upperNames = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// Output: ["ALICE", "BOB", "CHARLIE"]

// Example 2: Extract property from objects
List<Employee> employees = getEmployees();
List<String> employeeNames = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

// Example 3: Mathematical transformation
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> squared = numbers.stream()
    .map(n -> n * n)
    .collect(Collectors.toList());
// Output: [1, 4, 9, 16, 25]

// Example 4: Type conversion
List<String> numStrings = Arrays.asList("1", "2", "3");
List<Integer> nums = numStrings.stream()
    .map(Integer::parseInt)
    .collect(Collectors.toList());
```

**Common Use Case**: Extract data from complex objects.

### 3. sorted()

Orders stream elements.

```java
// Example 1: Natural order for primitives
List<Integer> numbers = Arrays.asList(5, 2, 8, 1, 9);
List<Integer> sorted = numbers.stream()
    .sorted()
    .collect(Collectors.toList());
// Output: [1, 2, 5, 8, 9]

// Example 2: Reverse order
List<Integer> descending = numbers.stream()
    .sorted((a, b) -> b.compareTo(a))
    .collect(Collectors.toList());
// Output: [9, 8, 5, 2, 1]

// Example 3: Sort objects by property
List<Employee> employees = getEmployees();
List<Employee> byName = employees.stream()
    .sorted(Comparator.comparing(Employee::getName))
    .collect(Collectors.toList());

// Example 4: Multi-level sorting
List<Employee> sorted = employees.stream()
    .sorted(Comparator.comparing(Employee::getDepartment)
                      .thenComparing(Employee::getSalary)
                      .reversed())
    .collect(Collectors.toList());

// Example 5: Null-safe sorting
List<Person> people = getPeople();
List<Person> sorted = people.stream()
    .sorted(Comparator.nullsLast(Comparator.comparing(Person::getCity)))
    .collect(Collectors.toList());
```

**Time Complexity**: O(n log n)  
**Note**: `sorted()` is **stateful** - it must examine all elements before producing any output.

### 4. distinct()

Removes duplicate elements.

```java
// Example 1: Remove duplicates from list
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 3, 4);
List<Integer> unique = numbers.stream()
    .distinct()
    .collect(Collectors.toList());
// Output: [1, 2, 3, 4]

// Example 2: Distinct with strings
List<String> words = Arrays.asList("apple", "app", "apple", "app");
List<String> unique = words.stream()
    .distinct()
    .collect(Collectors.toList());
// Output: ["apple", "app"]

// Example 3: Distinct objects (uses equals() method)
List<Person> people = getPeople();
List<Person> uniquePeople = people.stream()
    .distinct()  // Uses Person.equals()
    .collect(Collectors.toList());

// Example 4: Distinct by property (No direct method)
List<Person> uniqueByCity = people.stream()
    .collect(Collectors.toCollection(() -> 
        new TreeSet<>(Comparator.comparing(Person::getCity))
    ))
    .stream()
    .collect(Collectors.toList());
```

**Time Complexity**: O(n) - uses HashSet internally  
**Note**: For distinct by property, you typically need custom logic.

### 5. peek(Consumer)

Performs an action on each element without modifying the stream. Primarily for debugging.

```java
// Example 1: Debug stream processing
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> result = numbers.stream()
    .filter(n -> n % 2 == 0)
    .peek(n -> System.out.println("After filter: " + n))
    .map(n -> n * 2)
    .peek(n -> System.out.println("After map: " + n))
    .collect(Collectors.toList());

// Output:
// After filter: 2
// After map: 4
// After filter: 4
// After map: 8

// Example 2: Logging intermediate results
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
List<String> result = names.stream()
    .map(String::toUpperCase)
    .peek(n -> logger.debug("Processing: " + n))
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList());

// Example 3: Multiple peeks for tracing
List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5);
nums.stream()
    .peek(n -> System.out.println("1. Original: " + n))
    .filter(n -> n > 2)
    .peek(n -> System.out.println("2. After filter: " + n))
    .map(n -> n * 10)
    .peek(n -> System.out.println("3. After map: " + n))
    .forEach(n -> System.out.println("4. Final: " + n));
```

**⚠️ Important Warning**: `peek()` should NEVER be used for side-effects in production code - only for debugging!

```java
// ❌ BAD - Don't do this!
names.stream()
    .peek(name -> database.save(name))  // Side effect
    .collect(Collectors.toList());

// ✅ GOOD - Use forEach instead
names.stream()
    .forEach(name -> database.save(name));  // Clear intent
```

---

## 4. Intermediate Operations (Part 2: Advanced)

### 1. skip(long)

Skips the first n elements.

```java
// Example 1: Skip first N elements
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
List<Integer> skipped = numbers.stream()
    .skip(3)
    .collect(Collectors.toList());
// Output: [4, 5, 6, 7, 8, 9, 10]

// Example 2: Pagination (skip for offset)
List<Integer> page2 = numbers.stream()
    .skip(5)      // Skip first 5
    .limit(5)     // Take next 5
    .collect(Collectors.toList());
// Output: [6, 7, 8, 9, 10]

// Example 3: Real-world pagination
int pageSize = 10;
int pageNum = 2;  // 0-indexed
List<User> usersPage2 = allUsers.stream()
    .skip((long) pageNum * pageSize)
    .limit(pageSize)
    .collect(Collectors.toList());
```

**Time Complexity**: O(n) - must skip through n elements  
**Common Use**: Pagination and offset queries

### 2. limit(long)

Limits the stream to a maximum number of elements.

```java
// Example 1: Get first N elements
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
List<Integer> firstFive = numbers.stream()
    .limit(5)
    .collect(Collectors.toList());
// Output: [1, 2, 3, 4, 5]

// Example 2: Limit infinite streams
Stream<Integer> infinite = Stream.iterate(1, n -> n + 1);
List<Integer> first100 = infinite
    .limit(100)
    .collect(Collectors.toList());

// Example 3: Practical - Top N products by rating
List<Product> topProducts = products.stream()
    .sorted(Comparator.comparing(Product::getRating).reversed())
    .limit(10)
    .collect(Collectors.toList());

// Example 4: Short-circuit with limit
Stream<String> infinite = Stream.generate(() -> "x");
Stream<String> limited = infinite.limit(3);  // Doesn't block
List<String> result = limited.collect(Collectors.toList());
// Output: ["x", "x", "x"]
```

**Time Complexity**: O(n) where n is the limit  
**Note**: `limit()` is a short-circuiting operation - it can terminate early

### 3. takeWhile() and dropWhile() (Java 9+)

Take/drop elements while condition is true.

```java
// Example 1: takeWhile - Take elements while condition is met
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 3, 2, 1);
List<Integer> taken = numbers.stream()
    .takeWhile(n -> n < 4)
    .collect(Collectors.toList());
// Output: [1, 2, 3]

// Example 2: dropWhile - Skip elements while condition is met
List<Integer> dropped = numbers.stream()
    .dropWhile(n -> n < 4)
    .collect(Collectors.toList());
// Output: [4, 5, 3, 2, 1]

// Example 3: Real-world usage
List<Transaction> transactions = getTransactions();  // Sorted by date
List<Transaction> recentTransactions = transactions.stream()
    .dropWhile(t -> t.getDate().isBefore(startDate))
    .takeWhile(t -> t.getDate().isBefore(endDate))
    .collect(Collectors.toList());
```

---

## 5. Terminal Operations

### 1. collect(Collector)

Accumulates stream elements into a collection.

```java
// Example 1: Collect to List
List<String> names = stream
    .collect(Collectors.toList());

// Example 2: Collect to Set
Set<String> uniqueNames = stream
    .collect(Collectors.toSet());

// Example 3: Collect to Map
Map<Integer, String> idToName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,
        Employee::getName
    ));

// Example 4: Custom collector
String csv = names.stream()
    .collect(Collectors.joining(", "));
```

### 2. forEach(Consumer)

Performs an action on each element.

```java
// Example 1: Simple iteration
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
names.stream()
    .forEach(System.out::println);

// Example 2: With side effects
employees.stream()
    .filter(e -> e.getSalary() > 100000)
    .forEach(e -> database.updateBonus(e, 5000));

// Example 3: forEachOrdered (preserves order in parallel)
names.parallelStream()
    .forEachOrdered(System.out::println);
```

### 3. count()

Returns the number of elements.

```java
// Example 1: Count total elements
long total = numbers.stream().count();

// Example 2: Count with filter
long evenCount = numbers.stream()
    .filter(n -> n % 2 == 0)
    .count();

// Example 3: Conditional count
long highEarners = employees.stream()
    .filter(e -> e.getSalary() > 100000)
    .count();
```

### 4. findFirst() vs findAny()

```java
// Example 1: findFirst - Gets first element (deterministic)
Optional<String> first = names.stream()
    .filter(n -> n.startsWith("A"))
    .findFirst();

// Example 2: findAny - Gets any element (non-deterministic)
Optional<String> any = names.parallelStream()
    .filter(n -> n.startsWith("A"))
    .findAny();  // Better for parallel streams
```

**Key Difference**:
- `findFirst()`: Always returns first element in encounter order
- `findAny()`: Returns any element, better for parallel streams

### 5. anyMatch(), allMatch(), noneMatch()

```java
// Example 1: anyMatch - At least one matches
boolean hasAdult = people.stream()
    .anyMatch(p -> p.getAge() >= 18);

// Example 2: allMatch - All match
boolean allAdults = people.stream()
    .allMatch(p -> p.getAge() >= 18);

// Example 3: noneMatch - None match
boolean noMinors = people.stream()
    .noneMatch(p -> p.getAge() < 18);
```

**Short-Circuiting**: These operations can return early without processing all elements.

### 6. max() and min()

```java
// Example 1: Find maximum
Optional<Integer> max = numbers.stream()
    .max(Integer::compareTo);

// Example 2: Find minimum
Optional<Integer> min = numbers.stream()
    .min(Integer::compareTo);

// Example 3: With custom comparator
Optional<Employee> highestPaid = employees.stream()
    .max(Comparator.comparing(Employee::getSalary));
```

### 7. reduce()

Combines elements into a single result. (Covered in detail below)

---

## 6. Reduction Operations

### Understanding reduce()

Reduction combines stream elements into a single result using a binary operator.

```
reduce(BinaryOperator<T>) → Optional<T>
reduce(T identity, BinaryOperator<T>) → T
reduce(T identity, BiFunction, BinaryOperator) → R
```

### Example 1: Sum Using reduce()

```java
// Traditional way
int sum = 0;
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
for (Integer n : numbers) {
    sum += n;
}

// Using reduce - No identity
Optional<Integer> sum1 = numbers.stream()
    .reduce((a, b) -> a + b);
// Result: Optional[15]

// Using reduce - With identity
Integer sum2 = numbers.stream()
    .reduce(0, (a, b) -> a + b);
// Result: 15

// Using reduce - With method reference
Integer sum3 = numbers.stream()
    .reduce(0, Integer::sum);
// Result: 15
```

**Explanation:**
- First call: `(1, 2) → 3`
- Second call: `(3, 3) → 6`
- Third call: `(6, 4) → 10`
- Fourth call: `(10, 5) → 15`

### Example 2: String Concatenation

```java
// Without identity
List<String> words = Arrays.asList("Hello", " ", "World");
Optional<String> sentence1 = words.stream()
    .reduce((s1, s2) -> s1 + s2);
// Result: Optional["Hello World"]

// With identity
String sentence2 = words.stream()
    .reduce("", (s1, s2) -> s1 + s2);
// Result: "Hello World"

// Using StringJoiner (better for concatenation)
String sentence3 = words.stream()
    .collect(Collectors.joining(" "));
// Result: "Hello World"
```

### Example 3: Finding Maximum Value

```java
List<Integer> numbers = Arrays.asList(5, 2, 8, 1, 9);

// Without identity
Optional<Integer> max1 = numbers.stream()
    .reduce((a, b) -> a > b ? a : b);
// Result: Optional[9]

// Using Integer.max
Optional<Integer> max2 = numbers.stream()
    .reduce(Integer::max);
// Result: Optional[9]

// Equivalent to
Integer max3 = numbers.stream()
    .reduce(Integer.MIN_VALUE, Integer::max);
// Result: 9
```

### Example 4: Three-Argument reduce() - Complex Reduction

```java
// Convert List<Integer> to String representation
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

String result = numbers.stream()
    .reduce(
        "[",  // Identity
        (str, num) -> str + num + ", ",  // Accumulator
        (str1, str2) -> str1 + str2      // Combiner (for parallel)
    );
// Result: "[1, 2, 3, 4, 5, "

// Better example: Count and sum
class Summary {
    int count;
    long sum;
    
    Summary(int count, long sum) {
        this.count = count;
        this.sum = sum;
    }
}

Summary summary = numbers.stream()
    .reduce(
        new Summary(0, 0L),  // Identity
        (s, n) -> {  // Accumulator
            s.sum += n;
            s.count++;
            return s;
        },
        (s1, s2) -> {  // Combiner (for parallel streams)
            s1.sum += s2.sum;
            s1.count += s2.count;
            return s1;
        }
    );

double average = (double) summary.sum / summary.count;
```

### collect() - Mutable Reduction

Unlike `reduce()`, `collect()` uses a **mutable container** to accumulate results.

```java
// Traditional collect
List<String> words = Arrays.asList("apple", "banana", "cherry");

// To List
List<String> result1 = words.stream()
    .collect(Collectors.toList());

// To Set
Set<String> result2 = words.stream()
    .collect(Collectors.toSet());

// To String (joining)
String result3 = words.stream()
    .collect(Collectors.joining(", "));
// Result: "apple, banana, cherry"

// Custom collection
LinkedList<String> result4 = words.stream()
    .collect(Collectors.toCollection(LinkedList::new));
```

### Tricky Question: reduce() vs collect()

**Q: When should you use reduce() vs collect()?**

**Answer:**

| Aspect | reduce() | collect() |
|--------|----------|----------|
| **Purpose** | Immutable reduction | Mutable reduction |
| **Result** | Single value | Collection |
| **Efficiency** | Better for single value | Better for collections |
| **Parallelization** | Requires combiner logic | Handles parallel automatically |
| **Use Case** | Summing, finding max/min | Converting to List/Map/Set |

```java
// Use reduce() for single values
Integer sum = numbers.stream()
    .reduce(0, Integer::sum);

// Use collect() for collections
List<Integer> list = numbers.stream()
    .collect(Collectors.toList());

// DON'T mix purposes
// ❌ BAD: Using collect for a single value
List<Integer> sum = numbers.stream()
    .collect(Collectors.toList());  // Wrong!

// ✅ GOOD: Direct approach
Integer sum = numbers.stream()
    .reduce(0, Integer::sum);
```

---

## 7. Map vs FlatMap - Deep Dive

### Understanding the Difference

**map()**: 1-to-1 transformation  
**flatMap()**: 1-to-many transformation with flattening

### Visual Representation

```
Input: [[1,2], [3,4], [5,6]]

map():       [[1*10, 2*10], [3*10, 4*10], [5*10, 6*10]]
             [[10, 20], [30, 40], [50, 60]]

flatMap():   [10, 20, 30, 40, 50, 60]
             (Flattened single stream)
```

### Example 1: Simple map vs flatMap

```java
// map() - Transform list of lists
List<List<Integer>> listOfLists = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4),
    Arrays.asList(5, 6)
);

// Using map - creates Stream<List<Integer>>
List<List<Integer>> mapped = listOfLists.stream()
    .map(list -> list.stream()
        .map(x -> x * 2)
        .collect(Collectors.toList())
    )
    .collect(Collectors.toList());
// Output: [[2, 4], [6, 8], [10, 12]]

// Using flatMap - creates Stream<Integer>
List<Integer> flatMapped = listOfLists.stream()
    .flatMap(list -> list.stream()
        .map(x -> x * 2)
    )
    .collect(Collectors.toList());
// Output: [2, 4, 6, 8, 10, 12]
```

### Example 2: Practical - Split sentences into words

```java
// Requirement: Get all unique words from sentences
List<String> sentences = Arrays.asList(
    "Hello World",
    "Java is fun",
    "Hello Java"
);

// Using map (WRONG - nested structure)
List<Stream<String>> wrong = sentences.stream()
    .map(s -> Arrays.stream(s.split(" ")))
    .collect(Collectors.toList());
// Output: [Stream[Hello, World], Stream[Java, is, fun], ...]

// Using flatMap (CORRECT - flattened)
List<String> words = sentences.stream()
    .flatMap(s -> Arrays.stream(s.split(" ")))
    .collect(Collectors.toList());
// Output: [Hello, World, Java, is, fun, Hello, Java]

// With distinct
List<String> uniqueWords = sentences.stream()
    .flatMap(s -> Arrays.stream(s.split(" ")))
    .distinct()
    .collect(Collectors.toList());
// Output: [Hello, World, Java, is, fun]
```

### Example 3: flatMap with Optional

```java
// Scenario: Get user names whose addresses exist
class User {
    private String name;
    private Optional<Address> address;
    
    public Optional<Address> getAddress() {
        return address;
    }
}

List<User> users = getUsers();

// Without flatMap - verbose
List<User> withAddress = new ArrayList<>();
for (User u : users) {
    if (u.getAddress().isPresent()) {
        withAddress.add(u);
    }
}

// With flatMap - clean
List<String> names = users.stream()
    .flatMap(u -> u.getAddress().stream())  // Converts Optional to Stream
    .map(Address::getCity)
    .distinct()
    .collect(Collectors.toList());
```

### Example 4: One-to-Many Mapping

```java
// Scenario: Create all combinations of colors and sizes
List<String> colors = Arrays.asList("Red", "Blue", "Green");
List<String> sizes = Arrays.asList("S", "M", "L");

List<String> combinations = colors.stream()
    .flatMap(color -> sizes.stream()
        .map(size -> color + "-" + size)
    )
    .collect(Collectors.toList());

// Output: [Red-S, Red-M, Red-L, Blue-S, Blue-M, Blue-L, ...]
```

### Interview Question: Nested Stream Flattening

**Q: How would you flatten a deeply nested structure?**

```java
class Department {
    private String name;
    private List<Employee> employees;
    // getters
}

class Employee {
    private String name;
    private List<Project> projects;
    // getters
}

class Project {
    private String name;
}

List<Department> departments = getDepartments();

// Get all project names from all employees in all departments
List<String> allProjects = departments.stream()
    .flatMap(dept -> dept.getEmployees().stream())  // Flatten employees
    .flatMap(emp -> emp.getProjects().stream())     // Flatten projects
    .map(Project::getName)
    .distinct()
    .collect(Collectors.toList());
```

---

## 8. Collectors - Advanced Usage

### 1. Collectors.groupingBy()

Groups elements by a classification function.

```java
// Example 1: Group employees by department
List<Employee> employees = getEmployees();

Map<String, List<Employee>> byDepartment = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// Output: {IT: [emp1, emp2], HR: [emp3, emp4], ...}

// Example 2: Count employees by department
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));
// Output: {IT: 5, HR: 3, Finance: 4}

// Example 3: Average salary by department
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));
// Output: {IT: 85000.0, HR: 60000.0, ...}

// Example 4: Multiple level grouping
Map<String, Map<String, List<Employee>>> byDeptAndCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getCity)
    ));
// Output: {IT: {NYC: [emp1], LA: [emp2]}, ...}
```

### 2. Collectors.partitioningBy()

Partitions into two groups (true/false).

```java
// Example 1: Partition by condition
List<Employee> employees = getEmployees();

Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(
        e -> e.getSalary() > 80000
    ));

// Output: {true: [highEarners], false: [lowEarners]}

// Example 2: Get both high and low earners
List<Employee> highEarners = partitioned.get(true);
List<Employee> lowEarners = partitioned.get(false);

// Example 3: Partition with downstream collector
Map<Boolean, Long> countByHighSalary = employees.stream()
    .collect(Collectors.partitioningBy(
        e -> e.getSalary() > 80000,
        Collectors.counting()
    ));
// Output: {true: 10, false: 15}

// Example 4: Find senior and junior employees
Map<Boolean, List<Employee>> senior = employees.stream()
    .collect(Collectors.partitioningBy(
        e -> e.getYearsExperience() > 5
    ));
```

**Tricky Difference**: groupingBy() can create any number of groups, partitioningBy() creates exactly 2.

### 3. Collectors.toMap()

Transforms stream into a Map.

```java
// Example 1: Create map from objects
List<Employee> employees = getEmployees();

Map<Integer, String> idToName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,      // Key mapper
        Employee::getName     // Value mapper
    ));

// Example 2: Handle duplicate keys
Map<String, Employee> byEmail = employees.stream()
    .collect(Collectors.toMap(
        Employee::getEmail,
        Function.identity(),
        (existing, duplicate) -> existing  // Keep first
    ));

// Example 3: Custom map type
TreeMap<String, Employee> sortedMap = employees.stream()
    .collect(Collectors.toMap(
        Employee::getName,
        Function.identity(),
        (a, b) -> a,
        TreeMap::new  // Use TreeMap instead of HashMap
    ));

// Example 4: Complex value
Map<Integer, String> idToDetails = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,
        e -> e.getName() + " - " + e.getDepartment(),
        (a, b) -> a
    ));
```

### 4. Collectors.joining()

Concatenates strings.

```java
// Example 1: Simple join
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

String csv = names.stream()
    .collect(Collectors.joining(", "));
// Output: "Alice, Bob, Charlie"

// Example 2: With prefix and suffix
String formatted = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// Output: "[Alice, Bob, Charlie]"

// Example 3: Join employee names
List<Employee> employees = getEmployees();

String employeeList = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "Team: ", ""));
// Output: "Team: Alice, Bob, Charlie"

// Example 4: SQL IN clause
List<Integer> ids = Arrays.asList(1, 2, 3, 4, 5);

String sqlIn = ids.stream()
    .map(String::valueOf)
    .collect(Collectors.joining(",", "(", ")"));
// Output: "(1,2,3,4,5)"
```

### 5. Collectors.mapping()

Applies function to elements before collecting.

```java
// Example 1: Extract names before collecting to list
List<Employee> employees = getEmployees();

List<String> names = employees.stream()
    .collect(Collectors.mapping(
        Employee::getName,
        Collectors.toList()
    ));

// Example 2: Map within grouping
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));
// Output: {IT: [Alice, Bob], HR: [Charlie]}

// Example 3: Collect and transform
Set<String> upperNames = names.stream()
    .collect(Collectors.mapping(
        String::toUpperCase,
        Collectors.toSet()
    ));
```

---

## 9. Parallel Streams - Performance & Pitfalls

### When to Use Parallel Streams

```java
// Example 1: Large dataset with CPU-intensive operation
List<Integer> largeList = IntStream.rangeClosed(1, 1000000)
    .boxed()
    .collect(Collectors.toList());

// Sequential: Slower for large datasets
long sum1 = largeList.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .reduce(0L, Long::sum);

// Parallel: Faster for large datasets
long sum2 = largeList.parallelStream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .reduce(0L, (a, b) -> a + b);

// Example 2: Convert sequential to parallel
List<Integer> numbers = getNumbers();

List<Integer> result1 = numbers.stream()
    .parallel()
    .filter(n -> n > 100)
    .map(n -> n * 2)
    .sequential()  // Back to sequential if needed
    .collect(Collectors.toList());
```

### Parallel Stream Pitfalls

```java
// ❌ PITFALL 1: Shared mutable state
List<Integer> list = new ArrayList<>();
numbers.parallelStream()
    .forEach(n -> list.add(n));  // Race condition!

// ✅ CORRECT: Use collect instead
List<Integer> list = numbers.parallelStream()
    .collect(Collectors.toList());

// ❌ PITFALL 2: Expensive operations
List<String> urls = getUrls();
List<String> contents = urls.parallelStream()
    .map(url -> downloadContent(url))  // I/O bound, not CPU bound
    .collect(Collectors.toList());

// ✅ Use sequential for I/O operations
List<String> contents = urls.stream()
    .map(url -> downloadContent(url))
    .collect(Collectors.toList());

// ❌ PITFALL 3: Non-thread-safe operations
Map<String, Integer> map = new HashMap<>();
numbers.parallelStream()
    .forEach(n -> map.put(n.toString(), n));  // Not thread-safe!

// ✅ CORRECT: Use ConcurrentHashMap or collect
ConcurrentHashMap<String, Integer> map = numbers.parallelStream()
    .collect(Collectors.toMap(
        Object::toString,
        n -> n,
        (a, b) -> a,
        ConcurrentHashMap::new
    ));
```

### Performance Comparison

```java
// Benchmark: Which is faster?
List<Integer> numbers = IntStream.rangeClosed(1, 10_000_000)
    .boxed()
    .collect(Collectors.toList());

// Sequential approach
long start = System.nanoTime();
long result1 = numbers.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .reduce(0L, Long::sum);
long sequential = System.nanoTime() - start;

// Parallel approach
start = System.nanoTime();
long result2 = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .reduce(0L, Long::sum);
long parallel = System.nanoTime() - start;

System.out.println("Sequential: " + sequential + " ns");
System.out.println("Parallel: " + parallel + " ns");

// Result depends on:
// - Dataset size (> 10,000 usually beneficial)
// - Operation complexity (CPU-intensive vs simple)
// - Number of cores
// - Data structure (arrays > lists due to split characteristics)
```

### Encounter Order in Parallel Streams

```java
// Example: Encounter order affects results
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Sequential: Guaranteed order
numbers.stream()
    .forEach(System.out::println);
// Output: 1 2 3 4 5 (always)

// Parallel: Order not guaranteed
numbers.parallelStream()
    .forEach(System.out::println);
// Output: 3 1 5 2 4 (random)

// forEachOrdered: Preserves order in parallel
numbers.parallelStream()
    .forEachOrdered(System.out::println);
// Output: 1 2 3 4 5 (ordered)
```

**Rule of Thumb for Parallel Streams:**
- Dataset size: > 10,000 elements
- CPU-intensive: Yes (not I/O-bound)
- Stateless operations: Yes
- Thread-safe collections: Yes

---

## 10. Lazy Evaluation & Short-Circuiting

### Understanding Lazy Evaluation

Intermediate operations are lazy - they don't execute until a terminal operation is called.

```java
// Example 1: Nothing executes here
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
Stream<Integer> lazy = numbers.stream()
    .filter(n -> {
        System.out.println("Filter: " + n);
        return n > 2;
    })
    .map(n -> {
        System.out.println("Map: " + n);
        return n * 2;
    });
// No output!

// Now it executes
List<Integer> result = lazy.collect(Collectors.toList());
// Output:
// Filter: 1
// Filter: 2
// Filter: 3
// Map: 3
// Filter: 4
// Map: 4
// Filter: 5
// Map: 5

// Example 2: Order of operations
numbers.stream()
    .filter(n -> {
        System.out.println("Filter: " + n);
        return n > 2;
    })
    .map(n -> {
        System.out.println("Map: " + n);
        return n * 2;
    })
    .limit(2)
    .forEach(System.out::println);

// Output:
// Filter: 1
// Filter: 2
// Filter: 3
// Map: 3
// 6
// Filter: 4
// Map: 4
// 8
```

### Short-Circuiting Operations

Some terminal operations can return without processing all elements.

```java
// Example 1: anyMatch - Returns true as soon as found
boolean hasEven = numbers.stream()
    .filter(n -> {
        System.out.println("Checking: " + n);
        return n % 2 == 0;
    })
    .anyMatch(n -> true);
// Output: Checks until finding first even

// Example 2: findFirst - Stops after finding first
Optional<Integer> first = numbers.stream()
    .filter(n -> {
        System.out.println("Checking: " + n);
        return n > 2;
    })
    .findFirst();
// Output: Stops after finding first element > 2

// Example 3: limit with infinite stream
Stream<Integer> infinite = Stream.iterate(0, n -> n + 1);
List<Integer> first5 = infinite
    .limit(5)
    .collect(Collectors.toList());
// Output: [0, 1, 2, 3, 4]
```

**Key Benefits:**
1. **Performance**: Don't process unnecessary data
2. **Memory**: Process only what's needed
3. **Efficiency**: Especially useful with infinite streams

---

## 11. Stateless vs Stateful Operations

### Stateless Operations

Each element processed independently, no dependency on previous elements.

```java
// Stateless operations
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// filter - Each element checked independently
numbers.stream()
    .filter(n -> n > 2);

// map - Each element transformed independently
numbers.stream()
    .map(n -> n * 2);

// forEach - Each element processed independently
numbers.stream()
    .forEach(System.out::println);
```

**Characteristics:**
- No buffer needed
- Can parallelize easily
- Order doesn't matter
- Efficient for both sequential and parallel

### Stateful Operations

Must maintain state across elements (buffer/examine previous elements).

```java
// Stateful operations
List<Integer> numbers = Arrays.asList(5, 2, 8, 1, 9, 3);

// sorted() - Must examine all elements first
numbers.stream()
    .sorted();  // Requires buffering and sorting

// distinct() - Must track seen elements
numbers.stream()
    .distinct();  // Uses HashSet to track

// skip() - Must count elements
numbers.stream()
    .skip(2);  // Requires tracking count

// limit() - Must track count
numbers.stream()
    .limit(3);  // Requires count management
```

**Characteristics:**
- Requires buffering/state tracking
- More memory usage
- Harder to parallelize efficiently
- Order matters

### Tricky Question: Performance Implications

**Q: Why is `sorted()` slow on large datasets?**

**Answer:**
```java
// sorted() is stateful - it must:
// 1. Buffer ALL elements
// 2. Sort them using O(n log n) algorithm
// 3. Then stream results

// This means:
List<Integer> largeList = // 1 million elements

// Sequential sorted
largeList.stream()
    .sorted()  // Must hold 1M elements in memory!
    .limit(10)  // Only needs first 10
    .collect(Collectors.toList());

// Better approach: Use PriorityQueue
largeList.stream()
    .sorted()  // Still bufferes all
    .limit(10);

// Even better approach: Use a proper algorithm
List<Integer> topTen = largeList.stream()
    .sorted(Comparator.reverseOrder())
    .limit(10)
    .collect(Collectors.toList());
```

---

## 12. Non-Interfering & Immutability

### Non-Interfering Requirement

Behavior parameters (lambdas) must not interfere with the stream source.

```java
// ❌ BAD: Interfering operation
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
list.stream()
    .forEach(n -> list.add(n * 2));  // ConcurrentModificationException!

// ✅ CORRECT: No interference
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
List<Integer> result = list.stream()
    .map(n -> n * 2)
    .collect(Collectors.toList());

// ❌ BAD: Modifying captured variable
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
int[] counter = {0};
list.stream()
    .forEach(n -> counter[0]++);  // Side effect!

// ✅ CORRECT: Use count()
long count = list.stream().count();
```

### Stateless Lambdas

Lambdas must be stateless (no external state modification).

```java
// ❌ BAD: Stateful lambda
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
int[] sum = {0};

list.parallelStream()
    .forEach(n -> sum[0] += n);  // Not thread-safe!

// ✅ CORRECT: Use reduce
Integer sum = list.parallelStream()
    .reduce(0, Integer::sum);

// ❌ BAD: Sharing mutable object
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> results = new ArrayList<>();

list.parallelStream()
    .forEach(n -> results.add(n * 2));  // Race condition!

// ✅ CORRECT: Use collect
List<Integer> results = list.parallelStream()
    .map(n -> n * 2)
    .collect(Collectors.toList());
```

---

## 13. Encounter Order & Sequential vs Parallel

### Encounter Order

The order in which elements appear in the stream source.

```java
// Example 1: List maintains encounter order
List<Integer> list = Arrays.asList(3, 1, 4, 1, 5, 9);
list.stream()
    .forEach(System.out::println);
// Output: 3 1 4 1 5 9 (in order)

// Example 2: Set doesn't guarantee order
Set<Integer> set = new HashSet<>(list);
set.stream()
    .forEach(System.out::println);
// Output: Random order

// Example 3: TreeSet has defined order
Set<Integer> treeSet = new TreeSet<>(list);
treeSet.stream()
    .forEach(System.out::println);
// Output: 1 1 3 4 5 9 (sorted order)
```

### forEachOrdered vs forEach in Parallel Streams

```java
// Example 1: forEach doesn't guarantee order
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Diana");

System.out.println("forEach:");
names.parallelStream()
    .forEach(System.out::println);
// Output: Random order

System.out.println("forEachOrdered:");
names.parallelStream()
    .forEachOrdered(System.out::println);
// Output: Alice Bob Charlie Diana (ordered)
```

**Performance Trade-off:**
- `forEach`: Faster (no ordering constraint)
- `forEachOrdered`: Slower (must maintain order)

### Statistical Operations

```java
// IntStream statistics
IntStream stream = IntStream.rangeClosed(1, 10);
IntSummaryStatistics stats = stream.summaryStatistics();

System.out.println("Count: " + stats.getCount());      // 10
System.out.println("Sum: " + stats.getSum());          // 55
System.out.println("Min: " + stats.getMin());          // 1
System.out.println("Max: " + stats.getMax());          // 10
System.out.println("Average: " + stats.getAverage());  // 5.5
```

---

## 14. Debugging Streams with peek()

### Using peek() for Debugging

```java
// Example 1: Trace each stage
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

List<Integer> result = numbers.stream()
    .peek(n -> System.out.println("1. Original: " + n))
    .filter(n -> n > 2)
    .peek(n -> System.out.println("2. After filter: " + n))
    .map(n -> n * 2)
    .peek(n -> System.out.println("3. After map: " + n))
    .collect(Collectors.toList());

// Output:
// 1. Original: 1
// 1. Original: 2
// 1. Original: 3
// 2. After filter: 3
// 3. After map: 6
// 1. Original: 4
// 2. After filter: 4
// 3. After map: 8
// 1. Original: 5
// 2. After filter: 5
// 3. After map: 10

// Example 2: Debug with conditional logging
List<String> words = Arrays.asList("apple", "app", "application");

words.stream()
    .filter(w -> w.length() > 3)
    .peek(w -> {
        if (w.length() > 5) {
            System.out.println("Long word: " + w);
        }
    })
    .collect(Collectors.toList());
```

### Advanced Debugging

```java
// Example: Debugging with custom formatter
List<Employee> employees = getEmployees();

employees.stream()
    .filter(e -> e.getSalary() > 100000)
    .peek(e -> logger.debug(String.format(
        "Processing: %s ($%.2f)",
        e.getName(),
        e.getSalary()
    )))
    .map(e -> e.getName())
    .collect(Collectors.toList());
```

**⚠️ Important**: Never use peek() for side-effects in production!

---

## 15. Real-Time Interview Challenges

### Challenge 1: Find First Non-Repeated Character

```java
// Challenge: Find the first character that doesn't repeat
String input = "hello world";

Character result = input.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(
        Function.identity(),
        LinkedHashMap::new,  // Preserve insertion order
        Collectors.counting()
    ))
    .entrySet()
    .stream()
    .filter(e -> e.getValue() == 1)
    .map(Map.Entry::getKey)
    .findFirst()
    .orElse(null);

System.out.println(result);  // 'e'
```

### Challenge 2: Group Employees by Department and Calculate Average Salary

```java
// Challenge: Group employees by dept and get average salary
Map<String, Double> result = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

// Output: {IT: 85000.0, HR: 60000.0, Finance: 72000.0}
```

### Challenge 3: Convert List to Comma-Separated String

```java
// Challenge: Convert list to "item1, item2, item3"
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

String result = names.stream()
    .collect(Collectors.joining(", "));

System.out.println(result);  // "Alice, Bob, Charlie"
```

### Challenge 4: Find Top 3 Products by Sales

```java
// Challenge: Get top 3 best-selling products
List<Product> top3 = products.stream()
    .sorted(Comparator.comparing(Product::getSales).reversed())
    .limit(3)
    .collect(Collectors.toList());
```

### Challenge 5: Remove Duplicates Preserving Order

```java
// Challenge: Remove duplicates while maintaining order
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 3, 4);

List<Integer> unique = numbers.stream()
    .distinct()
    .collect(Collectors.toList());

// Or using Set.add() trick
List<Integer> unique2 = numbers.stream()
    .filter(new HashSet<>()::add)  // Returns false if already seen
    .collect(Collectors.toList());
```

### Challenge 6: Flatten List of Lists

```java
// Challenge: Flatten nested lists
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2, 3),
    Arrays.asList(4, 5),
    Arrays.asList(6, 7, 8, 9)
);

List<Integer> flattened = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());

// Output: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### Challenge 7: Partition into Two Lists

```java
// Challenge: Partition employees into high and low earners
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(
        e -> e.getSalary() > 80000
    ));

List<Employee> highEarners = partitioned.get(true);
List<Employee> lowEarners = partitioned.get(false);
```

### Challenge 8: Create Index-Value Map

```java
// Challenge: Create map of index → value
List<String> items = Arrays.asList("a", "b", "c", "d");

Map<Integer, String> indexed = IntStream.range(0, items.size())
    .boxed()
    .collect(Collectors.toMap(
        Function.identity(),
        items::get
    ));

// Output: {0=a, 1=b, 2=c, 3=d}
```

---

## 16. Tricky Questions & Common Mistakes

### Tricky Question 1: Can you reuse a stream?

**Q: What happens if you try to reuse a stream?**

```java
Stream<Integer> stream = Arrays.asList(1, 2, 3).stream();
long count1 = stream.count();
long count2 = stream.count();  // IllegalStateException!
```

**Answer**: 
```
IllegalStateException: stream has already been operated upon or closed

Reason: Streams are one-time use. Once a terminal operation is called,
the stream is consumed and cannot be used again.
```

**Solution**:
```java
List<Integer> list = Arrays.asList(1, 2, 3);
long count1 = list.stream().count();
long count2 = list.stream().count();  // Create new stream
```

### Tricky Question 2: What's the output of this code?

```java
List<Integer> nums = new ArrayList<>();
IntStream.range(0, 3)
    .peek(i -> System.out.println("Peek: " + i))
    .collect(ArrayList::new, (list, i) -> {
        System.out.println("Add: " + i);
        list.add(i);
    }, ArrayList::addAll);

System.out.println(nums);
```

**Expected Output**:
```
Peek: 0
Add: 0
Peek: 1
Add: 1
Peek: 2
Add: 2
[]
```

**Explanation**: The result is collected into a new ArrayList, not `nums`.

### Tricky Question 3: Stream doesn't modify original list

```java
List<String> names = new ArrayList<>(Arrays.asList("alice", "bob"));
List<String> result = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());

System.out.println(names);  // ["alice", "bob"]
System.out.println(result);  // ["ALICE", "BOB"]
```

**Key Point**: Streams are immutable - they don't modify the source.

### Tricky Question 4: NullPointerException with map()

```java
// ❌ This throws NullPointerException
List<String> names = Arrays.asList("Alice", null, "Charlie");
names.stream()
    .map(String::toUpperCase)  // NPE when null
    .collect(Collectors.toList());

// ✅ Correct: Filter nulls first
names.stream()
    .filter(Objects::nonNull)
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// ✅ Or use ofNullable
names.stream()
    .map(n -> n == null ? "UNKNOWN" : n.toUpperCase())
    .collect(Collectors.toList());
```

### Tricky Question 5: Distinct() doesn't compare by property

```java
// ❌ This doesn't remove duplicates by email
class Person {
    String name;
    String email;
}

List<Person> people = /* ... */;
people.stream()
    .distinct()  // Uses Object.equals(), needs custom implementation
    .collect(Collectors.toList());

// ✅ Correct: Use TreeSet with comparator
Set<Person> unique = people.stream()
    .collect(Collectors.toCollection(() -> 
        new TreeSet<>(Comparator.comparing(Person::getEmail))
    ));

List<Person> uniquePeople = new ArrayList<>(unique);
```

### Tricky Question 6: Reduce with no identity on empty stream

```java
// ❌ This throws NoSuchElementException
List<Integer> empty = Collections.emptyList();
Integer sum = empty.stream()
    .reduce(Integer::sum)  // Error!
    .get();

// ✅ Correct: Use identity
Integer sum = empty.stream()
    .reduce(0, Integer::sum);  // Returns 0

// ✅ Or handle Optional
Optional<Integer> sum = empty.stream()
    .reduce(Integer::sum);
System.out.println(sum.orElse(0));
```

### Tricky Question 7: Collectors.toMap() with duplicate keys

```java
// ❌ Exception: IllegalStateException - Duplicate key
List<Employee> employees = /* ... */;
Map<String, Employee> byEmail = employees.stream()
    .collect(Collectors.toMap(
        Employee::getEmail,
        Function.identity()
        // No merge function - error on duplicate!
    ));

// ✅ Correct: Provide merge function
Map<String, Employee> byEmail = employees.stream()
    .collect(Collectors.toMap(
        Employee::getEmail,
        Function.identity(),
        (existing, duplicate) -> existing  // Keep first
    ));
```

### Tricky Question 8: Parallel stream ordering with findAny()

```java
// Sequential: Always same result
Optional<Integer> result1 = numbers.stream()
    .findAny();  // Returns 1 every time

// Parallel: Non-deterministic
Optional<Integer> result2 = numbers.parallelStream()
    .findAny();  // May return different values
```

---

## 17. Performance Optimization Best Practices

### 1. Choose Right Data Structure

```java
// ❌ Bad: LinkedList for stream
LinkedList<Integer> list = new LinkedList<>();
list.stream()
    .filter(n -> n > 50)
    .collect(Collectors.toList());

// ✅ Good: ArrayList for stream
ArrayList<Integer> list = new ArrayList<>();
list.stream()
    .filter(n -> n > 50)
    .collect(Collectors.toList());

// Reason: Streams optimize for ArrayList (good split characteristics)
```

### 2. Order Operations Efficiently

```java
// ❌ Bad: Filter late
List<Employee> result = employees.stream()
    .map(e -> e.getName().toUpperCase())  // 1000 objects
    .map(n -> n.length())                 // Processing everything
    .filter(len -> len > 5)               // Filter at end
    .collect(Collectors.toList());

// ✅ Good: Filter early
List<Employee> result = employees.stream()
    .filter(e -> e.getName().length() > 5)  // Reduce dataset first
    .map(e -> e.getName().toUpperCase())
    .map(String::length)
    .collect(Collectors.toList());
```

### 3. Use Primitive Streams

```java
// ❌ Bad: Boxed stream
List<Integer> sum = IntStream.range(1, 1000000)
    .boxed()  // Wraps in Integer objects
    .reduce(0, Integer::sum);

// ✅ Good: Use IntStream
long sum = IntStream.range(1, 1000000)
    .sum();  // Direct operation on primitives

// Memory and speed comparison:
// IntStream: ~8 bytes per element
// Stream<Integer>: ~24+ bytes per element
```

### 4. Avoid Unnecessary Terminal Operations

```java
// ❌ Bad: Multiple terminal operations
boolean hasEven = numbers.stream()
    .filter(n -> n % 2 == 0)
    .count() > 0;

// ✅ Good: Use short-circuit operation
boolean hasEven = numbers.stream()
    .filter(n -> n % 2 == 0)
    .anyMatch(n -> true);
```

### 5. Limit with Sorted

```java
// ❌ Bad: Sort all then limit
List<Product> topProducts = products.stream()
    .sorted(Comparator.comparing(Product::getSales).reversed())
    .limit(10)  // Sorted 1 million before limiting to 10
    .collect(Collectors.toList());

// ✅ Good: Better approach for top-N
// Use a priority queue-based approach or sort after filtering
```

---

## 18. Interview Day Preparation Checklist

### One Week Before
- [ ] Review intermediate operations (map, filter, flatMap, etc.)
- [ ] Practice reduce() and collect() with examples
- [ ] Understand lazy evaluation and short-circuiting
- [ ] Study parallel streams and when to use them
- [ ] Review groupingBy, partitioningBy, toMap()

### Day Before
- [ ] Solve 5 Stream API challenges
- [ ] Review tricky questions section
- [ ] Understand non-interfering and stateless requirements
- [ ] Practice debugging with peek()
- [ ] Review performance optimization tips

### Interview Day Strategy
- [ ] Listen carefully to requirements
- [ ] Ask clarifying questions
- [ ] Think about edge cases (null, empty streams)
- [ ] Consider performance implications
- [ ] Explain your thought process

### Key Points to Mention in Interview
1. **Lazy Evaluation**: Mention intermediate operations are lazy
2. **Pipeline**: Explain source → intermediate → terminal
3. **Immutability**: Streams don't modify source
4. **Performance**: Discuss when to use parallel streams
5. **Non-Interfering**: Explain importance of stateless operations

### Common Mistakes to Avoid
- ❌ Using peek() for side effects in production
- ❌ Trying to reuse a stream
- ❌ Parallel streams for small datasets
- ❌ Interfering with stream source
- ❌ Not handling null values
- ❌ Ignoring exception handling

---

## Quick Reference: Terminal Operations

| Operation | Purpose | Returns | Example |
|-----------|---------|---------|---------|
| `collect()` | Accumulate into collection | Collector result | `collect(toList())` |
| `forEach()` | Iterate with side-effect | void | `forEach(System.out::println)` |
| `count()` | Count elements | long | `count()` |
| `max()` | Find maximum | Optional | `max(Integer::compare)` |
| `min()` | Find minimum | Optional | `min(Integer::compare)` |
| `reduce()` | Combine elements | Optional or value | `reduce(0, Integer::sum)` |
| `findFirst()` | Get first element | Optional | `findFirst()` |
| `findAny()` | Get any element | Optional | `findAny()` |
| `anyMatch()` | Check if any matches | boolean | `anyMatch(n -> n > 5)` |
| `allMatch()` | Check if all match | boolean | `allMatch(n -> n > 0)` |
| `noneMatch()` | Check if none match | boolean | `noneMatch(n -> n < 0)` |

---

## Mock Interview Questions

**Q1**: Explain map() vs flatMap() with real example  
**Q2**: When should you use parallel streams?  
**Q3**: What is lazy evaluation? Demonstrate with code.  
**Q4**: How do you handle null in streams?  
**Q5**: Explain reduce() with identity and without  
**Q6**: How would you debug a complex stream pipeline?  
**Q7**: What are stateless and stateful operations?  
**Q8**: How do you prevent ConcurrentModificationException?  
**Q9**: Compare groupingBy() and partitioningBy()  
**Q10**: Performance: distinct().sorted() vs sorted().distinct()

---

## Conclusion

The Java Stream API is a powerful functional programming tool that enables expressive, composable data processing. Success in interviews depends on:

1. **Deep Understanding**: Know why each operation exists
2. **Practical Application**: Apply to real-world problems
3. **Performance Awareness**: Know when to parallelize
4. **Common Pitfalls**: Avoid mistakes that senior developers know
5. **Communication**: Explain your thought process clearly

**Good luck with your interviews! 🚀**

---

**Document Version:** 1.0  
**Last Updated:** November 13, 2025  
**Target Interview:** December 2025 & Beyond

*This guide is based on 2025 interview trends and real questions from top companies. Keep practicing and you'll master the Stream API!*