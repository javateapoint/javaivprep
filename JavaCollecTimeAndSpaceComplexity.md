# Java Collections: Time & Space Complexity Cheat Sheet

A quick-reference chart showing common Java collections, their performance characteristics, memory overhead, and typical use cases—plus iteration patterns and when to pick each.

---

## 1. Collection Complexities & Use Cases

| Collection         | Access (get/peek) | Search/Contains | Insert End    | Insert Middle / Front | Remove End    | Remove Middle / Front | Iterate | Memory Overhead | Typical Use Case                                      |
|--------------------|-------------------|-----------------|---------------|-----------------------|---------------|-----------------------|---------|-----------------|-------------------------------------------------------|
| **ArrayList**      | O(1)              | O(n)            | amortized O(1)| O(n)                  | O(1) amort    | O(n)                  | O(n)    | Low (contiguous array) | Random-access reads, bulk iteration, index-based ops |
| **LinkedList**     | O(n)              | O(n)            | O(1)          | O(1) (at ends)        | O(1)          | O(1)                  | O(n)    | High (node + 2 pointers) | Frequent insert/delete at head/tail; deque semantics |
| **HashSet**        | n/a               | O(1) avg        | O(1) avg      | n/a                   | O(1) avg      | n/a                   | O(n)    | Moderate (buckets + load factor) | Unique elements, fast membership tests               |
| **TreeSet**        | n/a               | O(log n)        | O(log n)      | n/a                   | O(log n)      | n/a                   | O(n) sorted | Moderate            | Sorted set operations                                |
| **HashMap**        | n/a               | O(1) avg        | O(1) avg      | n/a                   | O(1) avg      | n/a                   | O(n) entries | Moderate       | Key→value lookup, fast put/get                       |
| **TreeMap**        | n/a               | O(log n)        | O(log n)      | n/a                   | O(log n)      | n/a                   | O(n) sorted | Moderate       | Sorted map, range queries                            |
| **ConcurrentHashMap** | n/a            | O(1) avg        | O(1) avg      | n/a                   | O(1) avg      | n/a                   | O(n)    | Moderate–High    | High‐throughput thread-safe key→value store           |
| **CopyOnWriteArrayList** | O(1)        | O(n)            | O(n)          | O(n)                  | O(n)          | O(n)                  | O(n)    | Very High (copy on write) | Read-heavy, infrequent writes; snapshot iteration     |

> n/a indicates the operation isn’t meaningful (e.g. “insert middle” on a Map).

---

## 2. Iteration Patterns & Performance

| Pattern                            | Applies To          | Complexity  | Notes & When to Use                                                |
|------------------------------------|---------------------|-------------|--------------------------------------------------------------------|
| **Index-based for-loop**           | RandomAccess Lists  | O(n)        | Fastest for `ArrayList`; avoids iterator overhead                  |
| for-each (`for(E e : coll)`)       | All Collections     | O(n)        | Readable; uses iterator under the hood                             |
| **Iterator**                       | All Collections     | O(n)        | Allows safe `remove()` during traversal                            |
| **while (it.hasNext())**           | All Collections     | O(n)        | Equivalent to iterator for-each; explicit loop control             |
| **Streams**                        | All Collections     | O(n)+overhead | Functional style; may incur boxing/unboxing, lambda costs          |
| **parallelStream()**               | SIZED, SUBSIZED     | O(n/p)+merge| Parallel processing; best with large data sets and CPU cores       |
| **Spliterator**                    | All Collections     | O(n)        | Custom split for parallel streams; control over characteristics    |

---

## 3. Memory Overhead Summary

- **Low**: `ArrayList`, primitive arrays  
- **Moderate**: `HashMap`/`HashSet` (bucket array + nodes), `TreeMap`  
- **High**: `LinkedList` (node object + two pointers), `CopyOnWriteArrayList` (full copy per write)

---

### How to Choose

1. **Need random‐access & bulk reads?** → `ArrayList` + index-based loop.  
2. **Frequent head/tail inserts?** → `LinkedList` or `Deque` + iterator.  
3. **Unique membership tests?** → `HashSet`/`ConcurrentHashMap`.  
4. **Sorted data & range queries?** → `TreeSet`/`TreeMap`.  
5. **Thread‐safe, high‐throughput map?** → `ConcurrentHashMap`.  
6. **Read‐heavy, write‐light with snapshots?** → `CopyOnWriteArrayList`.




# Interview Questions on Time & Space Complexity  
*For Senior Java Developers*  

Below are a set of questions, each with a code snippet. For each you’ll (a) derive the time complexity (Big‐O) and (b) derive the space complexity. Detailed answers follow each question.

---

## Question 1: De‐dup with Nested Loops

```java
public static <T> List<T> removeDuplicates(List<T> input) {
    List<T> result = new ArrayList<>();
    for (T item : input) {                        // (1)
        boolean seen = false;
        for (T prev : result) {                   // (2)
            if (prev.equals(item)) {
                seen = true;
                break;
            }
        }
        if (!seen) {
            result.add(item);                     // (3)
        }
    }
    return result;
}
```

- What is the time complexity in terms of n = input.size()?

- What is the space complexity in terms of n?


#### Time

- Outer loop: n iterations.

- Inner loop: on average checks half of result (growing from 0→n).

- Total ≈ n × (n/2) ⇒ O(n²).

#### Space

- result can grow to at most n elements ⇒ O(n) extra.

- Other locals are O(1).

- Space Complexity: O(n).



## Question 2: Using a HashSet
```java
public static <T> List<T> removeDuplicatesFast(List<T> input) {
    Set<T> seen = new HashSet<>();
    List<T> result = new ArrayList<>();
    for (T item : input) {
        if (seen.add(item)) {    // add returns false if already present
            result.add(item);
        }
    }
    return result;
}
```
- Time complexity?

- Space complexity?



#### Time

- seen.add(item) is O(1) on average.

- Loop is n iterations ⇒ O(n).

#### Space

- seen and result each hold at most n elements ⇒ O(n) total.

- Space Complexity: O(n).



## Question 3: Nested Collection Operations
```java
public static List<String> crossConcat(List<String> a, List<String> b) {
    List<String> out = new ArrayList<>();
    for (String x : a) {
        for (String y : b) {
            out.add(x + "-" + y);
        }
    }
    return out;
}
```

Given sizes m = a.size(), n = b.size():

- Time complexity?

- Space complexity (ignoring characters in strings)?



####  Time

Outer loop m, inner loop n ⇒ O(m·n) concatenations.

#### Space

out grows to m·n entries ⇒ O(m·n) extra references. 






## Question 4: Stream Pipeline Analysis
``` java
public static List<Integer> heavyProcess(List<Integer> data) {
    return data.stream()
               .filter(x -> x % 2 == 0)      // (A)
               .map(x -> compute(x))         // (B)
               .distinct()                   // (C)
               .sorted()                     // (D)
               .collect(Collectors.toList());
}
```

Assume compute(x) is O(1) and initial size is n. Estimate the time complexity for this pipeline.


- (A) filter: O(n)

- (B) map: O(n)

- (C) distinct: uses a HashSet under the hood ⇒ O(n) average

- (D) sorted: O(k·log k), where k is number of distinct evens (≤ n)

- Total ≈ O(n + k log k). In worst-case k = n: O(n log n).




# Time & Space Complexity Cheat Sheet

A fast-reference guide to analyzing algorithms in Java (or any language). Learn to eyeball code, apply patterns, and report Big-O/B-Ω/B-Θ bounds—plus pro tips to sharpen your complexity sense.

---

## 1. Core Concepts

- **Big-O (Worst-Case)**: Upper bound on operations as input _n_ grows.  
- **Big-Ω (Best-Case)**: Lower bound on operations.  
- **Big-Θ (Tight Bound)**: When upper and lower coincide.  

Discard constant factors and lower-order terms:

> O(3n + 5) → O(n)  
> O(n² + n log n) → O(n²)

---

## 2. Quick Steps to Analyze Code

1. **Identify “n”**: the size of your input (list length, string length, tree nodes, etc.).  
2. **Locate the “hot” operations**:
   - Simple statements: O(1).  
   - Loops → multiply.  
   - Nested loops → multiply levels.  
   - Consecutive blocks → add, then keep the dominant term.  
3. **Handle method calls**: substitute their complexity.  
4. **Drop constants & lower-order**: keep highest growth.  
5. **Report**: worst-case Big-O, and optionally Θ (if you know best-case).

---

## 3. Common Code Patterns

| Pattern                      | Time Complexity    | Notes                                  |
|------------------------------|--------------------|----------------------------------------|
| Single `for`/`while` loop    | O(n)               | n = loop iterations                    |
| Nested 2-level loops         | O(n²)              | i from 1→n, j from 1→n                 |
| Loop with half-range         | O(n)               | e.g. `for(i=0; i<n/2; i++)`            |
| Consecutive loops            | O(n + m) → O(n)    | if m ∼ n, merge → O(n)                 |
| Divide & conquer (binary)    | O(log n)           | e.g. binary search, balanced tree ops  |
| Merge sort / QuickSort       | O(n log n)         | typical external sort                  |
| Hash lookup & insert         | O(1) average       | HashMap/HashSet                        |
| Recursion with two calls     | O(2ⁿ)              | naive Fibonacci                        |
| Recursive T(n) = T(n/2) + O(1) | O(log n)         | binary chop                            |
| Master Theorem (aT(n/b)+f(n))| depends on f(n)    | use case divisions: n^(log_b a) vs f(n) |

---

## 4. Space Complexity

- **Auxiliary space**: extra memory beyond inputs.  
- **Total space**: inputs + auxiliary.  

Calculate:
1. Primitive variables: O(1) each.  
2. Collections: O(n) for arrays, lists, maps sized by n.  
3. Recursion stack: depth × per-call locals.  

Example:
```java
int fib(int n) {
  if (n < 2) return n;        // O(1) space
  return fib(n-1) + fib(n-2); // recursion stack depth ≈ n → O(n) space
}
```

## 5. Pro Tips & Tricks
### Amortized Analysis:

- ArrayList.add() is O(1) amortized (occasional resize O(n)).

### Early Exits:

- break or return can reduce average case but worst-case remains.

### Two-Pointer / Sliding Window:

- Often O(n) instead of O(n²) for subarray sums, matching pairs.

### Master Theorem for recurrences:

- Recognize T(n) = a T(n/b) + O(nᵈ) patterns.

### Hash vs Tree:

- HashMap/HashSet: O(1) avg, O(n) worst; TreeMap/TreeSet: O(log n) guaranteed.


