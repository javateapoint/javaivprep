---
name: performance-jvm-tuning
description: >
  JVM 21 tuning, Generational ZGC, Virtual Thread pinning diagnosis, Heap/Thread dump tools, 
  and Spring Boot performance optimization skill for GitHub Copilot in IntelliJ.
---

# JVM 21 & Performance Tuning Skill

## Purpose
Provides technical references, JVM flags, diagnostic CLI commands, and code patterns for diagnosing performance bottlenecks in Java 21 Spring Boot applications.

---

## 1. Java 21 JVM Garbage Collection Reference

### Generational ZGC (Recommended for Low Latency)
```bash
# Recommended Java 21 production flags for sub-millisecond GC pauses
java -XX:+UseZGC -XX:+ZGenerational -Xms4g -Xmx4g -XX:+AlwaysPreTouch -jar app.jar
```

### G1GC Tuning (Recommended for High Throughput)
```bash
java -XX:+UseG1GC -Xms4g -Xmx4g -XX:MaxGCPauseMillis=200 -XX:InitiatingHeapOccupancyPercent=45 -jar app.jar
```

---

## 2. Virtual Thread (Loom) Pinning Diagnosis

Virtual threads pin their underlying OS carrier thread when blocking operations occur inside a `synchronized` block or native method.

### Diagnostic JVM Flag
```bash
-Djdk.tracePinnedThreads=short   # Prints stack trace when carrier thread is pinned
-Djdk.tracePinnedThreads=full    # Prints full stack trace including frames
```

### Pinning Fix Pattern
```java
// ❌ BAD: Synchronized block pins carrier thread during I/O
public synchronized String fetchData() {
    return restTemplate.getForObject(url, String.class); // Blocking I/O
}

// ✅ GOOD: ReentrantLock allows carrier thread to unmount during I/O
private final ReentrantLock lock = new ReentrantLock();

public String fetchData() {
    lock.lock();
    try {
        return restTemplate.getForObject(url, String.class);
    } finally {
        lock.unlock();
    }
}
```

---

## 3. JVM Diagnostic CLI Cheat Sheet

| Task | Command | Description |
|---|---|---|
| **Thread Dump** | `jcmd <pid> Thread.print` | Capture full thread stack traces. |
| **Heap Summary** | `jcmd <pid> GC.class_histogram` | Fast top memory consumer inspection. |
| **Heap Dump** | `jcmd <pid> GC.heap_dump /tmp/heap.hprof` | Trigger heap snapshot for Eclipse MAT. |
| **JFR Profile** | `jcmd <pid> JFR.start name=profile settings=profile.jfc duration=60s filename=recording.jfr` | Record 60s Java Flight Recorder profile. |

---

## 4. HikariCP Connection Pool Exhaustion Fix
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 10
      leak-detection-threshold: 2000 # Log warning if connection held > 2000ms
```
- **Rule**: Never execute external REST or HTTP RPC calls inside a `@Transactional` boundary, as it keeps the DB connection open during network latency.
