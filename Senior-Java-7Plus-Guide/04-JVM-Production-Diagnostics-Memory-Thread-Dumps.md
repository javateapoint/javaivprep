# JVM Production Diagnostics: Memory Leaks, Thread Dumps & Profiling Master Guide

A production-grade reference guide for Senior Java Engineers (7+ YOE) covering **RCA for `OutOfMemoryError`**, **Eclipse MAT heap dump analysis**, **Thread Dump analysis for deadlocks & CPU lockups**, **mapping Linux `top -H` to `jstack` thread IDs**, and zero-overhead profiling with **JFR (Java Flight Recorder)** and **async-profiler**.

---

## 1. Categorizing Production `OutOfMemoryError` (OOM)

Not all `OutOfMemoryError` crashes are caused by heap memory exhaustion. Senior engineers must immediately distinguish between the 4 major OOM variants:

```text
                                  ┌── 1. java.lang.OutOfMemoryError: Java heap space
                                  │      (Objects in Young/Old Gen exceeding -Xmx)
                                  │
                                  ├── 2. java.lang.OutOfMemoryError: Metaspace
                                  │      (Class metadata, dynamic proxies, byte-buddy leak)
  Production OutOfMemoryError ────┼──
                                  ├── 3. java.lang.OutOfMemoryError: Direct buffer memory
                                  │      (Netty / Off-heap NIO ByteBuffers exceeding -XX:MaxDirectMemorySize)
                                  │
                                  └── 4. java.lang.OutOfMemoryError: unable to create new native thread
                                         (OS process thread limit ulimit / memory exhaustion)
```

### 1.1 Summary & Fix Actions

| OOM Variant | Root Cause | Fix Action |
| :--- | :--- | :--- |
| **`Java heap space`** | Unbounded in-memory collections, unevicted caches, missing pagination. | Generate heap dump via `-XX:+HeapDumpOnOutOfMemoryError` and analyze retained memory in Eclipse MAT. |
| **`Metaspace`** | Classloader leak (dynamic class generation via Spring CGLIB, reflection, un-unloaded plugins). | Increase `-XX:MaxMetaspaceSize` and inspect classloader counts via `jcmd <pid> VM.classloaders`. |
| **`Direct buffer memory`** | Off-heap NIO `ByteBuffer.allocateDirect()` leaks (Netty/WebClient unreleased buffers). | Tune `-XX:MaxDirectMemorySize` and check Netty resource leak detection `-Dio.netty.leakDetection.level=PARANOID`. |
| **`unable to create native thread`** | Thread pool misconfiguration spawning thousands of OS platform threads. | Refactor to Java 21 Virtual Threads or tune Linux `ulimit -u` thread creation bounds. |

---

## 2. Analyzing Heap Dumps with Eclipse MAT

### Recommended JVM Flags for Production

```bash
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/java/heapdump.hprof \
     -XX:+ExitOnOutOfMemoryError \
     -jar application.jar
```

### 2.1 Key Eclipse Memory Analyzer (MAT) Concepts

- **Shallow Heap**: Memory consumed by the object **itself** (e.g. primitive fields and 4/8-byte object reference pointers).
- **Retained Heap**: Total memory that will be freed if this object is garbage collected (includes all references exclusively reachable from this object).
- **Dominator Tree**: Displays the largest retained objects in the JVM heap, highlighting candidate leak roots.
- **Incoming vs. Outgoing References**:
  - *Outgoing References*: Objects referenced by the current object.
  - *Incoming References*: Objects pointing to the current object (helps trace GC Roots!).

### Common Heap Memory Leak Culprits
1. **ThreadLocal Misuse**: Forgetting to invoke `ThreadLocal.remove()` in a thread-pooled environment (e.g., Tomcat thread pool reusing threads across requests).
2. **Unbounded Static Caches**: `HashMap` used as a cache without TTL or LRU eviction policies (replace with Caffeine Cache).
3. **Unclosed Event Listeners**: Subscribing to long-lived singleton event publishers without unsubscribing on component destruction.

---

## 3. RCA: Mapping High-CPU Threads with `top -H` and `jstack`

### Real Scenario
Production CPU spikes to 100%. An API endpoint hangs indefinitely. How do you find the exact line of Java code causing the CPU lockup?

```text
Step 1: Identify High-CPU Process ID (PID)
$ top

Step 2: Identify Specific High-CPU Lightweight Process (LWP / Thread ID)
$ top -H -p <PID>
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
14205 appuser   20   0 8540320 2.1g  18400 R  98.5  13.2   5:12.34 java

Step 3: Convert Decimal LWP ID (14205) to Hexadecimal Notation
$ printf "%x\n" 14205
377d

Step 4: Capture Thread Dump and Grep for Hex Thread ID (`nid=0x377d`)
$ jcmd <PID> Thread.print > thread_dump.txt
# OR
$ jstack <PID> | grep -A 30 "nid=0x377d"
```

### Sample Output Pinpointing Code Location

```text
"http-nio-8080-exec-4" #42 daemon prio=5 os_prio=0 tid=0x00007f8a10014800 nid=0x377d runnable [0x00007f8a1c3e4000]
   java.lang.Thread.State: RUNNABLE
	at com.example.demo.service.SearchService.regexMatch(SearchService.java:87)
	at com.example.demo.service.SearchService.processQuery(SearchService.java:45)
	...
```
*Result*: Line 87 of `SearchService.java` is executing a catastrophic backtracking regular expression in a tight loop!

---

## 4. Identifying Deadlocks in Thread Dumps

A **Deadlock** occurs when Thread A holds Lock 1 and waits for Lock 2, while Thread B holds Lock 2 and waits for Lock 1.

```text
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f311c004a80 (object 0x000000076bf12340, a java.lang.Object),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f311c004be0 (object 0x000000076bf12380, a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
	at com.example.demo.DeadlockDemo.lambda$0(DeadlockDemo.java:18)
	- waiting to lock <0x000000076bf12340> (a java.lang.Object)
	- locked <0x000000076bf12380> (a java.lang.Object)
"Thread-2":
	at com.example.demo.DeadlockDemo.lambda$1(DeadlockDemo.java:28)
	- waiting to lock <0x000000076bf12380> (a java.lang.Object)
	- locked <0x000000076bf12340> (a java.lang.Object)
```

---

## 5. Low-Overhead Production Profiling

### 5.1 Java Flight Recorder (JFR)

JFR is built directly into the HotSpot JVM with **< 1% overhead**, making it safe for 24/7 production use.

```bash
# Start a 60-second JFR recording on a running production JVM
jcmd <PID> JFR.start name=prod_profile settings=profile.jfc duration=60s filename=/tmp/profile.jfr

# Dump recording
jcmd <PID> JFR.dump name=prod_profile filename=/tmp/profile.jfr
```
*Inspect `/tmp/profile.jfr` using JDK Mission Control (JMC).*

### 5.2 async-profiler (Flame Graphs)

`async-profiler` avoids Safepoint Bias (a flaw in standard Java thread profiling tools) by using Linux `perf_events`.

```bash
# Profile CPU usage for 30 seconds and output Flame Graph HTML
./asprof -d 30 -f /tmp/flamegraph.html <PID>
```

---

## 💡 Senior Interview Summary Tips

1. **Always know how to convert Linux thread ID to hex `nid`**: `printf "%x\n" <LWP_ID>` to find high-CPU threads in a thread dump.
2. **Never run `jmap -dump` on a production heap without caution**, as it pauses all application threads. Use `jcmd <PID> GC.heap_dump` or trigger dumps on OOM automatically.
3. **Use Caffeine Cache instead of `ConcurrentHashMap`** for caches to prevent `Java heap space` OOM crashes.
