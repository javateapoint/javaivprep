# System Design Interview Mega Guide

> **Role:** Principal System Architect & Interview Expert  
> **Goal:** Bridge the gap between theory and production-grade distributed systems. This guide teaches you how to *think* like an architect and ace your system design interviews.

---

## 1. Introduction to System Design Interviews

### What System Design Interviews Evaluate
System design interviews are not about finding the "one correct answer." They are an open-ended conversation to evaluate your ability to:
*   **Navigate Ambiguity:** Can you take a vague problem statement (e.g., "Design Twitter") and turn it into a concrete architecture?
*   **Identify Trade-offs:** There are no perfect solutions, only trade-offs. (e.g., SQL vs. NoSQL, Consistency vs. Availability).
*   **Think at Scale:** Do you design for 100 users or 100 million users?
*   **Communicate Effectively:** Can you drive the discussion and articulate *why* you made specific choices?

### Coding vs. LLD vs. System Design (HLD)

| Feature | Coding Interview | Low-Level Design (LLD) | System Design (HLD) |
| :--- | :--- | :--- | :--- |
| **Focus** | Algorithms & Data Structures | Classes, Interfaces, Patterns | Distributed Architecture, Scalability |
| **Scope** | Single Function/Method | Single Service/Component | Entire Platform (Multiple Services) |
| **Key Skill** | Optimization (Time/Space) | Extensibility, Clean Code | Trade-offs, Reliability, Bottlenecks |
| **Output** | Working Code | Class Diagrams, Code Snippets | Architecture Diagrams, Data Flow |

---

## 2. System Design Interview Framework (The "RESHADED" Method)

Use this repeatable framework to structure any design interview:

1.  **R - Requirements Clarification (5-7 mins)**
    *   **Functional:** What does the system *do*? (e.g., Post a tweet, follow user).
    *   **Non-Functional:** What are the *constraints*? (Consistency, Availability, Latency, Reliability).
2.  **E - Estimation (Back-of-the-Envelope) (3-5 mins)**
    *   Scale: DAU/MAU (Daily/Monthly Active Users).
    *   Throughput: QPS (Queries Per Second) for Reads vs. Writes.
    *   Storage: Data needed for 5-10 years.
    *   *Tip:* Keep it rough; orders of magnitude matter more than precision.
3.  **S - System APIs (3-5 mins)**
    *   Define the contract between client and server.
    *   Example: `postTweet(userId, content) -> tweetId`
4.  **H - High-Level Design (HLD) (10-15 mins)**
    *   Draw the "30,000-foot view."
    *   Include: Clients, Load Balancers, API Gateway, App Servers, DBs, Cache, Message Queues.
5.  **A - Architecture & Data Model (Deep Dive) (10-15 mins)**
    *   Schema Design: SQL tables vs. NoSQL documents.
    *   Component details: How does the "Feed Service" actually work?
6.  **D - Detailed Design & Scaling (Remaining Time)**
    *   Identify bottlenecks (Single Point of Failure?).
    *   Scale specific components (Sharding DB, Adding Read Replicas).
7.  **E - Evaluation & Wrap-up**
    *   Summarize the design.
    *   Discuss what you'd improve if you had more time.

---

## 3. Core System Design Concepts (Must-Know)

### Scalability: Vertical vs. Horizontal
*   **Vertical (Scale Up):** Bigger server (More RAM/CPU). Easiest, but has a hard limit (cap) and is a Single Point of Failure (SPOF).
*   **Horizontal (Scale Out):** More servers. Infinite scale, but adds complexity (distributed systems issues).

### Load Balancing
Distributes traffic across servers.
*   **Algorithms:** Round Robin (simple), Least Connections (smart), Consistent Hashing (for caches).
*   **Placement:** Between Client & Web Server, Web Server & App Server, App Server & DB.

### Database Types & CAP Theorem
*   **Relational (SQL):** MySQL, PostgreSQL. Good for structured data, transactions (ACID), complex joins.
*   **NoSQL:**
    *   **Key-Value:** Redis, DynamoDB (High speed, simple lookups).
    *   **Document:** MongoDB (Flexible schema, JSON).
    *   **Wide-Column:** Cassandra, HBase (High write throughput, massive scale).
    *   **Graph:** Neo4j (Social networks, friend graphs).
*   **CAP Theorem:** You can only pick 2: **C**onsistency, **A**vailability, **P**artition Tolerance.
    *   *Real world:* P is mandatory in distributed systems. Choice is usually CP (Bank) vs AP (Social Media).

### Caching
Data access speed: RAM (nanoseconds) >> Disk (milliseconds) >> Network (variable).
*   **Strategies:** Read-Through, Write-Through, Write-Back, Cache-Aside.
*   **Eviction:** LRU (Least Recently Used), LFU (Least Frequently Used).

### Message Queues (Async Processing)
Decouple producers from consumers.
*   **Tools:** Kafka (High throughput, streaming), RabbitMQ (Complex routing).
*   **Use cases:** Processing video uploads, sending emails, generating reports.

---

## 4. Back-of-the-Envelope Estimation (Cheat Sheet)

Memorize these "Power of Two" numbers:
*   `2^10` ≈ 1 KB (Thousand)
*   `2^20` ≈ 1 MB (Million)
*   `2^30` ≈ 1 GB (Billion)
*   `2^40` ≈ 1 TB (Trillion)

**Standard Numbers:**
*   Char: 2 Bytes
*   Long/Int: 8 Bytes / 4 Bytes
*   UUID: 16 Bytes
*   Image: 200 KB (avg)
*   Video: 50 MB / min (HD)

**QPS Formula:**
`QPS = Daily Active Users * Avg Requests per User / 86,400 (seconds in a day)`
*   *Approximation:* 100K seconds in a day (makes math easier).

---

## 5. High-Level Architecture Building Blocks

```text
[Client] -> [CDN] (Static Assets)
    |
    v
[Load Balancer]
    |
    v
[API Gateway] (Auth, Rate Limiting, Routing)
    |
    v
[Service A] <--> [Cache (Redis)]
    |
    v
[Database (Primary)] -> [Replica] (Read-Only)
    |
    v
[Message Queue] -> [Async Workers]
```

---

## 🏗️ System Design Case Study 1: URL Shortener (TinyURL)

### 1. Requirements
*   **Functional:** Shorten Long URL -> Short URL. Redirect Short URL -> Long URL.
*   **Non-Functional:** Highly Available (redirects must work), Low Latency, Non-guessable short links (security).

### 2. Back-of-the-Envelope
*   Writes: 100M URLs / month.
*   Reads: 100:1 ratio (10B redirects / month).
*   Storage: 100M * 5 years * 500 bytes = ~30 TB. NoSQL preferred for easy scaling.

### 3. API Design
*   `createShortUrl(longUrl, apiKey) -> shortUrl`
*   `getOriginalUrl(shortUrl) -> longUrl` (HTTP 301 vs 302 Redirect)

### 4. Data Model
*   **DB:** NoSQL (Key-Value like DynamoDB or Riak).
*   **Schema:** `Hash(PK)`, `OriginalURL`, `CreationDate`, `UserId`.

### 5. High-Level Design
*   **Hash Function:** MD5/SHA-256 (too long) vs. Base62 conversion.
*   **Collision Handling:**
    *   *Solution 1:* Hash + collision check (expensive).
    *   *Solution 2 (Better):* Distributed ID Generator (Snowflake) -> Base62 Encode. No collisions guaranteed.

### 6. Scaling & Bottlenecks
*   **Reads:** Heavy caching (Redis/Memcached). 99% of requests serve from cache.
*   **Writes:** KGS (Key Generation Service) - Pre-generate keys and store in DB. App servers fetch a batch of keys.

---

## 🏗️ System Design Case Study 2: Chat System (WhatsApp)

### 1. Requirements
*   **Functional:** 1-on-1 Chat, Group Chat, Sent/Delivered/Read receipts, Online status.
*   **Non-Functional:** Low latency (real-time), consistency (order matters), high availability.

### 2. Architecture Choices
*   **Protocol:** HTTP is too slow (polling). Use **WebSockets** for bi-directional persistent connection.
*   **DB:** Write-heavy. HBase or Cassandra (LSM Trees optimize for writes).

### 3. High-Level Design
*   **Chat Server:** Maintains WebSocket connections.
*   **Session Service:** "Who is connected to which server?" (Redis).
*   **Message Service:** Persist messages.
*   **Push Notifications:** If user is offline, send via GCM/APNS.

### 4. Detailed Design: Message Flow (User A -> User B)
1.  User A sends msg to Chat Server 1.
2.  Server 1 persists msg to DB (via Message Service).
3.  Server 1 asks Session Service: "Where is User B?"
4.  If B is online (Server 2): Server 1 -> Server 2 -> Push to B over WebSocket.
5.  If B is offline: Push to Notification Service.

---

## 🏗️ System Design Case Study 3: Rate Limiter

### 1. Requirements
*   **Goal:** Prevent abuse. Limit: 10 requests / sec / IP.
*   **Accuracy vs. Performance:** High performance is critical; slightly loose accuracy is acceptable.

### 2. Algorithms
*   **Token Bucket:** Efficient, handles bursts. (Standard choice).
*   **Leaky Bucket:** Smooths out bursts (constant rate).
*   **Fixed Window Counter:** Edge case issues (burst at window boundary).
*   **Sliding Window Log:** Accurate but memory expensive.
*   **Sliding Window Counter:** Best balance of memory & accuracy.

### 3. Distributed Implementation
*   Local memory? No, won't work for multiple app servers.
*   **Redis:** Use centralized Redis with Lua scripts (for atomicity).
    *   *Key:* `User_ID + TimeWindow`
    *   *Value:* Counter.

---

## ⚙️ Distributed Systems Deep Dive

### Data Consistency
*   **Strong Consistency:** User sees latest write immediately. (SQL, Paxos/Raft). Latency hit.
*   **Eventual Consistency:** User sees latest write *eventually* (seconds/minutes). (DNS, Cassandra). High availability.

### Distributed Transactions (Saga Pattern)
Microservices split DBs, so no ACID across services.
*   **Choreography:** Services emit events; others listen and act.
*   **Orchestration:** Central coordinator tells services what to do. (Easier to manage).

---

## 🧠 Most Asked System Design Questions (Quick Fire)

| Question | Key Focus | Red Flags |
| :--- | :--- | :--- |
| **Design Twitter** | Fan-out (Push vs Pull model for feeds), Hybrid approach for celebrities. | Using SQL for everything. Ignoring "Justin Bieber" problem (Hot keys). |
| **Design Youtube** | Storage (Blob vs Metadata), CDN, Adaptive Bitrate Streaming. | Storing videos in DB. Not discussing CDN. |
| **Design Uber** | Geo-hashing (QuadTree / Google S2), High write volume for driver locations. | Updating DB for every location ping (too slow). |
| **Design Google Drive** | Block storage (splitting files), Deduplication, Sync conflict resolution. | Uploading whole files on every edit. |

---

## 🧾 Final System Design Cheat Sheet

*   **Read-Heavy?** -> Cache + Read Replicas.
*   **Write-Heavy?** -> Sharding + NoSQL (Cassandra/LSM Trees) + Async MQ.
*   **Search?** -> ElasticSearch / Inverted Index.
*   **Media?** -> CDN + Blob Storage (S3).
*   **Transactions?** -> SQL (ACID).
*   **Analytics?** -> Columnar DB (Redshift/Clickhouse) + Batch Processing (Spark).

---

**Document Version:** 1.0  
**Target Audience:** Senior Engineers, Architects, Interview Candidates  
**Usage:** GitHub Repository / Interview Prep Guide