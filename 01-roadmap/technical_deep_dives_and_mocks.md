# Comprehensive Technical Deep Dives, System Design, and Behavioral Interview Guide
**Target Role:** Senior Full-Stack Product Engineer at **Intelligent Machines (IM)**  
**Candidate:** Mobarak Hosen (10+ Years Experience | Software Architect & CSM®)

This guide provides deep technical concepts, architecture considerations, mock questions with answers, and step-by-step guides for the critical technologies, backend patterns, frontend concepts, system design scenarios, and behavioral questions tailored specifically to your resume and IM's needs.

---

## SECTION 1: TECHNICAL PREPARATION & DEEP DIVES

### 1. Modern Backend Stack: Node.js/TypeScript & Python vs. Java/Spring Boot
Intelligent Machines values language agnosticism, but the JD explicitly highlights a modern stack: **React or Next.js, Node.js or Python, PostgreSQL or MySQL, and Cloud (GCP/AWS)**. Your resume highlights deep expertise in **Java and Spring Boot** and **APISIX/Gateway Lua**. 

#### Runtime Architectures Comparison

```
+------------------------------------------+------------------------------------------+
| Node.js (V8 + libuv)                     | Java Virtual Machine (JVM)               |
+------------------------------------------+------------------------------------------+
| - Single-threaded event loop.            | - Multi-threaded runtime.                |
| - Non-blocking asynchronous I/O.         | - Thread-per-request model (Tomcat) or   |
| - Excellent for high-concurrency,       |   Virtual Threads (Project Loom).        |
|   I/O-heavy operations (microservices).   | - Excellent for heavy computation, complex|
| - Lower memory footprint at startup.     |   business logic, and strict typing.     |
+------------------------------------------+------------------------------------------+
```

#### Core Concepts for Node.js/TypeScript
* **Event Loop Phases:** Timer phase, Pending callbacks, Idle/Prepare, Poll (I/O execution), Check (setImmediate), Close callbacks. Understanding `process.nextTick()` and `Promise.resolve()` (Microtask queue) executing between phases.
* **Concurrency in a Single Thread:** Node.js executes your JavaScript in a single thread, but I/O operations are offloaded to the operating system or the libuv thread pool (default size: 4).
* **Memory Management:** V8 Garbage Collection uses Scavenge (young generation) and Mark-Sweep-Compact (old generation). Memory leaks can be caused by global variables, unresolved promises, or uncleared intervals.
* **TypeScript Best Practices:** strict mode enabled, custom type guards, utility types (`Omit`, `Pick`, `Partial`, `Record`), avoiding `any` in favor of `unknown`, structural typing.

#### Real-World Architecture & Performance Concerns
* **CPU-Bound Tasks in Node.js:** If your service executes heavy CPU tasks (e.g. cryptography, image compression, heavy data processing for bKash analytics), it blocks the event loop, freezing all other concurrent requests.
* **Mitigation:** Offload CPU-bound tasks to **Worker Threads** (using `worker_threads` module) or delegate them to a separate microservice (written in Python/Java).
* **Java Migrations:** At BJIT, you migrated Rakuten Beauty monolith to Java microservices. Translate that thought process to Node.js/TypeScript: *"We identify resource-heavy paths. For network-bound proxies and gateways, we use Node.js. For raw transaction processing and heavy billing calculations, we use services with strict compiler checks."*

#### Mock Question 1: "Explain how you handle asynchronous errors in Node.js/TypeScript. How do you prevent unhandled promise rejections in production?"
* **Answer:**
  > "In Node.js, an unhandled promise rejection can crash the process or leave it in an unstable state. I prevent this by enforcing:
  > 1. **Global Process Handlers:** Registering listeners for `unhandledRejection` and `uncaughtException` in the entrypoint file to log the error context using Winston or Pino and gracefully exit the container (allowing Kubernetes to restart it).
  > 2. **Express/Koa Middleware:** Using async wrapper middleware or native Express 5 handlers that automatically catch rejected promises in controllers and pass them to a global error-handling middleware.
  > 3. **Zod Validation & Try-Catch:** Validating all incoming payloads with Zod at the routing boundary. For downstream async actions (database queries, Kafka publishing), wrapping them in try-catch blocks and using typed Custom Error classes to map errors to proper HTTP response codes."

---

### 2. Database Optimization & PostgreSQL Internals
Since IM handles scale for clients like bKash and Unilever, database performance is critical.

#### PostgreSQL Core Internals
* **ACID Transactions & MVCC (Multi-Version Concurrency Control):** PostgreSQL doesn't lock tables during reads. Instead, every transaction sees a snapshot of the data. When a row is updated, PostgreSQL writes a new version (tuple) and marks the old one as dead.
* **Vacuuming:** The autovacuum daemon cleans up dead tuples to prevent table bloat and index degradation.
* **Locking Mechanisms:**
  * **Row-level locks:** Shared (`FOR SHARE`) and Exclusive (`FOR UPDATE`).
  * **Table-level locks:** Access Share (reads), Row Exclusive (writes), and Access Exclusive (DDL modifications like `ALTER TABLE`).
* **Connection Pooling:** PostgreSQL spawns a separate OS process for every connection, which is memory-expensive (~10MB/conn). Using **PgBouncer** in front of PostgreSQL pools connections and prevents connection starvation, especially under serverless AWS Lambda / Cloud Run scales.

#### Mock Question 2: "Suppose we have a table `tenant_orders` containing 100M rows. Users complain that fetching orders by `tenant_id` sorted by `created_at` DESC is extremely slow. Explain your diagnostic steps and how you resolve it."
* **Answer:**
  > "First, I run an `EXPLAIN (ANALYZE, BUFFERS)` on the slow query to examine the execution plan.
  > 1. **Check for Seq Scan:** If I see a `Seq Scan` (Sequential Scan) on `tenant_orders`, it means the query planner is scanning the entire table.
  > 2. **Review Caching:** Look at the `Buffers` output. If `shared read` is high and `shared hit` is low, the query is hitting the disk repeatedly.
  > 3. **Create a Composite Index:** A single index on `tenant_id` is insufficient because the database still has to sort the results by `created_at`. I would create a composite index:
  >    `CREATE INDEX CONCURRENTLY idx_orders_tenant_created ON tenant_orders (tenant_id, created_at DESC);`
  >    The `CONCURRENTLY` keyword ensures PostgreSQL builds the index without acquiring a write lock on the table, preventing production downtime.
  > 4. **Check work_mem:** If the query plan shows a `Sort Method: external merge Disk`, it means the sorting operation exceeds the allocated `work_mem` and is spilling to disk. I would safely adjust `work_mem` for that session.
  > 5. **Table Partitioning:** If a single tenant's data grows excessively, I would propose partitioning the table by Range on `created_at` or Hash on `tenant_id` to ensure queries only scan relevant partitions."

---

## SECTION 2: BACKEND ARCHITECTURE & SCALABILITY DEEP DIVE

### 1. API Gateway Patterns & Security (APISIX & WAF)
You have direct, hands-on experience building an API Gateway handling **10M+ daily requests** using **APISIX and Coraza WAF**. Highlight this during interviews.

#### Architectural Principles of a Gateway
* **Routing & Dynamic Upstreams:** Gateway acts as a reverse proxy, decoupling client-facing URLs from internal microservice endpoints. Dynamic routing avoids hardcoded upstream configurations.
* **Authentication Offloading:** Validating JWT tokens or HMAC signatures at the gateway layer, injecting sanitized user/tenant headers (`X-Tenant-ID`, `X-User-Role`) before forwarding requests.
* **WAF (Web Application Firewall):** Integrating Coraza WAF compiled as a plugin to inspect payloads against OWASP Core Rule Set (CRS) to block SQL injections, Cross-Site Scripting (XSS), and path traversal attacks at the perimeter.

```
[ Client Request ] ──► [ APISIX Gateway ] ──► [ Coraza WAF Check ] ──► [ Auth & Rate Limit ] ──► [ Internal Service ]
```

#### Mock Question 3: "How do you implement Dynamic Rate Limiting at the API Gateway level based on different tenant subscription tiers?"
* **Answer:**
  > "In my API Gateway architecture at TechnoNext, I implemented dynamic rate limiting using Apache APISIX and Redis.
  > 1. **Tenant Lookup:** When a request arrives, the gateway extracts the API Key or JWT, fetches the tenant metadata (tier, rate limit quota) from an in-memory L1 cache (local LRU cache in the worker process with a 10s TTL to prevent hitting Redis on every request).
  > 2. **Redis Rate Check:** If the metadata is not in L1, the gateway queries the Redis cluster (L2 cache). It executes a Redis Lua script implementing the **Token Bucket** algorithm.
  > 3. **Lua Execution:** The script receives a key like `limit:tenant_abc:tier_gold` and parameters for bucket capacity and refill rate. It evaluates whether the tenant has remaining quota, decrements the token, and updates the bucket state in Redis atomically.
  > 4. **Response Headers:** The gateway injects rate limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) into the HTTP response. If the bucket is empty, it short-circuits the request and returns an HTTP `429 Too Many Requests` status."

---

### 2. Caching Strategies & Stampede Mitigation
Caching is crucial to reduce database load and latency, but poorly implemented caching leads to stale data or outages.

#### Caching Strategies
* **Cache-Aside:** Application queries cache first. On miss, it reads from DB, writes to cache, and returns. (Best for general read-heavy workloads).
* **Write-Through:** Application writes to cache, cache writes to database synchronously. (Ensures cache consistency but adds write latency).
* **Write-Behind (Write-Back):** Application writes to cache, cache writes to database asynchronously. (High write throughput, but risks data loss if the cache node crashes).

#### Cache Stampede (Thundering Herd)
Under high concurrency, when a popular cache key expires, multiple request threads simultaneously query the database to rebuild the cache, saturating DB connections.

#### Mitigation Strategies
1. **Mutex Locking (Single Recommender):** Use a distributed lock (Redis `SETNX`) on the cache key. Only the first thread that acquires the lock queries the database and updates the cache; all other threads sleep and retry.
2. **Probabilistic Early Expiration (XFetch):** Background threads recalculate the cache before it officially expires based on a probabilistic calculation of request density.

---

### 3. Distributed Messaging & Concurrency (Kafka vs. RabbitMQ)
You have worked with both Kafka and RabbitMQ. You need to explain when and why to use each.

#### Comparative Use Cases at IM

```
+------------------------------------------+------------------------------------------+
| Apache Kafka                             | RabbitMQ                                 |
+------------------------------------------+------------------------------------------+
| - Commit Log architecture.               | - Message Broker (AMQP).                 |
| - High throughput (Millions/sec).        | - Complex routing (Exchange bindings).   |
| - Message replayability (offsets).       | - Message deleted on ack.                |
| - Best for: Ingestion pipelines,         | - Best for: Background tasks, email      |
|   analytics, audit trails, event sourcing|   notifications, transactional jobs.     |
+------------------------------------------+------------------------------------------+
```

#### The Transactional Outbox Pattern
How do you update a database and publish an event to Kafka in a single operation without risking distributed failures?

```
[ Create Order API ]
        │
        ▼
[ DB Transaction ] ──► (Update orders table & Insert into outbox table) ──► [ DB Commit ]
                                                                                │
                                                                                ▼
[ CDC / Debezium / Poller ] ◄───────────────────────────────────────────────────┘
        │
        ▼
[ Publish to Kafka ]
```

1. **Local DB Transaction:** Write your business data (e.g. `orders`) and write an event payload to an `outbox` table in the same database using a single ACID transaction.
2. **CDC/Outbox Reader:** A background worker (using Debezium for Change Data Capture or a simple database poller) reads the `outbox` table and publishes the messages to Kafka.
3. **Delete/Mark Processed:** Once Kafka acknowledges receipt, the worker deletes the outbox record or marks it as processed. This guarantees **At-Least-Once Delivery** to Kafka.

---

## SECTION 3: FRONTEND DEEP DIVE (React & Next.js)

### 1. Next.js App Router & Rendering Strategies
Since IM uses React and Next.js, you must be prepared to discuss Next.js 14+ App Router, Server Components, and data fetching.

#### Rendering Paths
* **Static Site Generation (SSG) / Static Rendering:** Pages are pre-rendered at build time. Best for static content (documentation, landing pages).
* **Server-Side Rendering (SSR) / Dynamic Rendering:** Pages are rendered on the server on every request. Best for user dashboards with dynamic, personalized data.
* **Incremental Static Regeneration (ISR):** Pre-renders static pages but updates them in the background after a specified duration (`revalidate` tag).
* **React Server Components (RSC) vs. Client Components:**
  * RSC (default) render on the server, producing raw HTML. They send 0KB of JS to the browser.
  * Client Components (declared with `'use client'`) are hydrated in the browser. Use them only when using hooks (`useState`, `useEffect`) or browser-only APIs.

#### State Management Blueprint for SaaS Dashboards
* **Server State:** Use **TanStack Query (React Query)** to handle remote API fetching, client-side caching, pagination state, and automatic cache invalidation.
* **Global Client State:** Use **Zustand** for lightweight client-only states (e.g. sidebar toggle, dark mode, persistent search filters). Avoid wrapping the entire application in context providers if not needed.

---

### 2. Frontend Performance & Core Web Vitals Optimization
You must show an understanding of how client-side performance affects business outcomes (LCP, INP, CLS).

#### Web Vitals Diagnostics

```
┌──────────────────────────┬──────────────────────────┬──────────────────────────┐
│ LCP (Largest Contentful)  │ INP (Interaction to Next)│ CLS (Cumulative Layout)  │
├──────────────────────────┼──────────────────────────┼──────────────────────────┤
│ - Measures loading speed.│ - Measures page response.│ - Measures visual        │
│ - Target: < 2.5 seconds. │ - Target: < 200 ms.      │   stability of layout.   │
│                          │                          │ - Target: < 0.1.         │
└──────────────────────────┴──────────────────────────┴──────────────────────────┘
```

#### Diagnostic & Optimization Steps
1. **LCP Fixes:**
   * Use Next.js `next/image` to automatically format, compress, and resize images.
   * Add `priority` attribute to images above the fold to trigger preloading.
   * Eliminate render-blocking resources by deferring non-critical scripts.
2. **INP Fixes:**
   * Break up long JavaScript execution blocks (> 50ms) using `requestIdleCallback` or React's `useTransition`.
   * Defer tracking scripts and analytics integrations using Next.js `Script` component with `strategy="lazyOnload"`.
3. **CLS Fixes:**
   * Define explicit width and height dimensions for all image and video elements.
   * Use skeleton loaders with reserved dimensions for dynamic dashboard charts and lists to prevent content layout shifts.

---

## SECTION 4: SYSTEM DESIGN PREPARATION

### Scenario 1: Multi-Tenant Enterprise B2B SaaS Platform (Map Phantom / IM SaaS)

#### 1. Requirements
* **Functional:**
  * Tenant registration and workspace onboarding.
  * API key generation, rotation, and revocation.
  * Tiered usage metering based on API calls.
  * Tenant admin consumption reports and billing dashboard.
* **Non-Functional:**
  * API gateway authentication overhead under 5ms.
  * Strict data isolation (zero cross-tenant data leakage).
  * Scalability up to 100M daily API events.
  * Cost-aware infrastructure footprint.

#### 2. High-Level Architecture (ASCII)

```
                       [ Enterprise Client Applications ]
                                      │
                                      ▼
                        [ Load Balancer (AWS ALB) ]
                                      │
                                      ▼
                    [ API Gateway (Apache APISIX Pods) ]
                                      │
           ┌──────────────────────────┴──────────────────────────┐
           ▼                                                     ▼
 [ In-Memory L1 Cache ]                                  [ Auth Service ]
 (10s local TTL for keys)                                (Verifies HMAC signatures)
           │                                                     │
           ▼                                                     ▼
[ Redis Cluster (L2 Cache) ] ◄───────────────────────────────────┘
(Token Bucket Rates, Key Expiry)
           │
           ▼
[ Microservice Router ] ──► Injects X-Tenant-ID Header ──► [ Backend Services ]
                                                                 │
                                         ┌───────────────────────┴───────────────────────┐
                                         ▼                                               ▼
                         [ PostgreSQL Multi-Tenant DB ]                    [ Kafka Event Stream ]
                         (Row-Level Security Active)                       (Topic: tenant-usage-events)
```

#### 3. Deep Dive Design Decisions

##### Tenant Isolation (PostgreSQL)
Implement a **Shared Database with Row-Level Security (RLS)**. It provides a balance between cost efficiency and security for mid-market clients, with the option to deploy dedicated DB instances for enterprise-tier accounts.
```sql
-- Enable Row-Level Security on critical tables
ALTER TABLE billing_transactions ENABLE ROW LEVEL SECURITY;

-- Create Tenant Access Policy
CREATE POLICY tenant_isolation_policy ON billing_transactions
    USING (tenant_id = current_setting('app.current_tenant_id', true));
```
Before executing queries, the application database connection pool runner executes:
`SET LOCAL app.current_tenant_id = 'tenant_123';`
This query ensures that even if a developer forgets a `WHERE` clause, the database itself blocks access to any other tenant's data.

##### Billing & Metering Pipeline
* The API Gateway processes transactions and publishes an async log message to a partitioned **Kafka topic** (`tenant-usage-events`). Partition key is set to `tenant_id` to guarantee ordering and load distribution.
* A consumer group written in Node.js/Go reads events in batches. It updates a **Redis Sorted Set** for real-time dashboard visualization (`zincrby tenant:usage:2026-07 tenant_id 1`).
* Every 5 minutes, events are persisted to a cold analytical store (e.g. TimescaleDB or Postgres usage table partitioned by month) for invoice generation.

##### Trade-offs & Failure Modes
* **Redis Cache Outage:** If the Redis L2 rate-limit cluster crashes, the gateway falls back to a fail-open local rate-limiter, allowing requests through to protect uptime while firing alerts to pagerduty.

---

### Scenario 2: Offline-First Field Operations Sync Engine

#### 1. Requirements
* **Functional:**
  * Local offline data capture for field sales agents (Unilever distribution scenario).
  * Data queues up locally when there is no internet connection.
  * Delta sync transmits changes when connection is re-established.
  * Automated server-side conflict resolution.
* **Non-Functional:**
  * 99%+ reduction in data loss (consistent with Field Info GmbH project).
  * Low memory usage on mobile devices.
  * Minimal network payload during sync (delta-only).

#### 2. Sync Protocol & High-Level Architecture (ASCII)

```
[ Mobile App (Flutter/Android) ]               [ Web Application / Server API ]
  - SQLite Database                              - Node.js Sync Controller
  - Local Event Log Table                        - Transaction Coordinator
        │                                              │
        ├─────────── Sync Payload (Delta JSON) ─────────┤
        ▼                                              ▼
[ Connection Monitor ]                         [ PostgreSQL Database ]
  - Monitor internet status                      - Master Mutation Log
  - Queue management                             - Conflict resolution worker
```

#### 3. Database Schema Design (SQLite Client & Server)

##### SQLite Client Event Log Table
```sql
CREATE TABLE local_mutation_log (
    id TEXT PRIMARY KEY,
    entity_name TEXT NOT NULL,       -- e.g. 'inventory_orders'
    action_type TEXT NOT NULL,       -- 'CREATE', 'UPDATE', 'DELETE'
    record_id TEXT NOT NULL,
    payload TEXT NOT NULL,           -- JSON representation of updated columns
    client_timestamp INTEGER NOT NULL,
    sync_status TEXT DEFAULT 'PENDING' -- 'PENDING', 'SYNCED', 'FAILED'
);
```

##### Sync Protocol Execution Steps
1. **Offline Write:** The app writes to the local database and inserts a log row in `local_mutation_log` within a single local SQLite transaction.
2. **Sync Trigger:** When connectivity is detected, the app queries all `PENDING` mutations and sends a JSON payload:
   ```json
   {
     "client_id": "agent_987",
     "last_synced_at": 1782349000,
     "mutations": [
       { "id": "uuid-1", "entity_name": "orders", "action_type": "CREATE", "record_id": "ord-55", "payload": "...", "client_timestamp": 1782349120 }
     ]
   }
   ```
3. **Server Processing:** The server processes mutations sequentially in a PostgreSQL transaction.
   * It checks the record's `updated_at` timestamp in the database.
   * If `server_last_updated_at > last_synced_at`, a conflict is detected.
   * **Conflict Resolution Rule:** The server applies a deterministic rule (e.g. *Stock counts use additions; status updates prefer Server-state*), writes the final state, and logs the conflict resolution.
4. **Acknowledgment:** The server returns a status code and the server-side timestamp. The client marks marked mutations as `SYNCED` and updates its local state with the server's response.

---

## SECTION 5: BEHAVIORAL INTERVIEW PREPARATION (STAR stories)

### Question 1 (Ownership & Initiative): "Tell me about a time you identified a critical flaw in your system's architecture and fixed it without being asked."
* **STAR Response:**
  * **Situation:** At TechnoNext, our microservices platforms were routing requests through a third-party paid API Gateway. I noticed that as our transaction volume scaled toward 10M+ daily requests, the cloud costs were increasing rapidly, and the gateway was adding 80ms of latency to our requests.
  * **Task:** I set a goal to replace this infrastructure with a custom open-source alternative to reduce costs and improve performance, without disrupting live user traffic.
  * **Action:** I researched alternative architectures and selected Apache APISIX as the routing core due to its low-latency Lua engine. I integrated Coraza WAF to handle OWASP-compliant security checks. I designed a two-tiered caching system using local LRU cache in gateway pods for L1 and Redis for L2, preventing unnecessary database hits. I containerized the gateway on Kubernetes, configured auto-scaling rules based on active connections, and worked with the team to migrate platforms over using a gradual DNS rollover.
  * **Result:** The custom gateway reduced annual infrastructure and licensing costs by 70–80%, cut average response latencies by 35–50%, and maintained 99.9% uptime.

### Question 2 (Mentorship & Culture): "How do you mentor mid-level engineers to raise the technical bar of the team?"
* **STAR Response:**
  * **Situation:** At BJIT, I was leading a team of 8 developers on the Pocketalk Ventena 2 IoT system. We had several mid-level engineers who were writing functional code, but it lacked unit test coverage and struggled to meet performance standards under load.
  * **Task:** I wanted to establish clear standards for coding practices, automated testing, and optimization without hurting development velocity.
  * **Action:** I initiated three practices:
    1. **Code Review Checklist:** Created a shared checklist focused on database queries (identifying N+1 issues), error boundaries, and logging standards.
    2. **TDD Pair Programming:** Spent 2 hours weekly pair-programming on complex modules, demonstrating how to write unit tests first using Mockito.
    3. **Architecture Show-and-Tell:** Hosted bi-weekly lunch-and-learn sessions where we broke down system designs, such as distributed locks and transaction isolation levels.
  * **Result:** Within four months, the team's unit test coverage increased from 40% to 75%, production incident rates dropped by 30%, and two mid-level engineers were promoted to senior roles.

### Question 3 (Founder Collaboration & Trade-offs): "Tell me about a time you collaborated with non-technical leadership to make a hard technical trade-off."
* **STAR Response:**
  * **Situation:** During the Map Phantom project at TechnoNext, the founder wanted to launch a subscription API key model immediately to capitalize on client interest. However, our billing consumption pipeline and usage dashboard were still in development.
  * **Task:** I needed to align the business team's desire for speed with the technical team's need for stability and accurate usage tracking.
  * **Action:** I proposed a temporary phased approach to the founder:
    1. **Phase 1 (The MVP):** Instead of waiting to complete the automated billing system, we manually provisioned tenant keys with fixed rate-limits stored in Redis.
    2. **Phase 2 (The Async Pipeline):** We built the core usage tracking pipeline using Kafka to collect usage events in the background, but deferred the complex invoice generation service.
    3. **Phase 3 (Full Automation):** Rolled out billing dashboards and automated invoicing two months later.
  * **Result:** This approach allowed us to sign our first three enterprise customers six weeks ahead of the original schedule, validating product market fit while maintaining database integrity and security.

---

## SECTION 6: RESUME BULLET DEEP DIVE (Technical Defense)

### Bullet: "Migrated Rakuten Beauty monolith to Java microservices with 60% improved performance."
* **How did you measure the 60% performance improvement?**
  > "We established baseline metrics on the legacy monolith under typical load: average response time was 850ms, database CPU utilization was at 75%, and maximum throughput was 400 requests/sec. 
  > After migrating the critical Booking and Search modules to separate microservices, we ran load tests using JMeter. Under the same simulated load, the microservice average response time dropped to 340ms (a ~60% reduction in latency) and throughput capacity increased to 950 requests/sec."
* **What was the migration strategy? How did you handle data splitting?**
  > "We used the Strangler Fig Pattern to migrate the modules incrementally. For database splitting, we separated the Booking and Salon catalog tables from the core monolith schema. 
  > Because the monolith database was highly coupled, we initially kept a shared database with separate schemas, using application-level service calls rather than database joins. Once the logic was stable, we migrated to separate database instances, coordinating distributed reads using REST/gRPC and handling cross-domain mutations asynchronously using RabbitMQ."

### Bullet: "Designed serverless Pocketalk Test Automation using AWS Lambda..."
* **Why did you choose AWS Lambda over EC2 or ECS containers for test automation?**
  > "Our test suites ran on demand—triggered by CI/CD commits. If we used EC2 or ECS, we would either pay for idle servers or wait minutes for container provisioning during peak times.
  > AWS Lambda provided an on-demand serverless environment. We packaged our automated test runners inside a custom Docker image deployed on Lambda. When a developer pushed code, Jenkins triggered up to 50 concurrent Lambda execution tasks to run tests in parallel. This reduced test execution time from 45 minutes to 3 minutes, while cutting infrastructure costs by only charging for the exact milliseconds the tests ran."

---

## SECTION 7: MOCK INTERVIEW SIMULATION

### Round 1: Senior Technical Interview (Scale & Concurrency)

#### Interviewer: "How do you handle transactional safety when a single user request mutates data across three separate microservices? For instance, creating an order, updating inventory, and deducting credit."
* **Your Response:**
  > "In microservices, we avoid distributed transactions (like 2-Phase Commit) because they introduce locking and latency bottlenecks. Instead, we use the **Saga Pattern** with **Eventual Consistency**.
  > I would implement an Orchestrator-based Saga:
  > 1. The **Order Service** receives the request and creates a pending order. It publishes an `OrderCreated` event to Kafka.
  > 2. The **Inventory Service** consumes this event, reserves stock, and publishes `InventoryReserved`. (If stock is unavailable, it publishes `InventoryReservationFailed`).
  > 3. The **Billing Service** consumes `InventoryReserved`, attempts to deduct client balance, and publishes `PaymentCompleted`.
  > 4. The **Order Service** updates the status to `COMPLETED`.
  > **Handling Failures:** If the payment fails in step 3, the Billing Service publishes a `PaymentFailed` compensation event. The Inventory Service consumes this and releases the reserved stock, and the Order Service cancels the order. This ensures the system eventually reaches a consistent state."

---

### Round 2: System Design Interview (Enterprise API Integration)

#### Interviewer: "Design an integration gateway that consumes raw webhooks from Unilever distributor systems and processes them into bKash payment notifications. Expected load is 5,000 requests/sec, and the distributor APIs are prone to temporary network timeouts."
* **Your Response:**
  > "To build a resilient integration gateway under this load, we must decouple the ingestion layer from the processing layer.
  > 1. **Ingestion Layer:** Build a lightweight Node.js API endpoint. When a webhook arrives, it performs basic token authorization, validates the payload structure, and writes it directly to **Apache Kafka** (partition key = `distributor_id`). The gateway immediately returns an HTTP `202 Accepted` response. This takes under 5ms, protecting the upstream client from timeouts.
  > 2. **Processing Layer:** A consumer worker pool reads events from Kafka. It formats the data into bKash-compliant notifications and sends them to the bKash API.
  > 3. **Handling Downstream Timeouts:** If the bKash API returns a timeout or 5xx error, the worker retries the request using **exponential backoff with jitter** (e.g. retry after 1s, 2s, 4s, plus a random offset).
  > 4. **Dead Letter Queue (DLQ):** If a message fails after 5 retries, we write it to a Kafka DLQ topic (`failed-payment-notifications`) and fire a Prometheus alert. A support team can review the DLQ payload and trigger a replay once the downstream system recovers."

---

## SECTION 8: FINAL LAST-MINUTE CHEAT SHEET

### Top 10 High-Frequency Architecture Patterns
1. **Row-Level Security (RLS):** For multi-tenant database isolation.
2. **Strangler Fig Pattern:** For migrating legacy monoliths to microservices.
3. **Transactional Outbox Pattern:** For safe database-and-message publishing.
4. **Saga Pattern:** For eventual consistency across distributed services.
5. **Token Bucket / Leaky Bucket:** For rate limiting.
6. **2-Tier Caching:** Local LRU cache (L1) + Redis (L2) to prevent DB overload.
7. **Mutex Locking (Redis `SETNX`):** To mitigate Cache Stampede under peak load.
8. **CDC (Change Data Capture):** Reading database logs to stream updates.
9. **Optimistic Concurrency Control (OCC):** Version-based database write validation.
10. **Kubernetes Rolling Updates:** For zero-downtime microservice deployments.

### Top 5 Technical Metrics from Mobarak's Career
* **70–80%** Annual cost reduction by replacing third-party API Gateways with APISIX.
* **10M+** Daily requests processed with 99.9% uptime under peak load.
* **35–50%** Latency reduction using optimized two-tier caching layers.
* **60%** Performance improvement achieved migrating Rakuten monolith to microservices.
* **99%** Data loss reduction in offline-first apps using transaction local logs.

### Questions to Ask the Founder / Hiring Manager
1. *"As Intelligent Machines transitions to proprietary B2B SaaS, what is the biggest technical challenge the team is facing with tenant provisioning and infrastructure costs?"*
2. *"Can you describe the engineering culture around technical debt? How does the team balance fast shipping with long-term codebase health?"*
3. *"How does IM leverage AI-assisted engineering tools inside your current development workflow, and how do you evaluate engineering velocity?"*

---
*Preparation companion compiled for Mobarak Hosen | Targeted for Intelligent Machines (IM)*
