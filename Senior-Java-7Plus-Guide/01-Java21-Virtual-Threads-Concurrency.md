# Java 21 Virtual Threads & Modern Concurrency Master Guide

A production-grade reference guide for Senior Java Developers (7+ YOE) covering **Java 21 Project Loom**, Virtual Threads vs Platform Threads, Carrier Thread scheduling, Thread Pinning pitfalls, Structured Concurrency, Scoped Values, and Sequenced Collections.

---

## 1. Virtual Threads Architecture (Project Loom)

Prior to Java 21, every `java.lang.Thread` was a **Platform Thread** wrapped directly 1:1 around an OS thread. Operating system threads are expensive:
- Memory footprint: ~1 MB of off-heap stack memory per OS thread.
- Context switching overhead: Kernel-level context switches cost microseconds.
- Scalability limit: ~2,000 to 5,000 active OS threads per JVM instance before memory/CPU thrashing occurs.

### How Virtual Threads Work
**Virtual Threads** (`java.lang.Thread.ofVirtual()`) are lightweight user-mode threads managed entirely by the JVM runtime, decoupled from 1:1 OS thread mappings.

```text
[ Virtual Thread 1 ]  [ Virtual Thread 2 ]  [ Virtual Thread 3 ] ... [ Virtual Thread 10,000 ]
         │                     │                     │                      │
         └─────────────────────┴──────────┬──────────┴──────────────────────┘
                                          │ (Mounted / Unmounted on demand)
                                          ▼
                         [ Carrier Thread (ForkJoinPool) ]
                                          │ (1:1 Mapping)
                                          ▼
                                     [ OS Thread ]
```

- **Carrier Threads**: A default FIFO `ForkJoinPool` of platform OS threads (sized to available CPU cores by default) acts as the execution host.
- **Unmounting on I/O**: When a Virtual Thread hits a blocking I/O operation (e.g. database query, HTTP call, file read), the JVM **unmounts** the Virtual Thread's stack frame from the Carrier Thread and stores it in Java heap memory. The Carrier Thread is immediately free to execute other Virtual Threads.
- **Remounting**: Once the network/file I/O completes via OS asynchronous notifications (epoll/kqueue), the JVM scheduler **remounts** the Virtual Thread onto an available Carrier Thread to resume execution.

---

## 2. Platform Threads vs. Virtual Threads Comparison

| Feature | Platform Threads (Legacy) | Virtual Threads (Java 21+) |
| :--- | :--- | :--- |
| **OS Mapping** | 1:1 Kernel Thread mapping | $M:N$ User-mode mapping onto Carrier Threads |
| **Memory Footprint** | ~1 MB pre-allocated per thread stack | ~Bites/KBs dynamically allocated on JVM Heap |
| **Creation Cost** | Expensive (~1ms). Requires Thread Pooling | Virtually free (<1µs). **DO NOT POOL VIRTUAL THREADS!** |
| **Max Scale** | ~5,000 threads per JVM | 1,000,000+ concurrent threads per JVM |
| **Best Used For** | CPU-bound tasks (video encoding, crypto) | I/O-bound tasks (REST APIs, DB queries, microservices) |

---

## 3. Thread Pinning: The #1 Production Pitfall

### What is Thread Pinning?
When a Virtual Thread enters a **pinned** state, it **cannot be unmounted** from its Carrier Thread during blocking I/O. As a result, the underlying Carrier OS Thread remains blocked, negating the throughput benefits of Virtual Threads and causing thread pool starvation.

### Causes of Thread Pinning
1. Executing blocking I/O inside a **`synchronized` block or method**.
2. Executing blocking native calls (`JNI`).

### 🔴 Anti-Pattern: Pinning via `synchronized`

```java
// BAD: synchronized pins the Virtual Thread to the Carrier Thread!
public class LegacyInventoryService {

    public synchronized String fetchInventoryWithPinning(String sku) {
        // Blocking HTTP call inside synchronized block -> PINNED!
        return restTemplate.getForObject("https://supplier.com/api/" + sku, String.class);
    }
}
```

### 🟢 Production Solution: Refactoring to `ReentrantLock`

`java.util.concurrent.locks.ReentrantLock` does **NOT** pin Virtual Threads during blocking I/O.

```java
package com.example.demo.service;

import java.util.concurrent.locks.ReentrantLock;
import org.springframework.stereotype.Service;

@Service
public class ModernInventoryService {

    private final ReentrantLock lock = new ReentrantLock();

    public String fetchInventorySafe(String sku) {
        lock.lock();
        try {
            // Virtual Thread can cleanly UNMOUNT here during I/O!
            return restTemplate.getForObject("https://supplier.com/api/" + sku, String.class);
        } finally {
            lock.unlock();
        }
    }
}
```

### Detecting Pinning in Production
Pass the JVM flag at startup to log pinned Virtual Threads:
```bash
java -Djdk.tracePinnedThreads=full -jar application.jar
```

---

## 4. Production Code Examples

### 4.1 Creating Virtual Threads in Java 21

```java
package com.example.demo.concurrency;

import java.time.Duration;
import java.util.concurrent.Executors;

public class VirtualThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        // 1. Direct creation via Thread.ofVirtual()
        Thread vThread = Thread.ofVirtual()
                .name("user-worker-1")
                .start(() -> {
                    System.out.println("Running inside: " + Thread.currentThread());
                });
        vThread.join();

        // 2. ExecutorService with Virtual Thread Per Task
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 10_000; i++) {
                final int taskId = i;
                executor.submit(() -> {
                    Thread.sleep(Duration.ofMillis(100));
                    return "Task " + taskId + " completed";
                });
            }
        } // Executor automatically waits for all tasks to complete on close()
    }
}
```

### 4.2 Spring Boot 3.2+ Virtual Threads Configuration

In Spring Boot 3.2+, enable Virtual Threads for all Tomcat HTTP request handling and `@Async` tasks with a single configuration property:

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

Or programmatic bean configuration:

```java
package com.example.demo.config;

import org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.AsyncTaskExecutor;
import org.springframework.core.task.support.TaskExecutorAdapter;

import java.util.concurrent.Executors;

@Configuration
public class VirtualThreadConfig {

    @Bean(name = TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    public AsyncTaskExecutor asyncTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

---

## 5. Structured Concurrency (JEP 453)

Structured Concurrency treats sub-tasks running in different threads as a single unit of work, simplifying error handling, cancellation, and observability.

```java
package com.example.demo.concurrency;

import java.util.concurrent.StructuredTaskScope;
import java.util.function.Supplier;

public class OrderAggregationService {

    public record OrderDetails(String user, String inventory, String payment) {}

    public OrderDetails getOrderDetails(String orderId) throws InterruptedException {
        // Short-circuit on failure: If any subtask fails, cancel all other subtasks!
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            
            Supplier<String> userTask = scope.fork(() -> fetchUser(orderId));
            Supplier<String> inventoryTask = scope.fork(() -> fetchInventory(orderId));
            Supplier<String> paymentTask = scope.fork(() -> fetchPayment(orderId));

            scope.join();           // Join all sub-threads
            scope.throwIfFailed();  // Propagate exception if any sub-task failed

            return new OrderDetails(userTask.get(), inventoryTask.get(), paymentTask.get());
        }
    }

    private String fetchUser(String id) { return "User-123"; }
    private String fetchInventory(String id) { return "In-Stock"; }
    private String fetchPayment(String id) { return "Paid"; }
}
```

---

## 6. Scoped Values (JEP 446) vs. `ThreadLocal`

`ThreadLocal` has major drawbacks when millions of Virtual Threads are spawned:
- Unbounded memory inheritance (`InheritableThreadLocal`).
- Mutable state leading to memory leaks if not cleaned up via `remove()`.

**Scoped Values** (`java.lang.ScopedValue`) provide **immutable, bounded context propagation** across virtual threads with near-zero overhead.

```java
package com.example.demo.concurrency;

import java.lang.ScopedValue;

public class ScopedValueDemo {

    public static final ScopedValue<String> CURRENT_TENANT = ScopedValue.newInstance();

    public void handleTenantRequest(String tenantId) {
        // Bind tenantId for the duration of the lambda execution block
        ScopedValue.where(CURRENT_TENANT, tenantId).run(() -> {
            processOrder();
        });
    }

    private void processOrder() {
        // Safely access tenantId inside child method without passing parameters
        String tenant = CURRENT_TENANT.get();
        System.out.println("Processing order for tenant: " + tenant);
    }
}
```

---

## 7. Sequenced Collections (JEP 431)

Java 21 unifies ordered collection interfaces with `SequencedCollection`, `SequencedSet`, and `SequencedMap`.

```java
package com.example.demo.collections;

import java.util.ArrayList;
import java.util.SequencedCollection;

public class SequencedDemo {
    public static void main(String[] args) {
        SequencedCollection<String> list = new ArrayList<>();
        list.addLast("Second");
        list.addFirst("First");
        list.addLast("Third");

        System.out.println("First Element: " + list.getFirst()); // "First"
        System.out.println("Last Element: " + list.getLast());   // "Third"

        // Reverse order view in O(1) time
        SequencedCollection<String> reversed = list.reversed();
        System.out.println("Reversed: " + reversed); // ["Third", "Second", "First"]
    }
}
```

---

## 💡 Senior Interview Summary Tips

1. **When NOT to use Virtual Threads?**  
   CPU-intensive tasks (e.g. heavy crypto calculation, image processing). Virtual Threads do not add extra CPU cores; they optimize waiting time during I/O blocking.
2. **Never Pool Virtual Threads!**  
   `Executors.newVirtualThreadPerTaskExecutor()` creates a new Virtual Thread for every single task and discards it after execution. Pooling them breaks Loom's design.
3. **Always Check for Thread Pinning!**  
   Replace legacy `synchronized` blocks wrapping I/O calls with `ReentrantLock`.
