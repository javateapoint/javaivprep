# Time and Space Complexity: A Complete Deep Dive

## Introduction - The Big Picture

### What Do Time and Space Complexity Mean?

Imagine you're organizing a library. You could arrange books alphabetically, by color, by size, or randomly. Each method takes a different amount of time and uses different amounts of space. Time and space complexity are simply ways to measure how efficient your algorithms are.

**Time Complexity** measures how the number of operations grows as your input size increases. It answers the question: "If I double the input, does the algorithm take twice as long, or way longer?"

**Space Complexity** measures how much memory your algorithm needs as the input grows. It answers: "If I double the input, do I need twice the memory, or way more?"

### Why They Matter in Real-World Software

Consider Netflix's recommendation engine processing millions of users. If an algorithm with O(n²) complexity worked fine for 100 users, it would grind to a halt with 1 million users. Understanding complexity helps you:

- **Build scalable systems** that work for 10 users and 10 million users
- **Optimize performance** by choosing the right algorithm
- **Predict costs** in cloud computing where memory and CPU time cost money
- **Avoid production disasters** where slow algorithms crash your system under load

### Why Interviewers Care

Tech companies ask about complexity because it reveals:

1. **Problem-solving ability**: Can you think about efficiency, not just correctness?
2. **Scalability mindset**: Do you consider how code performs at scale?
3. **Trade-off awareness**: Can you balance speed vs memory?
4. **Mathematical thinking**: Can you analyze and reason about code?

### Real-Life Analogy

Think of time complexity like cooking recipes:

- **O(1)**: Taking one cookie from a jar. Doesn't matter if there are 5 or 500 cookies, it takes the same time.
- **O(n)**: Counting all cookies in the jar. If you have twice as many cookies, it takes twice as long.
- **O(n²)**: Comparing every cookie with every other cookie to find matches. Double the cookies means four times the work.
- **O(log n)**: Finding a word in a dictionary by repeatedly opening to the middle and eliminating half. Even with thousands of pages, you find it in about 10-15 steps.

---

## What is Time Complexity?

### Definition in Plain English

Time complexity describes how the runtime of an algorithm grows relative to the size of the input. It's not about exact seconds or milliseconds, but about the pattern of growth.

If your algorithm takes 5 operations for 10 items, does it take 10 operations for 20 items (linear), or 100 operations (exponential)? That's what time complexity tells you.

### What We Actually Measure

We count fundamental operations:

- Comparisons (`if (a > b)`)
- Arithmetic operations (`sum = a + b`)
- Array accesses (`arr[i]`)
- Assignment statements (`x = 5`)
- Function calls

We ignore constants and focus on the dominant term as input grows infinitely large.

### Why Execution Time ≠ Time Complexity

**Execution time** depends on:
- Your computer's processor speed
- Programming language
- Compiler optimizations
- Current system load
- Input data specifics

**Time complexity** is independent of all these factors. It's a mathematical abstraction that describes the algorithm's fundamental efficiency.

Example: A O(n²) algorithm might run faster than an O(n) algorithm for small inputs on a fast computer, but O(n) always wins for large inputs regardless of hardware.

### Best, Average, and Worst Case Explained

Consider searching for a number in an unsorted array:

**Best Case**: The number is the first element. You find it immediately. Time: O(1)

**Average Case**: The number is somewhere in the middle. You check about half the array. Time: O(n/2) = O(n)

**Worst Case**: The number is the last element or not in the array. You check everything. Time: O(n)

In interviews and real-world analysis, we almost always focus on the **worst case** because:
- It guarantees performance bounds
- It's easier to analyze
- Systems must handle worst-case scenarios without crashing

---

## Big-O Notation - The Core Concept

### What Big-O Represents

Big-O notation describes the **upper bound** of an algorithm's growth rate. The "O" stands for "Order of magnitude."

When we write O(n), we're saying: "In the worst case, this algorithm's runtime grows proportionally to n."

Big-O gives us a simplified, standardized way to communicate about algorithm efficiency without getting lost in implementation details.

### Why Constants and Lower Terms Are Ignored

Consider this algorithm:

```java
// 3n + 5 operations
for (int i = 0; i < n; i++) {      // n iterations
    System.out.println(arr[i]);     // 2 operations per iteration
}
System.out.println("Done");         // 5 operations
```

Technically, this does 3n + 5 operations. But as n grows:
- For n = 10: 3(10) + 5 = 35
- For n = 1000: 3(1000) + 5 = 3005
- For n = 1,000,000: 3(1,000,000) + 5 = 3,000,005

The constant 5 becomes insignificant. The coefficient 3 also doesn't change the growth pattern. So we say this is **O(n)**.

**Rule**: Drop constants and keep only the fastest-growing term.
- O(2n) → O(n)
- O(500) → O(1)
- O(n² + n) → O(n²)
- O(n + log n) → O(n)

### Growth Rate Visualization in Words

Imagine processing an input of size n = 1,000:

- **O(1)**: 1 operation. Instant.
- **O(log n)**: ~10 operations. Nearly instant.
- **O(n)**: 1,000 operations. Very fast.
- **O(n log n)**: ~10,000 operations. Fast.
- **O(n²)**: 1,000,000 operations. Slow.
- **O(2ⁿ)**: Forget about it. The universe will end first.

### Common Time Complexities Explained

#### O(1) - Constant Time

**Definition**: Runtime doesn't depend on input size. Always takes the same time.

**Real-world examples**:
- Accessing an array element by index: `arr[5]`
- Checking if a number is even or odd
- Pushing to a stack
- Getting the size of an ArrayList

**Think of it like**: Opening a specific page in a book when you know the page number.

#### O(log n) - Logarithmic Time

**Definition**: Runtime grows logarithmically. Each step eliminates a significant portion of the remaining data (usually half).

**Real-world examples**:
- Binary search in a sorted array
- Searching in a balanced binary search tree
- Finding an element in a sorted rotated array

**Think of it like**: Finding a word in a dictionary. You don't check every page; you keep dividing the search space in half.

**Why it's efficient**: Even for 1 billion elements, you need only about 30 operations.

#### O(n) - Linear Time

**Definition**: Runtime grows directly proportional to input size. Double the input, double the time.

**Real-world examples**:
- Finding the maximum element in an unsorted array
- Summing all elements
- Linear search
- Traversing a linked list

**Think of it like**: Counting people standing in a line. You must check each person once.

#### O(n log n) - Linearithmic Time

**Definition**: Faster than quadratic but slower than linear. Common in efficient sorting algorithms.

**Real-world examples**:
- Merge Sort
- Quick Sort (average case)
- Heap Sort
- Sorting then searching

**Think of it like**: Organizing a large deck of cards by repeatedly dividing and conquering.

**Why this complexity**: You divide the problem (log n) and do linear work (n) at each division level.

#### O(n²) - Quadratic Time

**Definition**: Runtime grows with the square of input size. Double the input, quadruple the time.

**Real-world examples**:
- Bubble Sort
- Selection Sort
- Insertion Sort
- Checking all pairs in an array
- Nested loops over the same collection

**Think of it like**: Comparing every person in a room with every other person for handshakes. For 10 people, that's 45 handshakes. For 100 people, it's 4,950 handshakes.

#### O(2ⁿ) - Exponential Time

**Definition**: Runtime doubles with each additional input element. Extremely slow.

**Real-world examples**:
- Recursive Fibonacci without memoization
- Solving the Towers of Hanoi
- Generating all subsets of a set
- Brute-force password cracking

**Think of it like**: A chain letter where each person sends to 2 people. After 30 levels, over a billion people are involved.

**Reality check**: Even for n = 50, this is usually impossible to compute in reasonable time.

---

## Time Complexity with Java Examples

### Example 1: O(1) - Constant Time Lookup

```java
public class ConstantTimeExample {
    public static int getFirstElement(int[] array) {
        // Accessing array by index is O(1)
        return array[0];
    }
    
    public static int calculateSum(int a, int b) {
        // Simple arithmetic is O(1)
        return a + b;
    }
    
    public static void main(String[] args) {
        int[] numbers = {10, 20, 30, 40, 50};
        
        System.out.println("First element: " + getFirstElement(numbers));
        System.out.println("Sum: " + calculateSum(100, 200));
    }
}
```

**Output:**
```
First element: 10
Sum: 300
```

**Line-by-Line Explanation:**
- `return array[0]`: Direct memory access using index. Always 1 operation regardless of array size.
- `return a + b`: Single arithmetic operation. Always 1 operation.

**Time Complexity Breakdown:**
- Both operations are O(1)
- Whether array has 5 elements or 5 million, accessing index 0 takes the same time
- No loops, no recursion, no input-dependent operations

---

### Example 2: O(n) - Single Loop

```java
public class LinearTimeExample {
    // Find maximum element in array
    public static int findMax(int[] array) {
        if (array.length == 0) {
            throw new IllegalArgumentException("Array is empty");
        }
        
        int max = array[0];                    // O(1) - constant operation
        
        for (int i = 1; i < array.length; i++) { // Loop runs n-1 times
            if (array[i] > max) {              // O(1) - constant comparison
                max = array[i];                 // O(1) - constant assignment
            }
        }
        
        return max;                             // O(1) - constant operation
    }
    
    public static void main(String[] args) {
        int[] numbers = {23, 45, 12, 67, 34, 89, 11};
        
        System.out.println("Maximum element: " + findMax(numbers));
    }
}
```

**Output:**
```
Maximum element: 89
```

**Line-by-Line Explanation:**
1. `int max = array[0]`: Initialize with first element. O(1)
2. `for (int i = 1; i < array.length; i++)`: Loop executes (n-1) times where n is array length
3. `if (array[i] > max)`: Each comparison is O(1), but happens n-1 times
4. `max = array[i]`: Assignment is O(1), but happens at most n-1 times

**Time Complexity Breakdown:**
- Initialization: O(1)
- Loop body: O(1) operations × (n-1) iterations = O(n-1)
- Total: O(1) + O(n-1) = O(n)

For n = 7, we do approximately 7 operations. For n = 7000, we do approximately 7000 operations.

---

### Example 3: O(n²) - Nested Loops

```java
public class QuadraticTimeExample {
    // Check if array contains duplicates
    public static boolean hasDuplicates(int[] array) {
        int n = array.length;
        
        // Outer loop: runs n times
        for (int i = 0; i < n; i++) {
            
            // Inner loop: runs n times for each outer iteration
            for (int j = i + 1; j < n; j++) {
                
                if (array[i] == array[j]) {    // O(1) comparison
                    return true;                // Found duplicate
                }
            }
        }
        
        return false;  // No duplicates found
    }
    
    public static void main(String[] args) {
        int[] numbers1 = {1, 2, 3, 4, 5};
        int[] numbers2 = {1, 2, 3, 2, 5};
        
        System.out.println("Array 1 has duplicates: " + hasDuplicates(numbers1));
        System.out.println("Array 2 has duplicates: " + hasDuplicates(numbers2));
    }
}
```

**Output:**
```
Array 1 has duplicates: false
Array 2 has duplicates: true
```

**Line-by-Line Explanation:**
1. Outer loop runs from i = 0 to n-1: n iterations
2. Inner loop runs from j = i+1 to n-1: decreases each time
   - When i=0, inner loop runs (n-1) times
   - When i=1, inner loop runs (n-2) times
   - When i=n-2, inner loop runs 1 time
3. Total comparisons: (n-1) + (n-2) + ... + 1 = n(n-1)/2

**Time Complexity Breakdown:**
- Number of comparisons: n(n-1)/2
- Expanding: (n² - n)/2
- Simplifying: O(n²/2 - n/2)
- Dropping constants and lower terms: O(n²)

For n = 5, roughly 10 operations. For n = 100, roughly 5000 operations. For n = 1000, roughly 500,000 operations.

---

### Example 4: O(log n) - Loop with Halving

```java
public class LogarithmicTimeExample {
    // Binary search in sorted array
    public static int binarySearch(int[] array, int target) {
        int left = 0;
        int right = array.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;  // Avoid overflow
            
            if (array[mid] == target) {
                return mid;                        // Found target
            } else if (array[mid] < target) {
                left = mid + 1;                    // Search right half
            } else {
                right = mid - 1;                   // Search left half
            }
        }
        
        return -1;  // Target not found
    }
    
    // Alternative: counting how many times we can divide n by 2
    public static int countDivisions(int n) {
        int count = 0;
        
        while (n > 1) {
            n = n / 2;      // Halve n each iteration
            count++;
            System.out.println("After division " + count + ": n = " + n);
        }
        
        return count;
    }
    
    public static void main(String[] args) {
        int[] sortedArray = {2, 5, 8, 12, 16, 23, 38, 45, 56, 67, 78};
        int target = 23;
        
        int index = binarySearch(sortedArray, target);
        System.out.println("Target " + target + " found at index: " + index);
        
        System.out.println("\nDividing 32 by 2 repeatedly:");
        int divisions = countDivisions(32);
        System.out.println("Total divisions: " + divisions);
    }
}
```

**Output:**
```
Target 23 found at index: 5

Dividing 32 by 2 repeatedly:
After division 1: n = 16
After division 2: n = 8
After division 3: n = 4
After division 4: n = 2
After division 5: n = 1
Total divisions: 5
```

**Line-by-Line Explanation:**
1. Calculate mid-point of current search range
2. If target found, return immediately
3. If target is larger, eliminate left half by moving left pointer
4. If target is smaller, eliminate right half by moving right pointer
5. Each iteration halves the search space

**Time Complexity Breakdown:**
- Start with n elements
- After 1st iteration: n/2 elements
- After 2nd iteration: n/4 elements
- After 3rd iteration: n/8 elements
- After k iterations: n/2^k elements
- Stop when n/2^k = 1, which means k = log₂(n)
- Therefore, time complexity is O(log n)

For n = 1024, only 10 iterations needed. For n = 1,048,576 (1 million), only 20 iterations needed!

---

### Example 5: O(2ⁿ) - Recursive Example (Fibonacci)

```java
public class ExponentialTimeExample {
    // Recursive Fibonacci - O(2^n) - VERY SLOW
    public static int fibonacciRecursive(int n) {
        if (n <= 1) {
            return n;                          // Base case
        }
        
        return fibonacciRecursive(n - 1) +     // Recursive call 1
               fibonacciRecursive(n - 2);      // Recursive call 2
    }
    
    // For comparison: Iterative Fibonacci - O(n) - FAST
    public static int fibonacciIterative(int n) {
        if (n <= 1) {
            return n;
        }
        
        int prev = 0, curr = 1;
        
        for (int i = 2; i <= n; i++) {
            int next = prev + curr;
            prev = curr;
            curr = next;
        }
        
        return curr;
    }
    
    public static void main(String[] args) {
        System.out.println("Fibonacci numbers using recursion:");
        for (int i = 0; i <= 10; i++) {
            System.out.print(fibonacciRecursive(i) + " ");
        }
        
        System.out.println("\n\nFibonacci numbers using iteration:");
        for (int i = 0; i <= 10; i++) {
            System.out.print(fibonacciIterative(i) + " ");
        }
        
        // Demonstrate the performance difference
        System.out.println("\n\nPerformance test (n=40):");
        
        long startRecursive = System.currentTimeMillis();
        int resultRecursive = fibonacciRecursive(40);
        long endRecursive = System.currentTimeMillis();
        
        long startIterative = System.currentTimeMillis();
        int resultIterative = fibonacciIterative(40);
        long endIterative = System.currentTimeMillis();
        
        System.out.println("Recursive result: " + resultRecursive + 
                           " (Time: " + (endRecursive - startRecursive) + "ms)");
        System.out.println("Iterative result: " + resultIterative + 
                           " (Time: " + (endIterative - startIterative) + "ms)");
    }
}
```

**Output:**
```
Fibonacci numbers using recursion:
0 1 1 2 3 5 8 13 21 34 55 

Fibonacci numbers using iteration:
0 1 1 2 3 5 8 13 21 34 55 

Performance test (n=40):
Recursive result: 102334155 (Time: 1205ms)
Iterative result: 102334155 (Time: 0ms)
```

**Line-by-Line Explanation (Recursive):**
- Each call to `fibonacciRecursive(n)` makes two more calls: `fib(n-1)` and `fib(n-2)`
- This creates a binary tree of calls
- For fib(5), the tree has these calls:
  ```
           fib(5)
         /        \
      fib(4)      fib(3)
      /    \      /    \
   fib(3) fib(2) fib(2) fib(1)
   /   \   /  \   /  \
  ...  ...  ... ... ... ...
  ```

**Time Complexity Breakdown:**
- Each node in the call tree represents one function call
- The tree has depth n
- Each level has roughly twice as many nodes as the previous level
- Total nodes ≈ 2^0 + 2^1 + 2^2 + ... + 2^n ≈ 2^(n+1) - 1
- Therefore, O(2^n)

For n = 10, about 1,024 calls. For n = 20, about 1,048,576 calls. For n = 40, over 1 trillion calls!

---

## What is Space Complexity?

### Definition in Simple Terms

Space complexity measures the total amount of memory an algorithm needs to complete its execution, as a function of the input size.

Just like time complexity doesn't measure seconds, space complexity doesn't measure bytes or megabytes. It measures the growth pattern of memory usage.

### Input Space vs Auxiliary Space

**Input Space**: Memory needed to store the input itself.
- Example: An array of n integers takes O(n) space

**Auxiliary Space**: Extra memory used by the algorithm beyond the input.
- Example: Variables, temporary arrays, recursion call stack

When we say "space complexity" in interviews, we usually mean **auxiliary space** unless specified otherwise.

```java
public int sum(int[] arr) {
    int total = 0;           // O(1) auxiliary space
    for (int num : arr) {    // arr is input space, not auxiliary
        total += num;
    }
    return total;
}
// Space complexity: O(1) auxiliary space
// Total space: O(n) [input] + O(1) [auxiliary] = O(n)
```

### Stack Memory vs Heap Memory

**Stack Memory**:
- Stores local variables and function call information
- Fixed size per function call
- Automatically managed
- Used for recursion tracking

**Heap Memory**:
- Stores dynamically allocated objects and arrays
- Can grow as needed
- Manually managed (though Java has garbage collection)
- Used for `new` keyword allocations

```java
public void example(int n) {
    int x = 5;              // Stack: local variable
    int[] arr = new int[n]; // Heap: array object; Stack: arr reference
    
    // x and arr reference are on stack
    // actual array data is on heap
}
```

### Why Recursion Consumes Extra Space

Each recursive call adds a new frame to the call stack containing:
- Function parameters
- Local variables
- Return address

These frames stack up until the base case is reached, then unwind as functions return.

```java
public int factorial(int n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}

// For factorial(5), call stack grows:
// factorial(5) -> stores n=5
// factorial(4) -> stores n=4
// factorial(3) -> stores n=3
// factorial(2) -> stores n=2
// factorial(1) -> stores n=1
// factorial(0) -> base case
// Space complexity: O(n) due to n stack frames
```

---

## Space Complexity with Java Examples

### Example 1: O(1) - Constant Space (Primitive Variables)

```java
public class ConstantSpaceExample {
    // Swap two numbers without extra array
    public static void swap(int a, int b) {
        System.out.println("Before swap: a = " + a + ", b = " + b);
        
        // Using only primitive variables - O(1) space
        int temp = a;    // One extra variable
        a = b;
        b = temp;
        
        System.out.println("After swap: a = " + a + ", b = " + b);
    }
    
    // Check if number is even
    public static boolean isEven(int n) {
        return n % 2 == 0;   // No extra space needed
    }
    
    // Find sum of digits
    public static int sumOfDigits(int n) {
        int sum = 0;         // Single variable - O(1)
        
        while (n > 0) {
            sum += n % 10;
            n /= 10;
        }
        
        return sum;
    }
    
    public static void main(String[] args) {
        swap(10, 20);
        System.out.println("\n15 is even: " + isEven(15));
        System.out.println("Sum of digits of 12345: " + sumOfDigits(12345));
    }
}
```

**Output:**
```
Before swap: a = 10, b = 20
After swap: a = 20, b = 10

15 is even: false
Sum of digits of 12345: 15
```

**Space Complexity Analysis:**
- `swap()`: Uses 1 temporary variable regardless of input values. O(1)
- `isEven()`: No extra variables. O(1)
- `sumOfDigits()`: Uses 1 variable `sum` regardless of input size. O(1)

**Key Point**: Number of extra variables doesn't depend on input size.

---

### Example 2: O(n) - Linear Space (Arrays)

```java
public class LinearSpaceExample {
    // Create a copy of array - O(n) space
    public static int[] copyArray(int[] original) {
        int n = original.length;
        int[] copy = new int[n];     // New array of size n
        
        for (int i = 0; i < n; i++) {
            copy[i] = original[i];
        }
        
        return copy;
    }
    
    // Store running sums - O(n) space
    public static int[] cumulativeSum(int[] arr) {
        int n = arr.length;
        int[] cumSum = new int[n];   // New array of size n
        
        cumSum[0] = arr[0];
        for (int i = 1; i < n; i++) {
            cumSum[i] = cumSum[i-1] + arr[i];
        }
        
        return cumSum;
    }
    
    public static void main(String[] args) {
        int[] original = {1, 2, 3, 4, 5};
        
        int[] copied = copyArray(original);
        System.out.println("Original: " + java.util.Arrays.toString(original));
        System.out.println("Copied: " + java.util.Arrays.toString(copied));
        
        int[] cumulative = cumulativeSum(original);
        System.out.println("Cumulative sum: " + java.util.Arrays.toString(cumulative));
    }
}
```

**Output:**
```
Original: [1, 2, 3, 4, 5]
Copied: [1, 2, 3, 4, 5]
Cumulative sum: [1, 3, 6, 10, 15]
```

**Space Complexity Analysis:**
- `copyArray()`: Creates new array of size n. O(n) space.
- `cumulativeSum()`: Creates new array of size n. O(n) space.
- Both functions allocate memory proportional to input size.

**Memory Breakdown:**
- Input array: n integers (not counted as auxiliary space)
- Output array: n integers (auxiliary space)
- Loop variables: constant (negligible)
- Total auxiliary space: O(n)

---

### Example 3: O(n) - Recursive Call Stack

```java
public class RecursiveSpaceExample {
    // Recursive factorial - O(n) space due to call stack
    public static int factorialRecursive(int n) {
        System.out.println("factorialRecursive(" + n + ") called");
        
        if (n == 0 || n == 1) {
            return 1;
        }
        
        return n * factorialRecursive(n - 1);
    }
    
    // Recursive sum of array - O(n) space
    public static int sumRecursive(int[] arr, int index) {
        if (index >= arr.length) {
            return 0;
        }
        
        return arr[index] + sumRecursive(arr, index + 1);
    }
    
    // Recursive print in reverse - O(n) space
    public static void printReverse(int[] arr, int index) {
        if (index < 0) {
            return;
        }
        
        System.out.print(arr[index] + " ");
        printReverse(arr, index - 1);
    }
    
    public static void main(String[] args) {
        System.out.println("Factorial of 5:");
        int result = factorialRecursive(5);
        System.out.println("Result: " + result);
        
        System.out.println("\nSum of array:");
        int[] numbers = {1, 2, 3, 4, 5};
        System.out.println("Sum: " + sumRecursive(numbers, 0));
        
        System.out.println("\nArray in reverse:");
        printReverse(numbers, numbers.length - 1);
    }
}
```

**Output:**
```
Factorial of 5:
factorialRecursive(5) called
factorialRecursive(4) called
factorialRecursive(3) called
factorialRecursive(2) called
factorialRecursive(1) called
Result: 120

Sum of array:
Sum: 15

Array in reverse:
5 4 3 2 1
```

**Space Complexity Analysis:**

For `factorialRecursive(5)`, the call stack grows:
```
Stack Frame 5: factorialRecursive(5) - stores n=5
Stack Frame 4: factorialRecursive(4) - stores n=4
Stack Frame 3: factorialRecursive(3) - stores n=3
Stack Frame 2: factorialRecursive(2) - stores n=2
Stack Frame 1: factorialRecursive(1) - base case
```

- Maximum depth of recursion: n
- Each frame takes constant space: O(1)
- Total space: n × O(1) = O(n)

**Key Insight**: Even though we don't create arrays or objects, recursion uses stack memory proportional to the recursion depth.

---

### Example 4: Iterative vs Recursive Comparison

```java
public class IterativeVsRecursiveSpace {
    // Iterative sum - O(1) space
    public static int sumIterative(int[] arr) {
        int sum = 0;                      // Single variable
        
        for (int num : arr) {
            sum += num;
        }
        
        return sum;                        // No recursion, no extra arrays
    }
    
    // Recursive sum - O(n) space
    public static int sumRecursive(int[] arr, int index) {
        if (index >= arr.length) {
            return 0;
        }
        
        return arr[index] + sumRecursive(arr, index + 1);
    }
    
    // Iterative reverse array in-place - O(1) space
    public static void reverseIterative(int[] arr) {
        int left = 0;
        int right = arr.length - 1;
        
        while (left < right) {
            // Swap elements
            int temp = arr[left];
            arr[left] = arr[right];
            arr[right] = temp;
            
            left++;
            right--;
        }
    }
    
    // Recursive reverse array (creates new array) - O(n) space
    public static int[] reverseRecursive(int[] arr, int index) {
        if (index < 0) {
            return new int[arr.length];
        }
        
        int[] reversed = reverseRecursive(arr, index - 1);
        reversed[arr.length - 1 - index] = arr[index];
        return reversed;
    }
    
    public static void main(String[] args) {
        int[] numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
        
        System.out.println("Sum (Iterative): " + sumIterative(numbers));
        System.out.println("Sum (Recursive): " + sumRecursive(numbers, 0));
        
        int[] arr1 = {1, 2, 3, 4, 5};
        reverseIterative(arr1);
        System.out.println("\nReversed (Iterative, in-place): " + 
                          java.util.Arrays.toString(arr1));
        
        int[] arr2 = {1, 2, 3, 4, 5};
        int[] reversedArr = reverseRecursive(arr2, arr2.length - 1);
        System.out.println("Reversed (Recursive, new array): " + 
                          java.util.Arrays.toString(reversedArr));
    }
}
```

**Output:**
```
Sum (Iterative): 55
Sum (Recursive): 55

Reversed (Iterative, in-place): [5, 4, 3, 2, 1]
Reversed (Recursive, new array): [5, 4, 3, 2, 1]
```

**Comparison Table:**

| Operation | Iterative Space | Recursive Space | Why? |
|-----------|----------------|-----------------|------|
| Sum array | O(1) | O(n) | Recursion uses n stack frames |
| Reverse in-place | O(1) | O(n) | Recursion depth = n |
| Reverse with new array | O(n) | O(n) + O(n) | New array + stack frames |

**Key Takeaway**: Iteration is almost always more space-efficient than recursion for the same problem.

---

## How to Analyze Time & Space Complexity - Step by Step

### The Repeatable Mental Process

Follow this systematic approach every time:

#### Step 1: Count the Loops

- **No loops**: Likely O(1)
- **One loop**: Likely O(n)
- **Nested loops**: Likely O(n²) or worse
- **Loop that divides**: Likely O(log n)

```java
// Example: Count loops
for (int i = 0; i < n; i++) {              // 1 loop = O(n)
    System.out.println(arr[i]);
}

for (int i = 0; i < n; i++) {              // 2 nested loops = O(n²)
    for (int j = 0; j < n; j++) {
        System.out.println(arr[i] + arr[j]);
    }
}
```

#### Step 2: Check Nesting

Nested structures multiply complexity:

```java
// Two sequential loops: O(n) + O(n) = O(n)
for (int i = 0; i < n; i++) { ... }
for (int j = 0; j < n; j++) { ... }

// Two nested loops: O(n) × O(n) = O(n²)
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n; j++) { ... }
}

// Three nested loops: O(n) × O(n) × O(n) = O(n³)
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n; j++) {
        for (int k = 0; k < n; k++) { ... }
    }
}
```

#### Step 3: Ignore Constants

Remove coefficient multipliers and additive constants:

```java
// 5n + 3 operations → O(n)
for (int i = 0; i < n; i++) {     // n iterations
    x = x + 1;                     // 2 operations each
    y = y + 2;
}
System.out.println("Done");        // 3 operations

// Result: 2n + 3 = O(n)
```

#### Step 4: Focus on Worst Case

Always analyze the worst-case scenario:

```java
// Worst case: target not found, checks entire array
public int linearSearch(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++) {
        if (arr[i] == target) {
            return i;                    // Best case: O(1)
        }
    }
    return -1;                           // Worst case: O(n)
}
// Time Complexity: O(n) - we report worst case
```

#### Step 5: Combine Steps Correctly

**Sequential operations**: Add complexities
```java
doSomething();     // O(n)
doSomethingElse(); // O(n²)
// Total: O(n + n²) = O(n²)  [keep dominant term]
```

**Nested operations**: Multiply complexities
```java
for (int i = 0; i < n; i++) {         // O(n)
    binarySearch(arr, target);         // O(log n)
}
// Total: O(n × log n) = O(n log n)
```

### Worked Example 1: Find Pairs with Given Sum

```java
public static void findPairs(int[] arr, int target) {
    int n = arr.length;                              // O(1)
    
    for (int i = 0; i < n; i++) {                    // Outer: n iterations
        for (int j = i + 1; j < n; j++) {            // Inner: (n-i-1) iterations
            if (arr[i] + arr[j] == target) {         // O(1) operation
                System.out.println("(" + arr[i] + ", " + arr[j] + ")");
            }
        }
    }
}
```

**Analysis:**
1. Count loops: 2 nested loops
2. Outer loop: runs n times
3. Inner loop: runs (n-1) + (n-2) + ... + 1 = n(n-1)/2 times total
4. Total operations: n(n-1)/2 = (n² - n)/2
5. Drop constants and lower terms: O(n²)

**Space Complexity**: O(1) - only using loop variables

---

### Worked Example 2: Merge Two Sorted Arrays

```java
public static int[] mergeSortedArrays(int[] arr1, int[] arr2) {
    int n1 = arr1.length;                             // O(1)
    int n2 = arr2.length;                             // O(1)
    int[] merged = new int[n1 + n2];                  // O(n1 + n2) space
    
    int i = 0, j = 0, k = 0;                          // O(1)
    
    // Merge both arrays
    while (i < n1 && j < n2) {                        // O(n1 + n2) iterations
        if (arr1[i] <= arr2[j]) {
            merged[k++] = arr1[i++];
        } else {
            merged[k++] = arr2[j++];
        }
    }
    
    // Copy remaining elements from arr1
    while (i < n1) {                                   // O(n1) worst case
        merged[k++] = arr1[i++];
    }
    
    // Copy remaining elements from arr2
    while (j < n2) {                                   // O(n2) worst case
        merged[k++] = arr2[j++];
    }
    
    return merged;
}
```

**Time Analysis:**
1. First while loop: at most (n1 + n2) iterations
2. Second while loop: at most n1 iterations
3. Third while loop: at most n2 iterations
4. Only one of the last two loops executes
5. Total: (n1 + n2) + max(n1, n2) ≈ (n1 + n2)
6. **Time Complexity: O(n1 + n2)** or **O(n)** where n = n1 + n2

**Space Analysis:**
1. New merged array: (n1 + n2) elements
2. Loop variables: constant
3. **Space Complexity: O(n1 + n2)** or **O(n)**

---

## Common Mistakes

### Mistake 1: Confusing Time Complexity with Execution Time

**Wrong Thinking:**
"My code took 2 seconds to run, so the time complexity is O(2)."

**Correct Understanding:**
Execution time depends on hardware, language, and implementation. Time complexity describes growth rate independent of these factors.

**Example:**
```java
// Both are O(n), but different execution times
public int sum1(int[] arr) {
    int sum = 0;
    for (int i = 0; i < arr.length; i++) {
        sum += arr[i];
    }
    return sum;
}

public int sum2(int[] arr) {
    int sum = 0;
    for (int i = 0; i < arr.length; i++) {
        sum += arr[i];
        sum = sum * 1;  // Pointless operation, slower execution
        sum = sum / 1;  // but still O(n)
    }
    return sum;
}
```

---

### Mistake 2: Forgetting Recursion Stack Space

**Wrong Analysis:**
```java
public int factorial(int n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}
// "No arrays or objects created, so space complexity is O(1)"  ❌ WRONG
```

**Correct Analysis:**
```java
// Each recursive call adds a stack frame
// Maximum stack depth = n
// Space complexity: O(n) due to call stack  ✓ CORRECT
```

**How to remember:** Every recursive call is like adding a box to a stack. More calls = more boxes = more space.

---

### Mistake 3: Incorrectly Adding Complexities

**Wrong:**
```java
// Sequential independent loops
for (int i = 0; i < n; i++) { ... }    // O(n)
for (int i = 0; i < m; i++) { ... }    // O(m)

// Claimed: O(n * m)  ❌ WRONG
```

**Correct:**
```java
// Sequential loops ADD, not multiply
// Correct: O(n + m)  ✓ CORRECT

// For nested loops, you MULTIPLY:
for (int i = 0; i < n; i++) {
    for (int j = 0; j < m; j++) {
        // O(n * m)  ✓ CORRECT
    }
}
```

**Rule:** Sequential → Add, Nested → Multiply

---

### Mistake 4: Ignoring Nested Loop Boundaries

**Subtle Case:**
```java
// What's the complexity?
for (int i = 0; i < n; i++) {
    for (int j = i; j < n; j++) {    // Note: j starts at i, not 0
        System.out.println(i + ", " + j);
    }
}
```

**Wrong Analysis:** "Two loops, so O(n²)" (technically correct, but let's verify)

**Detailed Analysis:**
- When i=0, inner loop runs n times
- When i=1, inner loop runs (n-1) times
- When i=2, inner loop runs (n-2) times
- ...
- When i=(n-1), inner loop runs 1 time
- Total: n + (n-1) + (n-2) + ... + 1 = n(n+1)/2 = n²/2 + n/2
- Simplified: O(n²) ✓

The conclusion O(n²) is correct, but understanding why matters for optimization.

---

### Mistake 5: Confusing O(log n) with O(n/2)

**Wrong Thinking:**
"The loop divides n by 2 each time, so it's O(n/2) = O(n)"

**Correct Understanding:**
```java
int i = n;
while (i > 1) {
    i = i / 2;    // Dividing BY 2, not running n/2 times
}
// This runs log₂(n) times, not n/2 times
// Complexity: O(log n), not O(n)  ✓ CORRECT
```

**Memory trick:** 
- Dividing n by 2 repeatedly → O(log n)
- Running n/2 iterations → O(n)

---

## Most Asked Interview Coding Questions with Full Solutions

### Problem 1: Find Maximum Element in Array

**Problem Statement:** Given an array of integers, find and return the maximum element.

#### Approach: Single Pass Linear Scan

```java
public class MaximumElement {
    public static int findMax(int[] arr) {
        // Edge case: empty array
        if (arr == null || arr.length == 0) {
            throw new IllegalArgumentException("Array cannot be empty");
        }
        
        // Assume first element is maximum
        int max = arr[0];
        
        // Compare with rest of elements
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] > max) {
                max = arr[i];
            }
        }
        
        return max;
    }
    
    public static void main(String[] args) {
        int[] numbers = {3, 7, 1, 9, 4, 6, 2, 8};
        
        System.out.println("Array: " + java.util.Arrays.toString(numbers));
        System.out.println("Maximum element: " + findMax(numbers));
    }
}
```

**Output:**
```
Array: [3, 7, 1, 9, 4, 6, 2, 8]
Maximum element: 9
```

**Step-by-Step Execution:**
```
Initial: max = 3 (arr[0])
i=1: arr[1]=7 > max=3 → max=7
i=2: arr[2]=1 < max=7 → no change
i=3: arr[3]=9 > max=7 → max=9
i=4: arr[4]=4 < max=9 → no change
i=5: arr[5]=6 < max=9 → no change
i=6: arr[6]=2 < max=9 → no change
i=7: arr[7]=8 < max=9 → no change
Final: max = 9
```

**Complexity Analysis:**
- **Time Complexity:** O(n) - We visit each element exactly once
- **Space Complexity:** O(1) - Only using one variable `max`

---

### Problem 2: Reverse an Array

**Problem Statement:** Given an array, reverse it in-place.

#### Approach: Two-Pointer Technique

```java
public class ReverseArray {
    public static void reverse(int[] arr) {
        int left = 0;
        int right = arr.length - 1;
        
        // Swap elements from both ends moving toward center
        while (left < right) {
            // Swap arr[left] and arr[right]
            int temp = arr[left];
            arr[left] = arr[right];
            arr[right] = temp;
            
            // Move pointers
            left++;
            right--;
        }
    }
    
    public static void main(String[] args) {
        int[] numbers = {1, 2, 3, 4, 5, 6, 7};
        
        System.out.println("Original: " + java.util.Arrays.toString(numbers));
        reverse(numbers);
        System.out.println("Reversed: " + java.util.Arrays.toString(numbers));
    }
}
```

**Output:**
```
Original: [1, 2, 3, 4, 5, 6, 7]
Reversed: [7, 6, 5, 4, 3, 2, 1]
```

**Step-by-Step Execution:**
```
Initial: [1, 2, 3, 4, 5, 6, 7]  left=0, right=6

Step 1: Swap arr[0] and arr[6]
        [7, 2, 3, 4, 5, 6, 1]  left=1, right=5

Step 2: Swap arr[1] and arr[5]
        [7, 6, 3, 4, 5, 2, 1]  left=2, right=4

Step 3: Swap arr[2] and arr[4]
        [7, 6, 5, 4, 3, 2, 1]  left=3, right=3

left >= right, stop
```

**Complexity Analysis:**
- **Time Complexity:** O(n) - Process n/2 pairs, which is O(n)
- **Space Complexity:** O(1) - Only using pointers and one temp variable

---

### Problem 3: Check if Array Contains Duplicates

**Problem Statement:** Given an array, determine if it contains any duplicate elements.

#### Brute Force Approach: O(n²) Time, O(1) Space

```java
public static boolean hasDuplicatesBruteForce(int[] arr) {
    for (int i = 0; i < arr.length; i++) {
        for (int j = i + 1; j < arr.length; j++) {
            if (arr[i] == arr[j]) {
                return true;
            }
        }
    }
    return false;
}
```

#### Optimized Approach: O(n) Time, O(n) Space

```java
import java.util.HashSet;
import java.util.Set;

public class CheckDuplicates {
    public static boolean hasDuplicates(int[] arr) {
        Set<Integer> seen = new HashSet<>();
        
        for (int num : arr) {
            // If number already in set, we found a duplicate
            if (seen.contains(num)) {
                return true;
            }
            // Add to set
            seen.add(num);
        }
        
        return false;  // No duplicates found
    }
    
    public static void main(String[] args) {
        int[] arr1 = {1, 2, 3, 4, 5};
        int[] arr2 = {1, 2, 3, 2, 5};
        
        System.out.println("Array 1 has duplicates: " + hasDuplicates(arr1));
        System.out.println("Array 2 has duplicates: " + hasDuplicates(arr2));
    }
}
```

**Output:**
```
Array 1 has duplicates: false
Array 2 has duplicates: true
```

**Step-by-Step Execution (arr2):**
```
arr2 = [1, 2, 3, 2, 5]
seen = {}

i=0: num=1, 1 not in seen, add 1 → seen = {1}
i=1: num=2, 2 not in seen, add 2 → seen = {1, 2}
i=2: num=3, 3 not in seen, add 3 → seen = {1, 2, 3}
i=3: num=2, 2 IS in seen → return true (duplicate found!)
```

**Complexity Comparison:**

| Approach | Time | Space | Trade-off |
|----------|------|-------|-----------|
| Brute Force | O(n²) | O(1) | Slow but no extra memory |
| HashSet | O(n) | O(n) | Fast but uses extra memory |

---

### Problem 4: Find Missing Number

**Problem Statement:** Given an array containing n distinct numbers from 0 to n, find the missing number.

#### Optimized Approach: Mathematical Formula

```java
public class FindMissingNumber {
    public static int findMissing(int[] arr) {
        int n = arr.length;  // Array has n elements, should have n+1
        
        // Sum of numbers from 0 to n
        int expectedSum = n * (n + 1) / 2;
        
        // Sum of actual array elements
        int actualSum = 0;
        for (int num : arr) {
            actualSum += num;
        }
        
        // Missing number is the difference
        return expectedSum - actualSum;
    }
    
    public static void main(String[] args) {
        int[] numbers = {0, 1, 3, 4, 5, 6, 7, 8, 9};  // Missing: 2
        
        System.out.println("Array: " + java.util.Arrays.toString(numbers));
        System.out.println("Missing number: " + findMissing(numbers));
    }
}
```

**Output:**
```
Array: [0, 1, 3, 4, 5, 6, 7, 8, 9]
Missing number: 2
```

**Step-by-Step Execution:**
```
n = 9 (array length)
Expected range: 0 to 9 (10 numbers total)

expectedSum = 9 × 10 / 2 = 45

actualSum calculation:
0 + 1 + 3 + 4 + 5 + 6 + 7 + 8 + 9 = 43

missing = 45 - 43 = 2
```

**Complexity Analysis:**
- **Time Complexity:** O(n) - Single pass through array
- **Space Complexity:** O(1) - Only using two variables

---

### Problem 5: Fibonacci - Recursive vs Iterative

**Problem Statement:** Calculate the nth Fibonacci number.

#### Recursive Approach (Inefficient)

```java
public static int fibonacciRecursive(int n) {
    if (n <= 1) {
        return n;
    }
    return fibonacciRecursive(n - 1) + fibonacciRecursive(n - 2);
}
```

**Complexity:**
- **Time:** O(2ⁿ) - Exponential, extremely slow
- **Space:** O(n) - Recursion depth

#### Iterative Approach (Efficient)

```java
public class Fibonacci {
    public static int fibonacciIterative(int n) {
        if (n <= 1) {
            return n;
        }
        
        int prev = 0;      // F(0)
        int curr = 1;      // F(1)
        
        for (int i = 2; i <= n; i++) {
            int next = prev + curr;    // F(i) = F(i-1) + F(i-2)
            prev = curr;                // Shift window
            curr = next;
        }
        
        return curr;
    }
    
    public static void main(String[] args) {
        System.out.println("First 15 Fibonacci numbers:");
        for (int i = 0; i < 15; i++) {
            System.out.print(fibonacciIterative(i) + " ");
        }
    }
}
```

**Output:**
```
First 15 Fibonacci numbers:
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377
```

**Step-by-Step Execution (n=6):**
```
Initial: prev=0, curr=1

i=2: next = 0+1=1,  prev=1, curr=1  → F(2)=1
i=3: next = 1+1=2,  prev=1, curr=2  → F(3)=2
i=4: next = 1+2=3,  prev=2, curr=3  → F(4)=3
i=5: next = 2+3=5,  prev=3, curr=5  → F(5)=5
i=6: next = 3+5=8,  prev=5, curr=8  → F(6)=8

Return 8
```

**Complexity Analysis:**
- **Time:** O(n) - Single loop from 2 to n
- **Space:** O(1) - Only three variables

**Comparison:**

| Approach | Time | Space | Notes |
|----------|------|-------|-------|
| Recursive | O(2ⁿ) | O(n) | Unusable for n > 40 |
| Iterative | O(n) | O(1) | Efficient, recommended |

---

### Problem 6: Two Sum Problem

**Problem Statement:** Given an array and a target sum, return indices of two numbers that add up to the target.

#### Brute Force: O(n²) Time, O(1) Space

```java
public static int[] twoSumBruteForce(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[i] + nums[j] == target) {
                return new int[]{i, j};
            }
        }
    }
    return new int[]{-1, -1};  // Not found
}
```

#### Optimized: O(n) Time, O(n) Space

```java
import java.util.HashMap;
import java.util.Map;

public class TwoSum {
    public static int[] twoSum(int[] nums, int target) {
        // Map: number → index
        Map<Integer, Integer> map = new HashMap<>();
        
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            
            // Check if complement exists
            if (map.containsKey(complement)) {
                return new int[]{map.get(complement), i};
            }
            
            // Store current number and index
            map.put(nums[i], i);
        }
        
        return new int[]{-1, -1};  // Not found
    }
    
    public static void main(String[] args) {
        int[] nums = {2, 7, 11, 15};
        int target = 9;
        
        int[] result = twoSum(nums, target);
        System.out.println("Indices: [" + result[0] + ", " + result[1] + "]");
        System.out.println("Numbers: " + nums[result[0]] + " + " + 
                          nums[result[1]] + " = " + target);
    }
}
```

**Output:**
```
Indices: [0, 1]
Numbers: 2 + 7 = 9
```

**Step-by-Step Execution:**
```
nums = [2, 7, 11, 15], target = 9
map = {}

i=0: nums[0]=2, complement = 9-2 = 7
     7 not in map, add 2 → map = {2: 0}

i=1: nums[1]=7, complement = 9-7 = 2
     2 IS in map at index 0!
     Return [0, 1]
```

**Complexity Analysis:**
- **Time:** O(n) - Single pass, O(1) hash operations
- **Space:** O(n) - HashMap stores up to n elements

---

### Problem 7: Check if String is Palindrome

**Problem Statement:** Determine if a string reads the same forward and backward.

#### Approach: Two-Pointer Technique

```java
public class PalindromeCheck {
    public static boolean isPalindrome(String s) {
        // Convert to lowercase and remove non-alphanumeric
        s = s.toLowerCase().replaceAll("[^a-z0-9]", "");
        
        int left = 0;
        int right = s.length() - 1;
        
        while (left < right) {
            if (s.charAt(left) != s.charAt(right)) {
                return false;  // Mismatch found
            }
            left++;
            right--;
        }
        
        return true;  // All characters matched
    }
    
    public static void main(String[] args) {
        String test1 = "racecar";
        String test2 = "A man, a plan, a canal: Panama";
        String test3 = "hello";
        
        System.out.println("'" + test1 + "' is palindrome: " + isPalindrome(test1));
        System.out.println("'" + test2 + "' is palindrome: " + isPalindrome(test2));
        System.out.println("'" + test3 + "' is palindrome: " + isPalindrome(test3));
    }
}
```

**Output:**
```
'racecar' is palindrome: true
'A man, a plan, a canal: Panama' is palindrome: true
'hello' is palindrome: false
```

**Step-by-Step Execution (test1 = "racecar"):**
```
After preprocessing: "racecar"
left=0, right=6

Step 1: s[0]='r', s[6]='r' ✓ match, left=1, right=5
Step 2: s[1]='a', s[5]='a' ✓ match, left=2, right=4
Step 3: s[2]='c', s[4]='c' ✓ match, left=3, right=3

left >= right, stop. Return true.
```

**Complexity Analysis:**
- **Time:** O(n) - Check n/2 pairs
- **Space:** O(n) - Preprocessing creates new string (can be optimized to O(1))

---

## Comparison Tables & Visual Aids

### Time Complexity Comparison Table

| Complexity | Name | Example Operations | n=10 | n=100 | n=1000 | Growth Pattern |
|------------|------|-------------------|------|-------|--------|----------------|
| O(1) | Constant | Array access, arithmetic | 1 | 1 | 1 | Flat line |
| O(log n) | Logarithmic | Binary search | 3 | 7 | 10 | Very slow growth |
| O(n) | Linear | Loop through array | 10 | 100 | 1000 | Straight diagonal |
| O(n log n) | Linearithmic | Merge sort, quick sort | 30 | 664 | 9966 | Slightly curved |
| O(n²) | Quadratic | Nested loops | 100 | 10,000 | 1,000,000 | Steep curve |
| O(2ⁿ) | Exponential | Recursive Fibonacci | 1024 | 1.27×10³⁰ | Impossible | Vertical line |

### Space Complexity Table

| Algorithm | Auxiliary Space | Explanation |
|-----------|----------------|-------------|
| Linear search | O(1) | Only uses loop variable |
| Binary search (iterative) | O(1) | Only uses pointers |
| Binary search (recursive) | O(log n) | Recursion stack depth |
| Merge sort | O(n) | Temporary arrays for merging |
| Quick sort | O(log n) | Recursion stack (average) |
| Bubble sort | O(1) | In-place swapping |
| Fibonacci (recursive) | O(n) | Call stack depth |
| Fibonacci (iterative) | O(1) | Only stores last two values |

### Iterative vs Recursive Comparison

| Aspect | Iterative | Recursive |
|--------|-----------|-----------|
| Space Complexity | Usually O(1) | Usually O(n) due to stack |
| Speed | Generally faster | Slower due to function call overhead |
| Code Readability | Can be verbose | Often more elegant and concise |
| Risk of Stack Overflow | No | Yes, for deep recursion |
| When to Use | Space-critical, performance-critical | Naturally recursive problems (trees, etc) |
| Examples | Loops, two-pointers | Tree traversal, divide and conquer |

### Common Patterns Recognition Table

| If You See... | Time Complexity is Likely... |
|---------------|------------------------------|
| Single loop | O(n) |
| Nested loop (both iterate n times) | O(n²) |
| Dividing n by constant each iteration | O(log n) |
| Sorting then searching | O(n log n) |
| Recursion with 2 branches | O(2ⁿ) |
| Recursion with 1 branch | O(n) |
| HashSet/HashMap lookup | O(1) average |
| Recursive tree traversal | O(n) where n = nodes |

---

## Summary and Cheat Sheet

### How to Quickly Identify Time Complexity

1. **No loops, no recursion** → O(1)
2. **One simple loop** → O(n)
3. **Loop that halves n** → O(log n)
4. **Two nested loops** → O(n²)
5. **Sorting involved** → O(n log n) minimum
6. **Recursion with two branches** → Likely O(2ⁿ)

### How to Spot Space Complexity

1. **Only variables, no arrays** → O(1)
2. **New array of size n** → O(n)
3. **Recursion depth is n** → O(n) stack space
4. **Creating 2D array** → O(n²)
5. **Using HashMap with n elements** → O(n)

### Interview Shortcuts & Tips

**Always Start With Brute Force**
- Mention it even if you know better solution
- Shows you can solve the problem
- Provides baseline for optimization

**Communicate Complexity**
- State time and space complexity for every solution
- Compare brute force vs optimized
- Explain trade-offs

**Common Optimization Strategies**
- Hash maps: trade space for time
- Two pointers: optimize array problems
- Binary search: when data is sorted
- Dynamic programming: cache recursive results

**Red Flags in Interviews**
- O(2ⁿ) or O(n!) → Usually wrong, look for optimization
- Claiming O(1) for array processing → Double-check
- Forgetting recursion stack space → Don't miss it

### One-Screen Revision Guide

```
TIME COMPLEXITY HIERARCHY (fastest to slowest):
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)

SPACE COMPLEXITY RULES:
- Primitive variables: O(1)
- Array of size n: O(n)
- Recursion depth d: O(d)
- 2D array n×m: O(n×m)

LOOP ANALYSIS:
- Sequential loops: ADD complexities
- Nested loops: MULTIPLY complexities
- Loop halving n: LOG complexity

BIG-O SIMPLIFICATION:
1. Drop constants: O(2n) → O(n)
2. Drop lower terms: O(n² + n) → O(n²)
3. Focus on dominant term

INTERVIEW CHECKLIST:
✓ State brute force approach first
✓ Mention time & space for all solutions
✓ Explain trade-offs
✓ Consider edge cases
✓ Optimize iteratively
✓ Test with examples
```

### Final Interview Advice

**Before Writing Code:**
- Understand the problem completely
- Ask about constraints (input size, memory limits)
- Discuss approach and complexity before coding

**While Writing Code:**
- Think out loud
- Mention complexity as you code
- Explain your logic

**After Writing Code:**
- Walk through an example
- Analyze time and space complexity
- Discuss potential optimizations
- Consider edge cases

**Remember:** Interviewers care more about your thought process than perfect code. Demonstrating understanding of complexity shows you think about scalability and efficiency, which is what makes a great engineer.

---

## Conclusion

Understanding time and space complexity is fundamental to becoming a strong software engineer. These concepts allow you to:

- Write scalable code that works for millions of users
- Make informed decisions about algorithm trade-offs
- Ace technical interviews at top companies
- Optimize systems for performance and cost

The key is practice. Analyze every algorithm you write, recognize patterns, and internalize the common complexities. With time, identifying O(n) vs O(n²) becomes second nature.

Keep coding, keep analyzing, and remember: the best algorithm is the one that solves the problem efficiently within your constraints.

---

**Document Version:** 1.0  
**Last Updated:** January 2026  
**Target Audience:** Java developers preparing for technical interviews  
**Prerequisites:** Basic Java knowledge, understanding of arrays and loops

---