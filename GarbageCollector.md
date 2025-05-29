# 1. JVM Runtime Data Areas
### Heap

Shared among all threads; stores all objects and arrays.

Divided into generations (young + old) for most collectors.

### Stack

One per thread; holds local variables, partial results, and frames.

### Metaspace

Replaces PermGen (since Java 8); stores class metadata off-heap.

### Code Cache

JIT-compiled native code, runtime stubs.

### Native/Direct Memory

Used by NIO, off-heap libraries, JNI.



# 2. Generational Heap & Collection Phases
### Young Generation

Eden + two Survivor spaces.

Minor GC: fast, stop-the-world copy/evacuation.

### Old (Tenured) Generation

Objects that survive multiple minors.

Major/Full GC: varies by collector (mark-compact, mark-sweep, evacuation).

### GC Roots

Local variables, active threads, static fields, JNI references.

### Typical GC Phases

Mark: identify live objects from roots.

Sweep/Compact/Evacuate: reclaim or relocate unreachable ones.

Restore: update references, resume threads.




| Collector                    |  Generational ? |             Pause Target             | Throughput ✓ |   Footprint Overhead   |                    Availability                   |
| ---------------------------- | :-------------: | :----------------------------------: | :----------: | :--------------------: | :-----------------------------------------------: |
| **Serial GC**                |       Yes       |            Single-threaded           |     High     |           Low          |                   All platforms                   |
| **Parallel (Throughput) GC** |       Yes       | Controlled by `-XX:MaxGCPauseMillis` |   Very High  |         Medium         |                   All platforms                   |
| **CMS (conc-mark-sweep)**    |       Yes       |        Low pause, non-compact        |    Medium    | Higher (fragmentation) | **Removed in JDK 14**                             |
| **G1 GC**                    |       Yes       |    User-tunable (default \~200 ms)   |     High     |           Low          |       Default since Java 9                        |
| **ZGC**                      |       Yes       |            < 1 ms (sub-ms)           |     High     |           Low          |      Production in Java 15                        |
| **Shenandoah**               |       Yes       |            < 1 ms (sub-ms)           |     High     |           Low          |        Production in Java 15                      |
| **Epsilon (No-Op)**          | No (alloc only) |         N/A (crashes on OOM)         |      N/A     |          None          |     Experimental since Java 11                    |

#### Note: CMS was deprecated in Java 9 and removed in Java 14 (JEP 363)




# 4. Collector Deep-Dive
### 4.1 Serial GC
Single-threaded mark-copy in young, mark-compact in old.

Use: small heaps, single-CPU.

### 4.2 Parallel GC
Multi-threaded young and old collections.

Flags: -XX:+UseParallelGC, tune with -XX:ParallelGCThreads, -XX:MaxGCPauseMillis.

### 4.3 CMS (Concurrent Mark-Sweep)
Concurrent mark + remark, brief pauses at start/remark phases.

Drawbacks: fragmentation (no compaction), many tuning flags.

Removed: JEP 363

### 4.4 G1 GC (Garbage-First)
Region-based, concurrent marking + evacuation; compacts to avoid fragmentation.

Features: pause-time goals, heap-region free list, humongous region handling.

Improvements:

NUMA-aware region allocation (JEP 345)

Return unused heap regions promptly (JEP 346)

### 4.5 ZGC
Uses colored pointers, concurrent relocation, sub-millisecond pauses.

Production-ready in Java 15 (JEP 377).

JEP 376 (JDK 16): Concurrent thread-stack processing removes stack-scan safepoints, drives pauses to ~100 μs

### 4.6 Shenandoah
Brooks-style barriers, concurrent evacuation.

Production-ready in Java 15 (JEP 379)

### 4.7 Epsilon GC
No-op: allocates only, never reclaims; useful for latency benchmarking and footprint analysis.

Crash once heap is exhausted.

Enable: -XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC






# Recent Enhancements in Java 17

ZGC concurrent stacks (JEP 376) lowered pause-set overhead by scanning stacks outside safepoints 

Shenandoah product (JEP 379) — no experimental flags 

Elastic Metaspace (JEP 387) returns unused class-metadata pages to the OS 

G1 refinements (NUMA: JEP 345; uncommit: JEP 346) improve footprint and mapping locality.


# 6. GC Tuning at a Glance
Heap sizing: -Xms/Xmx to control min/max heap.

Thread counts: -XX:ParallelGCThreads, -XX:ConcGCThreads.

Pause goals: -XX:MaxGCPauseMillis, -XX:InitiatingHeapOccupancyPercent (G1).

Logging: -Xlog:gc* for unified GC logs.

Region sizes: -XX:G1HeapRegionSize (G1).

Barrier costs: relevant for ZGC/Shenandoah.


# 7. Senior-Level GC Interview Q&A
Q: Explain the differences between G1 and CMS.
A: G1 is region-based and compacting, gives predictable pause targets and handles fragmentation smoothly; CMS is mark-sweep only, prone to fragmentation and longer tuning cycles 


Q: How does ZGC achieve sub-millisecond pauses?
A: Through colored pointers for concurrent relocation, barrier insertion to track pointer mutations, and off-safepoint stack scanning via JEP 376 


Q: When would you use Epsilon GC?
A: For performance baselining (to isolate GC overhead), memory-footprint profiling, ultra-low latency batch jobs, or very short-lived microservices 


Q: What is Elastic Metaspace and why is it important?
A: JEP 387 allows the JVM to return unused class-metadata pages to the OS, reducing long-running process footprint and improving container friendliness 


Q: How do you tune G1 for large heaps with NUMA?
A: Enable G1’s NUMA support (-XX:+G1UseNUMA), adjust region size, set -XX:ParallelGCThreads per socket, and monitor interleaving and affinity.

Q: Describe the GC root types and how they are processed.
A: JVM stack frames, static references, JNI handles, GC handles; collectors mark from these roots—some concurrently (G1/ZGC/Shenandoah), some stop-the-world (Serial/Parallel).




# Focused comparison of G1 vs. ZGC



| Aspect                     | G1 GC                                                                                                | ZGC                                                                                                   |
| -------------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Design Goal**            | Balanced throughput & pause-time targeting                                                           | Ultra-low latency (sub-millisecond pauses)                                                            |
| **Heap Organization**      | Region-based (1–32 MB regions), mixed young/old within same space                                    | Region-based (4 KB–1 MB pages), colored-pointer marking                                               |
| **Pause Strategy**         | “Garbage-First”: pause to collect most-occupied regions; concurrent marking + evacuation             | Concurrent relocation & marking; pauses only for safepoints (< 1 ms)                                  |
| **Concurrent Phases**      | - Concurrent marking<br>- Evacuation during STW pause segments                                       | - Concurrent marking<br>- Concurrent relocation<br>- Concurrent stack processing (JEP 376)            |
| **Throughput vs. Latency** | Good throughput, tunable pause goals (default \~200 ms)                                              | Sacrifices a bit of throughput for guaranteed sub-ms latency                                          |
| **Footprint & Overhead**   | - Some extra heap overhead for regions<br>- Moderate barrier costs                                   | - Low extra footprint<br>- Barrier overhead on every pointer write/read                               |
| **Memory Reclamation**     | Evacuates live objects to free regions; returns regions to OS (JEP 346)                              | Automatically returns unused pages to OS                                                              |
| **Fragmentation Handling** | Compacts via evacuation; no long-term fragmentation                                                  | Always relocates objects concurrently; no fragmentation                                               |
| **Tuning Flags**           | `-XX:+UseG1GC`, `-XX:MaxGCPauseMillis`, `-XX:InitiatingHeapOccupancyPercent`, `-XX:G1HeapRegionSize` | `-XX:+UseZGC` (no unlock flags), optional `-XX:ZCollectionIntervalMillis`, `-XX:ZUncommitDelayMillis` |
| **Barrier Cost**           | Minimal (mostly on collection phases)                                                                | Read/write/write-load barriers on every pointer access; optimized to nanoseconds per barrier          |
| **Supported Platforms**    | All major platforms; default since Java 9                                                            | All major 64-bit platforms since JDK 15; production-ready in Java 17                                  |
| **Maturity & Adoption**    | Widely used in production for “general” workloads                                                    | Increasing adoption where ultra-low latency is critical (finance, telecom)                            |







### 1. Collection Mechanics
#### G1 divides the heap into fixed-size regions.

Young GC: evacuates live objects from Eden + one Survivor region into Survivor or old regions.

Mixed GC: after a threshold of old-gen occupancy, G1 will also evacuate selected old regions in the same pause, targeting regions with the most reclaimable space.

#### ZGC uses a colored-pointer scheme: each object reference carries color bits.

Concurrent marking builds liveness maps.

Concurrent relocation moves objects to new memory addresses without stopping all threads, updating pointers via read/write barriers.

Stack processing (JEP 376) scans and relocates roots off-safepoints, keeping pauses under 1 ms.

### 2. Pause Behavior
G1 pauses are driven by region evacuation: you can tune your target max pause (-XX:MaxGCPauseMillis), but actual pauses typically range tens to low hundreds of ms depending on heap size and live data footprint.

ZGC guarantees sub-millisecond pauses regardless of heap size, since the bulk of work is concurrent. Actual pauses are usually in the 100–500 μs range, even on multi-terabyte heaps.

### 3. Tuning & Diagnostics
#### G1

Key flags:

-XX:MaxGCPauseMillis=...

-XX:InitiatingHeapOccupancyPercent=... (when to start mixed GCs)

-XX:G1HeapRegionSize=...

Logs: -Xlog:gc*,gc+heap=debug

#### ZGC

Key flags:

-XX:ZCollectionIntervalMillis=... (periodic concurrent collections)

-XX:ZUncommitDelayMillis=... (when to return free pages to OS)

Logs: -Xlog:gc*,gc+relocation=debug

### 4. When to Use Which

| Scenario                           | Recommended GC          |
| ---------------------------------- | ----------------------- |
| Throughput-sensitive batch jobs    | G1                      |
| General-purpose web/microservices  | G1 (default)            |
| Ultra-low latency (HFT, messaging) | ZGC                     |
| Large heaps (≥ 100 GB)             | ZGC (consistent pauses) |


### Bottom Line:

G1 remains the go-to “all-rounder” collector: easy to tune, reasonably low pauses, good throughput.

ZGC is the choice when predictable ultra-low latency matters more than marginal throughput. Its concurrent architecture scales to very large heaps while keeping pauses imperceptible.
