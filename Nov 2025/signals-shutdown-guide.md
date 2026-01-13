# Deep Dive: OS Signals and Java/Spring Boot Shutdown Hooks

## 1. Introduction

Graceful shutdown is one of the most misunderstood concepts in backend engineering, yet it's **critical** for production systems. Without proper shutdown handling, you risk:

- **Data corruption**: Unfinished database transactions left hanging
- **Lost requests**: In-flight API requests terminated mid-execution
- **Resource leaks**: Unclosed database connections, file handles, Kafka consumer groups
- **Cascading failures**: Downstream services unable to reconnect
- **Poor user experience**: Sudden disconnections during rolling deployments

This guide will teach you **why** signals exist, **how** they work at the OS level, and **how** to handle them gracefully in Java and Spring Boot applications. By the end, you'll understand the mental model so deeply that you'll never wonder again what happens when you press Ctrl+C.

---

## 2. Understanding OS Signals (Foundations)

### 2.1 What Is a Process?

In Unix/Linux, a **process** is a running instance of a program. Every process gets:

- **Process ID (PID)**: A unique identifier (e.g., PID 1234)
- **State**: Running, stopped, zombie, etc.
- **File descriptors**: Open files, sockets, pipes
- **Signal handlers**: Code that runs when the process receives a signal

Think of a process as **a living, breathing entity inside your OS that the kernel manages**.

### 2.2 What Is a Signal?

A **signal** is a **software interrupt** — a lightweight asynchronous notification sent to a process by the operating system or another process. It's the OS way of saying:

> "Hey process! Something important just happened. Stop what you're doing and handle this."

Signals are:
- **Asynchronous**: They arrive at unpredictable times
- **Low-overhead**: Much lighter than context switching
- **Named**: Each signal has a symbolic name (SIGTERM, SIGKILL, etc.) and a number
- **Catchable or fatal**: Some can be handled; others are unstoppable

### 2.3 Why Operating Systems Use Signals

Signals exist because processes need to respond to external events **without constant polling**. Imagine managing a restaurant:

- **Polling approach** (bad): Manager constantly asks every chef "Are you done?" — wastes time and attention
- **Signal approach** (good): When an order is ready, a bell rings (signal) — chef responds when ready

Real examples:
- **User presses Ctrl+C**: OS sends SIGINT to interrupt the program
- **Parent process dies**: OS sends SIGHUP to child processes
- **Memory pressure**: OS sends signals to reduce resource usage
- **Timeout occurs**: OS sends SIGALRM
- **Child process exits**: OS sends SIGCHLD to parent
- **Graceful shutdown requested**: Orchestrator (Kubernetes) sends SIGTERM

### 2.4 Signal Lifecycle: Who Sends, Who Receives, What Happens

Here's the complete flow:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. SIGNAL GENERATION                                        │
│    • User presses Ctrl+C                                    │
│    • Parent process calls kill() syscall                    │
│    • Kernel detects hardware exception                      │
│    • Timer expires                                          │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. SIGNAL DELIVERY                                          │
│    • Kernel marks signal as "pending" in process table      │
│    • Signal queued but not yet processed                    │
│    • Waits for next kernel context switch                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. SIGNAL HANDLING (at safe point)                          │
│    • Kernel transfers control to user-space signal handler  │
│    • Process executes handler code                          │
│    • Handler completes and returns to interrupted code      │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. DEFAULT ACTION (if no handler)                           │
│    • Process terminates                                     │
│    • Process dumps core (if SIGQUIT)                        │
│    • Process ignores signal (if default is Ignore)          │
│    • Process stops/continues (if SIGSTOP/SIGCONT)           │
└─────────────────────────────────────────────────────────────┘
```

**Key insight**: The kernel doesn't interrupt a process mid-instruction. It waits for a **safe point** (usually between system calls) to deliver the signal. This is why blocked system calls can be interrupted, but tight CPU loops can't receive signals until they make a syscall.

---

## 3. Deep Dive into Common Signals

### 3.1 Signal Comparison Table

| Signal  | Number | Can Catch? | Can Ignore? | Graceful? | Exit Code | Typical Use Case |
|---------|--------|-----------|-----------|-----------|-----------|------------------|
| SIGINT  | 2      | ✅ Yes    | ✅ Yes    | ✅ Yes    | 130       | Ctrl+C from terminal |
| SIGTERM | 15     | ✅ Yes    | ✅ Yes    | ✅ Yes    | 143       | `kill <pid>` or orchestrator shutdown request |
| SIGQUIT | 3      | ✅ Yes    | ✅ Yes    | ⚠️ Yes*   | 131       | Ctrl+\\ from terminal; creates core dump |
| SIGKILL | 9      | ❌ No     | ❌ No     | ❌ No     | 137       | Force kill (last resort) — cannot be caught |
| SIGSTOP | 19     | ❌ No     | ❌ No     | N/A       | -         | Pause process (cannot be caught) |

*SIGQUIT creates a core dump by default, which is useful for debugging but not "graceful" in the sense of cleanup code running.

### 3.2 SIGINT: Interrupt from Keyboard

**Purpose**: User-initiated termination  
**Trigger**: Pressing Ctrl+C in terminal  
**Can be caught**: ✅ Yes  
**Can be ignored**: ✅ Yes  

**What happens**:
1. You press Ctrl+C
2. Terminal driver sends SIGINT to the foreground process
3. Process receives signal and can:
   - Gracefully shut down (if handler installed)
   - Terminate immediately (if default action)
   - Ignore it (if handler ignores it)

**Real-world example**:
```bash
$ java -jar myapp.jar
# ... app running ...
^C  # You press Ctrl+C
# SIGINT delivered to Java process
# If handled: "Shutting down gracefully..."
# If not handled: Process terminates immediately
```

### 3.3 SIGTERM: Termination Signal

**Purpose**: Polite shutdown request from OS or another process  
**Trigger**: `kill <pid>`, `docker stop`, Kubernetes pod deletion, systemd service stop  
**Can be caught**: ✅ Yes  
**Can be ignored**: ✅ Yes  

**What happens**:
1. External actor (kill command, container runtime, orchestrator) sends SIGTERM
2. Process receives SIGTERM and should:
   - Release resources
   - Complete in-flight requests
   - Close connections gracefully
   - Exit cleanly

**Real-world examples**:

```bash
# Standard Unix kill command
$ kill 1234  # Sends SIGTERM to PID 1234

# Docker stop (sends SIGTERM, waits 10s, then SIGKILL)
$ docker stop my_container

# Kubernetes pod termination
# kubelet sends SIGTERM, waits 30s (terminationGracePeriodSeconds), then SIGKILL
$ kubectl delete pod my-pod

# systemd service stop
$ systemctl stop myservice
```

### 3.4 SIGQUIT: Quit and Core Dump

**Purpose**: Quit with core dump for debugging  
**Trigger**: Pressing Ctrl+\\ in terminal  
**Can be caught**: ✅ Yes  
**Can be ignored**: ✅ Yes  

**What happens**:
1. Press Ctrl+\\
2. Terminal sends SIGQUIT to process
3. By default, process terminates and writes a **core dump** (memory snapshot) to disk
4. Developers can analyze core dump for debugging

**Use case**: Advanced debugging only. Rarely used in production.

```bash
$ java -jar myapp.jar
# ... app running ...
^\  # You press Ctrl+\
# SIGQUIT delivered → core dump written
```

### 3.5 SIGKILL: Force Kill (Nuclear Option)

**Purpose**: Forceful, unstoppable termination  
**Trigger**: `kill -9 <pid>`, `docker kill`, or after SIGTERM grace period expires  
**Can be caught**: ❌ **No** — Cannot be caught or ignored  
**Can be ignored**: ❌ **No**  

**What happens**:
1. SIGKILL is sent to process
2. **The kernel forcefully terminates the process immediately**
3. **No signal handler runs** — no cleanup code executes
4. **All resources are abandoned** — files left open, transactions incomplete, connections dropped

**Why it can't be caught**: The kernel **never delivers SIGKILL to user-space**. The kernel terminates the process directly. This prevents a misbehaving process from ignoring it.

**Exit code**: 137 (128 + 9)

**Real-world scenario**:
```bash
$ docker stop my_container  # Sends SIGTERM
$ # 10 seconds pass...
$ # Container didn't stop, so docker sends SIGKILL
# Process forcefully terminated — no cleanup
```

### 3.6 SIGSTOP: Stop/Pause Process

**Purpose**: Pause process execution  
**Trigger**: Pressing Ctrl+Z in terminal or `kill -STOP <pid>`  
**Can be caught**: ❌ No  
**Can be ignored**: ❌ No  

**What happens**:
1. SIGSTOP is sent
2. Process pauses (all threads suspended)
3. Process can be resumed with SIGCONT
4. **No cleanup or shutdown — just paused**

**Use case**: Debugging, process management, container pause. Not for shutdown.

---

## 4. Real-Time Scenarios in Production

### 4.1 Scenario: Ctrl+C on Local Machine

**What you do**:
```bash
$ java -jar myapp.jar
Started MyApp in 2.3 seconds
^C  # Press Ctrl+C
```

**What happens inside**:
1. Terminal driver captures Ctrl+C
2. Terminal sends **SIGINT** to Java process (PID 1234)
3. JVM signal handler catches SIGINT
4. JVM initiates shutdown sequence
5. Shutdown hooks registered via `Runtime.addShutdownHook()` execute
6. Spring ApplicationContext closes gracefully
7. `@PreDestroy` methods run, database connections close
8. JVM exits (typically exit code 0)

**Without proper shutdown handling**:
- SIGINT received but no handler → immediate termination
- Database transactions abort mid-flight
- Connections left open
- Message queue consumers stop abruptly

### 4.2 Scenario: Stopping a Spring Boot Service

**Via actuator endpoint**:
```bash
$ curl -X POST http://localhost:8080/actuator/shutdown
```

**What happens**:
1. Spring Boot Actuator receives shutdown request
2. Calls `SpringApplication.exit()` programmatically
3. Spring's shutdown hook (registered automatically by Spring Boot) is triggered
4. ApplicationContext begins shutdown phase
5. SmartLifecycle beans receive `stop()` call in reverse phase order
6. Spring calls context.close()
7. All beans' `@PreDestroy` methods execute
8. DisposableBean.destroy() methods execute
9. Application gracefully exits

### 4.3 Scenario: Docker Container Stop

**Command**:
```bash
$ docker stop my_java_app  # Or docker-compose stop
```

**Timeline**:
1. `docker stop` sends **SIGTERM** to process with PID 1 inside container
2. Java process receives SIGTERM
3. Shutdown hooks execute (if installed)
4. **10-second grace period** (configurable via `docker stop --time`)
5. If process still running after 10s: docker sends **SIGKILL**
6. Process forcefully terminated

**Dockerfile matters**:
```dockerfile
# ❌ BAD: Entrypoint is shell script (shell becomes PID 1)
ENTRYPOINT ["/bin/sh", "-c", "java -jar app.jar"]
# SIGTERM goes to shell, not Java! Java never gets signal!

# ✅ GOOD: Java directly as PID 1
ENTRYPOINT ["java", "-jar", "app.jar"]
# SIGTERM goes directly to Java process
```

### 4.4 Scenario: Kubernetes Pod Termination

**Trigger**: Rolling update, manual deletion, node eviction, auto-scaling

**Timeline**:
1. API server sends pod deletion request
2. Kubelet receives request and marks pod as "Terminating"
3. Kubelet sends **SIGTERM** to main container process (PID 1)
4. **30-second grace period** (default, set via `terminationGracePeriodSeconds`)
5. If running after grace period: kubelet sends **SIGKILL**
6. Pod removed from cluster

**Kubernetes manifest**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-java-app
spec:
  containers:
  - name: app
    image: my-java-app:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # Additional grace period
  terminationGracePeriodSeconds: 60  # Total grace period
```

**What happens inside the Pod**:
1. SIGTERM arrives at Java process
2. Shutdown hooks execute (drain requests, close connections)
3. `preStop` hook executes (additional delay if needed)
4. JVM exits cleanly
5. Pod removed

**Without proper handling**:
- SIGTERM ignored → pods force-killed (SIGKILL) after 30s
- In-flight requests lost
- Database connections abandoned
- Load balancer still routing traffic to dying pod

### 4.5 Scenario: Auto-Scaling Shutdown

**Scenario**: Cloud provider scales down instance; 50 Java apps need to stop gracefully

**What happens**:
1. Cloud platform sends SIGTERM to all processes
2. Each process has limited time (grace period) to shut down
3. If any process doesn't stop in time, platform force-kills it
4. If multiple apps compete for resources, slower ones fail
5. Data loss or corruption possible

**Critical**: Must handle shutdown **fast enough** that all apps complete before force kill.

---

## 5. Java Shutdown Hooks (Core Concept)

### 5.1 What Is a Shutdown Hook?

A **shutdown hook** is a thread that the JVM starts just before it exits. Shutdown hooks are the **primary mechanism** for graceful shutdown in Java.

```java
// Simple shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("JVM shutting down! Cleaning up resources...");
    // Cleanup code runs here
}));
```

### 5.2 How JVM Shutdown Works Internally

```
┌──────────────────────────────────────────────────────────┐
│ JVM SHUTDOWN SEQUENCE                                    │
└──────────────────────────────────────────────────────────┘

1. Shutdown initiated
   • System.exit() called
   • SIGTERM/SIGINT received
   • All non-daemon threads finish

2. Shutdown hooks begin
   ├─ START all registered shutdown hooks
   ├─ Hooks run concurrently (unspecified order)
   ├─ JVM waits for all hooks to complete
   └─ If a hook never finishes → JVM hangs

3. Shutdown hooks complete
   ├─ Any hook that called System.exit() inside itself
   │  → causes JVM to hang (deadlock)
   └─ If any hook is slow → overall shutdown is slow

4. JVM halts
   • All resources freed by OS
   • Process exits
```

### 5.3 When Shutdown Hooks Run

Shutdown hooks run when:
- ✅ `System.exit()` called
- ✅ All non-daemon threads complete
- ✅ SIGINT (Ctrl+C) received
- ✅ SIGTERM received (and caught)
- ✅ User logs off (SIGHUP)
- ✅ System shuts down (SIGTERM from init)

Shutdown hooks **do NOT run** when:
- ❌ `Runtime.halt()` called (force stop, no cleanup)
- ❌ SIGKILL received (force kill, OS terminates process)
- ❌ Hardware failure, power loss, kernel panic
- ❌ Process killed externally without signal delivery

### 5.4 When They Do NOT Run (Critical!)

```java
// Example 1: Runtime.halt() → No shutdown hooks!
Runtime.getRuntime().addShutdownHook(...);
Runtime.getRuntime().halt(1);  // ❌ Shutdown hooks never run!

// Example 2: SIGKILL → No shutdown hooks!
$ kill -9 1234  # ❌ No signal handler, no cleanup

// Example 3: Calling System.exit() inside shutdown hook → Deadlock!
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Cleaning up...");
    System.exit(0);  // ❌ DEADLOCK! JVM hangs forever
}));
```

### 5.5 JVM Signal Handling

The JVM internally handles OS signals:

| Signal | JVM Behavior |
|--------|--------------|
| SIGINT | Triggers shutdown sequence (runs hooks) |
| SIGTERM | Triggers shutdown sequence (runs hooks) |
| SIGQUIT | Dumps thread stack trace; doesn't shutdown |
| SIGKILL | Force terminates JVM (no hooks, no cleanup) |
| SIGHUP | Triggers shutdown sequence on non-Windows systems |
| SIGUSR1, SIGUSR2 | Can be custom-handled by application |

---

## 6. Java Code Examples (Simple to Advanced)

### 6.1 Minimal Shutdown Hook Example

```java
public class BasicShutdownExample {
    public static void main(String[] args) {
        // Register a simple shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Shutdown hook running!");
            System.out.println("Cleaning up resources...");
        }));

        System.out.println("Application started");

        // Simulate long-running application
        try {
            Thread.sleep(Long.MAX_VALUE);
        } catch (InterruptedException e) {
            System.out.println("Application interrupted");
        }
    }
}
```

**Output when you press Ctrl+C**:
```
Application started
Shutdown hook running!
Cleaning up resources...
```

### 6.2 Graceful Resource Cleanup Example

```java
public class GracefulResourceCleanup {
    private static final Logger logger = LoggerFactory.getLogger(GracefulResourceCleanup.class);
    
    private Connection dbConnection;
    private Server httpServer;

    public GracefulResourceCleanup() {
        // Register shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(this::shutdown, "ShutdownHook"));
    }

    public void startup() throws Exception {
        // Initialize resources
        dbConnection = DriverManager.getConnection("jdbc:mysql://localhost/mydb");
        httpServer = new Server(8080);
        httpServer.start();

        logger.info("Application started with resources initialized");
    }

    private void shutdown() {
        logger.info("Shutdown hook triggered - starting graceful shutdown");

        // Clean up in reverse order of creation
        try {
            if (httpServer != null && httpServer.isRunning()) {
                logger.info("Stopping HTTP server...");
                httpServer.stop();
            }
        } catch (Exception e) {
            logger.error("Error stopping HTTP server", e);
        }

        try {
            if (dbConnection != null && !dbConnection.isClosed()) {
                logger.info("Closing database connection...");
                dbConnection.close();
            }
        } catch (SQLException e) {
            logger.error("Error closing database connection", e);
        }

        logger.info("Shutdown complete");
    }

    public static void main(String[] args) throws Exception {
        GracefulResourceCleanup app = new GracefulResourceCleanup();
        app.startup();
        
        // App runs until shutdown signal
        Thread.currentThread().join();
    }
}
```

### 6.3 Thread Pool and Executor Shutdown

```java
public class ExecutorShutdownExample {
    private static final Logger logger = LoggerFactory.getLogger(ExecutorShutdownExample.class);
    
    private ExecutorService taskExecutor;
    private static final int SHUTDOWN_TIMEOUT_SECONDS = 30;

    public void startup() {
        taskExecutor = Executors.newFixedThreadPool(5);

        // Register shutdown hook for executor
        Runtime.getRuntime().addShutdownHook(new Thread(this::shutdownExecutor, "ExecutorShutdown"));

        // Submit some tasks
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            taskExecutor.submit(() -> {
                try {
                    logger.info("Task {} executing", taskId);
                    Thread.sleep(5000);  // Simulate work
                    logger.info("Task {} completed", taskId);
                } catch (InterruptedException e) {
                    logger.warn("Task {} interrupted", taskId);
                    Thread.currentThread().interrupt();
                }
            });
        }
    }

    private void shutdownExecutor() {
        logger.info("Initiating executor shutdown...");
        
        // Stop accepting new tasks
        taskExecutor.shutdown();
        
        try {
            // Wait for running tasks to complete
            if (!taskExecutor.awaitTermination(SHUTDOWN_TIMEOUT_SECONDS, TimeUnit.SECONDS)) {
                logger.warn("Executor did not terminate in time, forcing shutdown");
                
                // Force shutdown remaining tasks
                List<Runnable> pending = taskExecutor.shutdownNow();
                logger.warn("Forced shutdown of {} pending tasks", pending.size());
                
                // Wait a bit more for forceful shutdown
                if (!taskExecutor.awaitTermination(5, TimeUnit.SECONDS)) {
                    logger.error("Executor still not terminated after forceful shutdown!");
                }
            } else {
                logger.info("All executor tasks completed successfully");
            }
        } catch (InterruptedException e) {
            logger.error("Interrupted waiting for executor shutdown", e);
            taskExecutor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) {
        ExecutorShutdownExample app = new ExecutorShutdownExample();
        app.startup();
    }
}
```

### 6.4 Database Connection Pool Shutdown

```java
public class DatabasePoolShutdown {
    private static final Logger logger = LoggerFactory.getLogger(DatabasePoolShutdown.class);
    
    private HikariDataSource dataSource;

    public void startup() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost/mydb");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(10);
        config.setIdleTimeout(60000);
        
        dataSource = new HikariDataSource(config);

        // Register shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(this::shutdownPool, "PoolShutdown"));
        
        logger.info("Database connection pool initialized");
    }

    private void shutdownPool() {
        logger.info("Shutting down database connection pool...");
        
        if (dataSource != null && !dataSource.isClosed()) {
            try {
                // HikariCP has internal graceful shutdown
                dataSource.close();
                logger.info("Connection pool closed gracefully");
            } catch (Exception e) {
                logger.error("Error closing connection pool", e);
            }
        }
    }

    public static void main(String[] args) throws Exception {
        DatabasePoolShutdown app = new DatabasePoolShutdown();
        app.startup();
        
        // Simulate work
        Thread.sleep(10000);
    }
}
```

### 6.5 Common Mistakes

```java
// ❌ MISTAKE 1: Calling System.exit() in shutdown hook → Deadlock!
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Cleaning up...");
    System.exit(0);  // DEADLOCK! Never do this!
}));

// ✅ CORRECT: Just cleanup, let JVM exit naturally
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Cleaning up...");
    // Cleanup code only
}));

// ❌ MISTAKE 2: Not handling InterruptedException in shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    try {
        Thread.sleep(5000);  // Simulate cleanup
    } catch (InterruptedException e) {
        // ❌ If interrupted, thread exits without completing cleanup!
    }
}));

// ✅ CORRECT: Handle interruption gracefully
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        logger.warn("Shutdown hook interrupted, attempting to finish cleanup");
        Thread.currentThread().interrupt();  // Restore interrupt status
    }
}));

// ❌ MISTAKE 3: Very slow shutdown hook → Long shutdown time
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    for (int i = 0; i < 1_000_000; i++) {
        expensiveOperation();  // Slow loop
    }
}));

// ✅ CORRECT: Quick shutdown hooks; move slow logic elsewhere
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // Quick cleanup only
    resource.close();
}));

// ❌ MISTAKE 4: Multiple hooks with dependencies, no guaranteed order
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    database.close();  // Might run second
}));
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    httpServer.stop();  // Might run first (depends on DB!)
}));

// ✅ CORRECT: Use single coordinated hook or explicit ordering
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    httpServer.stop();       // Stop accepting requests first
    completeInflightRequests();  // Then finish existing work
    database.close();        // Then close DB
}));
```

---

## 7. Spring Boot Shutdown Mechanism

### 7.1 How Spring Boot Integrates with JVM Shutdown Hooks

Spring Boot automatically handles shutdown gracefully. Here's what happens:

1. **Spring Boot creates ApplicationContext**
2. **Spring automatically registers a JVM shutdown hook** (if not disabled)
3. **When JVM receives SIGTERM/SIGINT, the shutdown hook runs**
4. **Spring's shutdown hook closes the ApplicationContext**
5. **ApplicationContext closing triggers bean destruction lifecycle**

```java
// Spring Boot automatically does this internally:
ConfigurableApplicationContext context = SpringApplication.run(Application.class, args);
// Behind the scenes:
// Runtime.getRuntime().addShutdownHook(new Thread(() -> {
//     context.close();  // Gracefully close Spring context
// }));
```

### 7.2 ApplicationContext Shutdown Process

```
┌─────────────────────────────────────────────────┐
│ ApplicationContext.close() is called            │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ 1. Stop accepting new requests                  │
│    (web server stops listening)                 │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ 2. Stop SmartLifecycle beans                    │
│    (reverse phase order)                        │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ 3. Invoke @PreDestroy on beans                  │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ 4. Call DisposableBean.destroy()                │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ 5. Call destroyMethod on @Bean                  │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│ Context closed - application shutdown complete │
└─────────────────────────────────────────────────┘
```

### 7.3 Destruction Method Order

Spring guarantees this order when beans are destroyed:

1. **SmartLifecycle.stop()** (in reverse phase order)
2. **@PreDestroy** annotated methods
3. **DisposableBean.destroy()**
4. **destroyMethod** on @Bean

```java
@Component
public class MyBean implements DisposableBean, SmartLifecycle {
    
    private static final Logger logger = LoggerFactory.getLogger(MyBean.class);
    
    // 1. SmartLifecycle.stop() runs first (in reverse phase)
    @Override
    public void stop() {
        logger.info("1. SmartLifecycle.stop() called");
    }
    
    // 2. @PreDestroy runs second
    @PreDestroy
    public void cleanup() {
        logger.info("2. @PreDestroy cleanup() called");
    }
    
    // 3. DisposableBean.destroy() runs third
    @Override
    public void destroy() {
        logger.info("3. DisposableBean.destroy() called");
    }
    
    @Override
    public int getPhase() {
        return Lifecycle.DEFAULT_PHASE;
    }
}
```

### 7.4 @PreDestroy vs DisposableBean vs SmartLifecycle

| Mechanism | When | Use Case | Order |
|-----------|------|----------|-------|
| **@PreDestroy** | Before bean destruction | General cleanup annotation | 2nd |
| **DisposableBean.destroy()** | Before bean destruction | Interface-based cleanup | 3rd |
| **SmartLifecycle.stop()** | During context shutdown | Graceful shutdown coordination | 1st |
| **destroyMethod** | After DisposableBean | Configured destroy method | 4th |

### 7.5 server.shutdown=graceful

Spring Boot 2.3+ supports graceful shutdown:

```yaml
# application.yml
server:
  shutdown: graceful  # Enable graceful shutdown (default: immediate)

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # Grace period for each shutdown phase
```

What graceful shutdown does:
- **Stops accepting new HTTP requests immediately**
- **Waits up to 30s for in-flight requests to complete**
- **If any request exceeds timeout: request is terminated**
- **After timeout: proceed to spring bean destruction**

### 7.6 spring.lifecycle.timeout-per-shutdown-phase

Configures how long Spring waits during each shutdown phase:

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 60s  # 60 seconds per phase
```

This applies to:
- SmartLifecycle bean stop() timeout
- Web server graceful shutdown timeout
- Any other lifecycle phase

**Important**: Each phase gets this full timeout independently.

---

## 8. Spring Boot Real-World Examples

### 8.1 Gracefully Stopping HTTP Requests

```java
@SpringBootApplication
public class GracefulHttpShutdownApp {
    
    public static void main(String[] args) {
        SpringApplication.run(GracefulHttpShutdownApp.class, args);
    }
}

// application.yml
/*
server:
  shutdown: graceful
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
*/

@RestController
@RequestMapping("/api")
public class ApiController {
    
    private static final Logger logger = LoggerFactory.getLogger(ApiController.class);
    
    @GetMapping("/long-task")
    public ResponseEntity<String> longTask() throws InterruptedException {
        logger.info("Long task started");
        Thread.sleep(10000);  // 10-second task
        logger.info("Long task completed");
        return ResponseEntity.ok("Task complete");
    }
}

// When you send shutdown signal (Ctrl+C):
// 1. Spring stops accepting new requests
// 2. /long-task request in progress → allowed to finish
// 3. Wait up to 30s for completion
// 4. Then proceed with bean destruction
```

### 8.2 Draining In-Flight Requests

```java
@Component
public class RequestDrainManager implements SmartLifecycle {
    
    private static final Logger logger = LoggerFactory.getLogger(RequestDrainManager.class);
    
    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private volatile boolean running = false;
    
    @Override
    public void start() {
        running = true;
        logger.info("RequestDrainManager started");
    }
    
    @Override
    public void stop() {
        logger.info("RequestDrainManager stopping - draining {} in-flight requests", 
                   activeRequests.get());
        running = false;
        
        // Wait for all in-flight requests to complete
        long startTime = System.currentTimeMillis();
        while (activeRequests.get() > 0) {
            try {
                logger.info("Waiting for {} requests to complete", activeRequests.get());
                Thread.sleep(500);
            } catch (InterruptedException e) {
                logger.warn("Interrupted while draining requests");
                Thread.currentThread().interrupt();
                break;
            }
            
            // Prevent infinite wait
            if (System.currentTimeMillis() - startTime > 25000) {
                logger.warn("Timeout draining requests after 25s, giving up");
                break;
            }
        }
        
        logger.info("RequestDrainManager stopped");
    }
    
    @Override
    public int getPhase() {
        return Integer.MAX_VALUE - 100;  // Run early in shutdown sequence
    }
    
    // Interceptor to track active requests
    public void recordRequest() {
        activeRequests.incrementAndGet();
    }
    
    public void releaseRequest() {
        activeRequests.decrementAndGet();
    }
    
    @Override
    public boolean isRunning() {
        return running;
    }
}

@Component
public class RequestTrackingInterceptor implements WebRequestInterceptor {
    
    private final RequestDrainManager drainManager;
    
    public RequestTrackingInterceptor(RequestDrainManager drainManager) {
        this.drainManager = drainManager;
    }
    
    @Override
    public void preHandle(WebRequest request) throws Exception {
        drainManager.recordRequest();
    }
    
    @Override
    public void afterCompletion(WebRequest request, Exception ex) throws Exception {
        drainManager.releaseRequest();
    }
}
```

### 8.3 Closing Database Connections

```java
@Configuration
public class DataSourceConfig {
    
    private static final Logger logger = LoggerFactory.getLogger(DataSourceConfig.class);
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost/mydb");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setIdleTimeout(600000);
        config.setConnectionTimeout(20000);
        
        HikariDataSource dataSource = new HikariDataSource(config);
        
        // HikariCP automatically shuts down gracefully when Spring closes the context
        // No explicit shutdown hook needed!
        
        return dataSource;
    }
}

// During shutdown:
// 1. HikariCP stops accepting new connections
// 2. Waits for existing connections to return to pool
// 3. Closes all pooled connections
// 4. Graceful shutdown complete
```

### 8.4 Stopping Kafka Consumers

```java
@Configuration
public class KafkaConsumerConfig {
    
    private static final Logger logger = LoggerFactory.getLogger(KafkaConsumerConfig.class);
    
    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> 
        kafkaListenerContainerFactory(ConsumerFactory<String, String> consumerFactory) {
        
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        
        // Configure graceful shutdown
        factory.getContainerProperties().setShutdownTimeout(30000);  // 30 seconds
        
        return factory;
    }
}

@Component
public class OrderEventListener implements SmartLifecycle {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderEventListener.class);
    
    private final KafkaListenerEndpointRegistry registry;
    
    public OrderEventListener(KafkaListenerEndpointRegistry registry) {
        this.registry = registry;
    }
    
    @KafkaListener(topics = "orders", groupId = "order-service")
    public void onOrderEvent(OrderEvent event) {
        logger.info("Processing order event: {}", event.getOrderId());
        // Process order
    }
    
    @Override
    public void stop() {
        logger.info("Stopping Kafka listeners...");
        registry.stop();  // Gracefully stop all Kafka listeners
        logger.info("Kafka listeners stopped");
    }
    
    @Override
    public void start() {
        logger.info("Starting Kafka listeners...");
        registry.start();
    }
    
    @Override
    public int getPhase() {
        return Integer.MAX_VALUE - 10;  // Stop early in shutdown
    }
    
    @Override
    public boolean isRunning() {
        return registry.isRunning();
    }
}
```

### 8.5 Preventing Data Loss in Async Tasks

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    private static final Logger logger = LoggerFactory.getLogger(AsyncConfig.class);
    
    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setWaitForTasksToCompleteOnShutdown(true);  // Wait for tasks!
        executor.setAwaitTerminationSeconds(60);  // Max 60 seconds to wait
        executor.initialize();
        return executor;
    }
}

@Component
public class OrderProcessor {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderProcessor.class);
    
    @Async("taskExecutor")
    public void processOrderAsync(Order order) {
        logger.info("Processing order {} asynchronously", order.getId());
        try {
            // Long-running task
            Thread.sleep(30000);  // 30 seconds of work
            logger.info("Order {} processing complete", order.getId());
        } catch (InterruptedException e) {
            logger.warn("Order {} processing interrupted", order.getId());
            Thread.currentThread().interrupt();
        }
    }
}

// During shutdown:
// 1. No new async tasks accepted
// 2. Wait up to 60s for running tasks to complete
// 3. Then proceed with shutdown
```

---

## 9. Best Practices (Production-Grade)

### 9.1 What to ALWAYS Do

✅ **Always register shutdown hooks for non-Spring-managed resources**

```java
// Custom resource that Spring doesn't manage
MyCustomConnection connection = new MyCustomConnection();
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    connection.close();
}));
```

✅ **Always use server.shutdown=graceful in Spring Boot 2.3+**

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

✅ **Always set explicit timeout on ThreadPoolExecutor**

```java
ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
executor.setWaitForTasksToCompleteOnShutdown(true);
executor.setAwaitTerminationSeconds(30);
```

✅ **Always log shutdown events**

```java
logger.info("Starting graceful shutdown");
logger.info("Shutdown complete");
```

✅ **Always close resources in reverse order of creation**

```java
// Started:  Connection → Server → Cache
// Shutdown: Cache → Server → Connection (reverse!)
```

### 9.2 What to NEVER Do

❌ **Never call System.exit() in a shutdown hook**

```java
// DEADLOCK!
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.exit(0);
}));
```

❌ **Never use Runtime.halt() unless you really mean it**

```java
// Skips all shutdown hooks!
Runtime.getRuntime().halt(0);
```

❌ **Never ignore InterruptedException in shutdown hooks**

```java
// Bad - exception swallowed
try {
    Thread.sleep(5000);
} catch (InterruptedException e) {
    // Ignore
}

// Good - restore interrupt
try {
    Thread.sleep(5000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

❌ **Never use shell scripts as Dockerfile ENTRYPOINT (with shell)**

```dockerfile
# WRONG - shell becomes PID 1, Java never gets signals
ENTRYPOINT ["/bin/sh", "-c", "java -jar app.jar"]

# RIGHT - Java directly as PID 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

❌ **Never do expensive operations in shutdown hooks**

```java
// BAD - expensive operation delays shutdown
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    for (long i = 0; i < 1_000_000_000_000L; i++) {
        expensiveComputation();
    }
}));

// GOOD - quick cleanup only
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    resource.close();  // Fast operation
}));
```

❌ **Never assume shutdown hooks will run indefinitely**

```java
// BAD - infinite loop, will hang shutdown
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    while (true) {  // Never exits!
        doSomething();
    }
}));

// GOOD - bounded cleanup
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    long start = System.currentTimeMillis();
    while (shouldContinue() && System.currentTimeMillis() - start < 25000) {
        doSomething();
    }
}));
```

### 9.3 Signal Handling Dos & Don'ts

| Do ✅ | Don't ❌ |
|------|---------|
| Handle SIGTERM gracefully | Ignore SIGTERM (it's the standard shutdown signal) |
| Handle SIGINT for interactive apps | Assume SIGKILL will call cleanup code |
| Use timeouts for shutdown | Leave shutdown indefinitely open |
| Log when shutdown starts/completes | Shutdown silently (hard to debug) |
| Test shutdown scenarios | Only test happy path |
| Use try-with-resources for AutoCloseable | Manual resource management in shutdown |
| Set grace periods in Kubernetes/Docker | Use default grace periods without thinking |

### 9.4 Timeouts and Observability

**Set multiple layers of timeouts**:

```yaml
# Kubernetes
terminationGracePeriodSeconds: 60  # 60s total

# Docker Compose
stop_grace_period: 60s

# Spring Boot
spring.lifecycle.timeout-per-shutdown-phase: 30s

# Application code
executor.setAwaitTerminationSeconds(25)  # Leave margin for Spring cleanup
```

**Always add logging**:

```java
private static final Logger logger = LoggerFactory.getLogger(...);

@Override
public void stop() {
    long start = System.currentTimeMillis();
    logger.info("Shutdown started");
    
    try {
        // Cleanup code
    } finally {
        long elapsed = System.currentTimeMillis() - start;
        logger.info("Shutdown completed in {}ms", elapsed);
    }
}
```

**Monitor with metrics**:

```java
@Component
public class ShutdownMetrics {
    private final MeterRegistry meterRegistry;
    
    public ShutdownMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void recordShutdownDuration(long durationMs) {
        meterRegistry.timer("app.shutdown.duration").record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

### 9.5 Logging During Shutdown

```java
@Component
public class GracefulShutdownLogger implements SmartLifecycle {
    
    private static final Logger logger = LoggerFactory.getLogger(GracefulShutdownLogger.class);
    private final ApplicationContext context;
    
    public GracefulShutdownLogger(ApplicationContext context) {
        this.context = context;
    }
    
    @Override
    public void stop() {
        logger.info("===============================");
        logger.info("GRACEFUL SHUTDOWN INITIATED");
        logger.info("===============================");
        logger.info("Active beans: {}", context.getBeanDefinitionCount());
        logger.info("Application will shutdown after {} seconds", 30);
    }
    
    @Override
    public void start() {
        logger.info("Shutdown logger started");
    }
    
    @Override
    public int getPhase() {
        return Integer.MIN_VALUE;  // Run last in shutdown
    }
    
    @Override
    public boolean isRunning() {
        return true;
    }
}
```

---

## 10. Mental Models & Memory Aids

### 10.1 The Restaurant Analogy

Imagine managing a restaurant:

**No shutdown handling (bad)**:
- Customer yells "CLOSE NOW!"
- Manager immediately locks doors and kicks everyone out
- Customers mid-meal abandon tables, food left on plates
- Servers don't get paid, equipment damaged
- Reputation ruined

**Graceful shutdown (good)**:
- Manager announces "We're closing in 30 minutes"
- New customers turned away
- Existing customers allowed to finish meals
- Servers clean up, equipment turned off safely
- Money counted and deposited
- Everyone leaves happy, everything secured

**SIGKILL (force close)**:
- Manager throws everyone out immediately
- No time to save work or money
- Equipment left running
- Data/inventory lost

### 10.2 Signal Escalation Hierarchy

Think of signals as increasingly forceful:

```
SIGTERM (polite request)
    ↓
    ↓ [if not handled, wait 30s]
    ↓
SIGKILL (force kill, no negotiation)
```

**SIGTERM**: "Please shut down nicely"  
**SIGKILL**: "You're shutting down NOW, I'm not asking"

### 10.3 The Three Layers Model

```
┌──────────────────────────────────┐
│ LAYER 1: OS Signals              │
│ (SIGTERM, SIGINT, SIGKILL)       │
│ What: Low-level process control  │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│ LAYER 2: JVM Shutdown Hooks      │
│ (Runtime.addShutdownHook())       │
│ What: Java-level cleanup         │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│ LAYER 3: Spring Lifecycle        │
│ (@PreDestroy, SmartLifecycle)    │
│ What: Framework-level cleanup    │
└──────────────────────────────────┘
```

Each layer depends on the layer below it. If you skip Layer 2, Layer 3 never runs.

### 10.4 Remember This Forever

> **Signals are OS events. Shutdown hooks are Java threads. Spring handles them together.**

When SIGTERM arrives:
1. OS delivers signal to JVM
2. JVM's signal handler runs
3. JVM executes shutdown hooks (registered via Runtime.addShutdownHook)
4. Spring's shutdown hook closes ApplicationContext
5. Spring destroys beans (@PreDestroy, DisposableBean, etc.)
6. JVM exits

**No proper shutdown handling = Step 5 never happens = Data loss.**

### 10.5 Quick Decision Tree

```
"My app needs graceful shutdown. What do I do?"

├─ Spring Boot app? YES
│  ├─ Add to application.yml:
│  │  server.shutdown: graceful
│  │  spring.lifecycle.timeout-per-shutdown-phase: 30s
│  └─ Use @PreDestroy for cleanup
│
├─ Non-Spring resource (custom DB, socket)?
│  └─ Runtime.getRuntime().addShutdownHook(new Thread(() -> resource.close()))
│
├─ ThreadPoolExecutor?
│  ├─ executor.setWaitForTasksToCompleteOnShutdown(true)
│  └─ executor.setAwaitTerminationSeconds(30)
│
├─ Running in Docker?
│  ├─ ENTRYPOINT ["java", "-jar", "app.jar"]  (NOT shell script!)
│  └─ docker stop waits 10s, then SIGKILL
│
└─ Running in Kubernetes?
   ├─ terminationGracePeriodSeconds: 60
   └─ preStop hook for additional work
```

---

## 11. Quick Reference Cheat Sheet

### 11.1 Signals Summary

```
SIGINT (2)   → Ctrl+C from terminal          | Catchable ✅ | Graceful ✅
SIGTERM (15) → kill, docker stop, k8s delete | Catchable ✅ | Graceful ✅
SIGQUIT (3)  → Ctrl+\ from terminal          | Catchable ✅ | Core dump 📸
SIGKILL (9)  → kill -9, last resort          | NOT catchable ❌ | Force ⚠️
SIGSTOP (19) → Ctrl+Z, pause process         | NOT catchable ❌ | Pause only
```

### 11.2 Java Shutdown Summary

```java
// Register shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // Cleanup code
}));

// What runs shutdown hooks:
// ✅ System.exit() called
// ✅ SIGTERM/SIGINT received
// ✅ All non-daemon threads finish
// ❌ Runtime.halt() called
// ❌ SIGKILL received
```

### 11.3 Spring Boot Shutdown Summary

```yaml
# Enable graceful shutdown
server:
  shutdown: graceful

# Grace period
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```java
// Destruction order:
// 1. SmartLifecycle.stop()
// 2. @PreDestroy
// 3. DisposableBean.destroy()
// 4. destroyMethod on @Bean

// Example:
@PreDestroy
void cleanup() { }
```

### 11.4 Docker/Kubernetes Summary

```bash
# Docker: sends SIGTERM, waits 10s, then SIGKILL
docker stop container_name

# Kubernetes: sends SIGTERM, waits 30s (terminationGracePeriodSeconds), then SIGKILL
kubectl delete pod pod_name
```

### 11.5 Common Exit Codes

```
Exit Code | Signal     | Meaning
0         | -          | Clean exit
130       | SIGINT     | Interrupted (Ctrl+C)
137       | SIGKILL    | Killed forcefully
143       | SIGTERM    | Terminated gracefully
```

---

## 12. Troubleshooting Guide

### Problem: App doesn't stop on Ctrl+C

**Cause**: No signal handler, or handler stuck

**Solution**:
```java
// Make sure shutdown hook exists and is quick
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // Quick cleanup only
    resource.close();
}));

// Not this (causes deadlock):
// System.exit(0);
```

### Problem: Docker stop takes 10+ seconds

**Cause**: Shutdown hook too slow

**Solution**:
```yaml
# Reduce grace period
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 5s  # Faster shutdown
```

### Problem: Kubernetes pod keeps restarting

**Cause**: Shutdown exceeds grace period

**Solution**:
```yaml
apiVersion: v1
kind: Pod
spec:
  terminationGracePeriodSeconds: 60  # Increase grace period
  containers:
  - lifecycle:
      preStop:
        exec:
          command: ["/bin/sleep", "5"]
```

### Problem: Data lost during shutdown

**Cause**: In-flight requests not drained

**Solution**:
```java
// In shutdown hook:
while (activeRequests.get() > 0) {
    Thread.sleep(100);  // Wait for requests
}
// Now safe to close database
```

---

## Appendix: Full Production Example

```java
@SpringBootApplication
public class ProductionReadyApp {
    private static final Logger logger = LoggerFactory.getLogger(ProductionReadyApp.class);

    public static void main(String[] args) {
        SpringApplication.run(ProductionReadyApp.class, args);
    }
}

// application.yml
/*
server:
  shutdown: graceful
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

logging:
  level:
    root: INFO
*/

@Component
public class ProductionReadyShutdown implements SmartLifecycle {
    
    private static final Logger logger = LoggerFactory.getLogger(ProductionReadyShutdown.class);
    
    private final KafkaListenerEndpointRegistry kafkaRegistry;
    private final ThreadPoolTaskExecutor taskExecutor;
    private volatile boolean running = false;

    public ProductionReadyShutdown(
            KafkaListenerEndpointRegistry kafkaRegistry,
            ThreadPoolTaskExecutor taskExecutor) {
        this.kafkaRegistry = kafkaRegistry;
        this.taskExecutor = taskExecutor;
    }

    @Override
    public void start() {
        running = true;
        logger.info("Production shutdown manager started");
    }

    @Override
    public void stop() {
        logger.info("============ GRACEFUL SHUTDOWN INITIATED ============");
        running = false;

        long startTime = System.currentTimeMillis();

        try {
            // 1. Stop accepting Kafka messages
            logger.info("Stopping Kafka listeners...");
            kafkaRegistry.stop();
            logger.info("Kafka listeners stopped");

            // 2. Drain async tasks
            logger.info("Draining async tasks (timeout: 20s)");
            if (!taskExecutor.getThreadPoolExecutor().awaitTermination(20, TimeUnit.SECONDS)) {
                logger.warn("Forcing async task shutdown");
                taskExecutor.getThreadPoolExecutor().shutdownNow();
            }
            logger.info("Async tasks drained");

        } catch (InterruptedException e) {
            logger.warn("Interrupted during shutdown", e);
            Thread.currentThread().interrupt();
        }

        long elapsed = System.currentTimeMillis() - startTime;
        logger.info("============ SHUTDOWN COMPLETE ({}ms) ============", elapsed);
    }

    @Override
    public int getPhase() {
        return Integer.MAX_VALUE - 10;  // Run very early
    }

    @Override
    public boolean isRunning() {
        return running;
    }
}

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(25);  // Leave margin
        executor.initialize();
        return executor;
    }
}

@RestController
@RequestMapping("/api")
public class ApiController {
    
    private static final Logger logger = LoggerFactory.getLogger(ApiController.class);

    @GetMapping("/health")
    public ResponseEntity<String> health() {
        return ResponseEntity.ok("OK");
    }

    @PostMapping("/process")
    public ResponseEntity<String> process(@RequestBody String data) throws InterruptedException {
        logger.info("Processing request: {}", data);
        Thread.sleep(5000);  // Simulate work
        return ResponseEntity.ok("Processed");
    }
}
```

---

## Conclusion

Understanding OS signals and graceful shutdown is fundamental to building reliable, production-ready systems. The concepts are timeless:

1. **Signals are lightweight asynchronous notifications** — the OS's way of communicating with processes
2. **SIGTERM and SIGINT should always be handled** — they're the standard shutdown signals
3. **SIGKILL is unstoppable** — never rely on it for cleanup
4. **Shutdown hooks are your primary Java tool** — use them for non-Spring resources
5. **Spring Boot handles most of this automatically** — but you must enable graceful shutdown
6. **Timeouts are critical** — at every layer: OS, container, framework, application
7. **Test shutdown scenarios** — don't just test happy paths

With these mental models, you'll write resilient applications that shut down cleanly, lose no data, and treat your operations teams with respect.
