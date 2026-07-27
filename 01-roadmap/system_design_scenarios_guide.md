# System Design Scenarios Study Guide

This guide details five high-probability system design scenarios tailored for Intelligent Machines (IM), including functional and non-functional requirements, detailed architectures, database choices, caching, scaling trade-offs, and failure mitigation strategies.

---

## Scenario 1: Multi-Tenant Enterprise B2B SaaS Platform (IM SaaS Core)

### 1. Requirements Gathering
* **Functional:**
  * Self-service tenant onboarding with custom subdomains (e.g. `tenant1.im.com`).
  * API Key generation, rotation, and usage quota enforcement.
  * Multi-tier subscription access (Bronze, Silver, Gold).
  * Consolidated billing dashboards displaying real-time API usage.
* **Non-Functional:**
  * Strict tenant isolation (prevent cross-tenant data leaks).
  * API authorization check overhead under 5ms.
  * Seamless horizonal scaling to support 100M daily transactions.
  * Operational cost efficiency.

### 2. High-Level Architecture (ASCII)

```
[ Client Application ] ──► [ AWS Route 53 (Dynamic DNS) ]
                                   │
                                   ▼
                       [ Application Load Balancer ]
                                   │
                                   ▼
                    [ API Gateway (Apache APISIX) ]
                                   │
            ┌──────────────────────┴──────────────────────┐
            ▼                                             ▼
 [ In-Memory L1 Cache ]                           [ Auth Service ]
 (Dynamic tenant token metadata)                  (Validates JWT/HMAC)
            │                                             │
            ▼                                             ▼
 [ Redis Cluster L2 Cache ] ◄─────────────────────────────┘
 (Rate limits & quota bucket)
            │
            ▼
 [ Core App Microservices ] ──► (X-Tenant-ID Context)
            │
    ┌───────┴───────┐
    ▼               ▼
[Postgres DB]   [Kafka Event Hub] ──► [Billing Consumer] ──► [TimescaleDB]
(RLS Policies)  (Usage streams)
```

### 3. Deep Dive System Decisions
* **Database Isolation Strategy:** Use a **Shared Database with Row-Level Security (RLS)** in PostgreSQL. This keeps database infrastructure costs minimal while providing database-enforced security boundaries.
  ```sql
  ALTER TABLE customer_invoices ENABLE ROW LEVEL SECURITY;
  
  CREATE POLICY tenant_isolation_policy ON customer_invoices
      USING (tenant_id = current_setting('app.current_tenant_id'));
  ```
  For high-paying enterprise tenants (e.g. bKash), route connection requests to dedicated PostgreSQL instances using a dynamic lookup service.
* **Usage Quota Metering:** The API Gateway validates requests against Redis cache. When validated, the gateway asynchronously writes an event to a partitioned Kafka topic. The Billing Consumer reads events in batches to compile usage history.
* **Interviewer Follow-up Question:** *How do you run database schema migrations across different isolation tiers without causing downtime?*
  * **Answer:** Use tools like Flyway or Liquibase. For shared databases, apply standard migrations. For dedicated databases, run migrations concurrently across instances using a CI/CD runner script, ensuring all changes are backward-compatible.

---

## Scenario 2: Real-Time High-Throughput Ingestion & Analytics Pipeline

### 1. Requirements Gathering
* **Functional:**
  * Ingest 20,000 events/second (analytics, clicks, transaction events) from mobile apps and web platforms.
  * Provide real-time dashboards showing active events.
  * Retain raw data for historical analysis.
* **Non-Functional:**
  * Latency from ingestion to dashboard under 500ms.
  * Durability: Zero event loss.
  * Scalability: Support spikes of up to 50,000 events/second.

### 2. Detailed Architecture (ASCII)

```
[ Clients ] ──► [ Load Balancer ] ──► [ Ingestion Service Pods ]
                                             │
                                             ▼
                              [ Apache Kafka Ingestion Hub ]
                              (Partitioned by tenant_id/user_id)
                                             │
                 ┌───────────────────────────┴───────────────────────────┐
                 ▼                                                       ▼
      [ Flink Stream Processor ]                               [ Vector / FluentBit ]
      (Sliding window aggregates)                                        │
                 │                                                       ▼
                 ▼                                              [ Object Storage ]
       [ Redis Aggregations ]                                    (Parquet files)
                 │
                 ▼
       [ Real-time Dashboard ]
```

### 3. Deep Dive System Decisions
* **Storage Layer Separation:**
  * **Hot Tier:** Store real-time metrics (like active clicks per minute) in **Redis Sorted Sets**, enabling fast dashboard queries.
  * **Warm Tier:** Store structured timeseries data in **TimescaleDB** (PostgreSQL extension) for analytical queries spanning hours or days.
  * **Cold Tier:** Export raw JSON data to **GCP Cloud Storage / AWS S3** in compressed Apache Parquet format for long-term storage and query tools like Athena or BigQuery.
* **Managing Ingestion Spikes:** Kafka partitions serve as buffers. If stream processors lag during traffic spikes, events accumulate in Kafka queues without impacting ingestion availability.
* **Interviewer Follow-up Question:** *How do you prevent duplicate events if mobile clients retry requests due to network issues?*
  * **Answer:** Enforce client-side UUID generation for every event. The stream processor checks incoming event UUIDs against a sliding-window deduplication cache in Redis.

---

## Scenario 3: High-Availability API Gateway & Cost-Control Router

### 1. Requirements Gathering
* **Functional:**
  * Intercept and route all client requests to backend services.
  * Handle authentication (JWT / HMAC), rate limiting, and request routing.
  * Web Application Firewall (WAF) integration.
* **Non-Functional:**
  * Low overhead (adds less than 3ms to request latency).
  * High availability (99.99% uptime).
  * Reduce cloud infrastructure licensing and resource costs (consistent with Mobarak's 70-80% cost reduction).

### 2. Detailed Architecture (ASCII)

```
                  [ Inbound Internet Traffic ]
                                │
                                ▼
                   [ APISIX Gateway Cluster ]
                   (Containerized on Kubernetes)
                                │
       ┌────────────────────────┼────────────────────────┐
       ▼                        ▼                        ▼
 [ Lua WAF Plugin ]       [ L1 Cache (Local) ]     [ L2 Cache (Redis) ]
 (Coraza / OWASP rules)   (In-memory token metadata)(Distributed lock & limits)
       │                        │                        │
       └────────────────────────┬────────────────────────┘
                                ▼
                   [ Internal Pod Network ] ──► [ Microservices ]
```

### 3. Deep Dive System Decisions
* **Technology Choice:** Use **Apache APISIX** configured with Lua plugins. APISIX utilizes Nginx event-driven architecture, which offers lower latency and resource usage compared to JVM-based or paid gateways.
* **Resource Optimization:** Configure Kubernetes Horizontal Pod Autoscaler (HPA) to scale pods based on active connection metrics rather than CPU utilization. During low-traffic periods, the gateway cluster scales down to minimal resource allocations.
* **Interviewer Follow-up Question:** *If the WAF plugin causes latency spikes, how do you optimize it?*
  * **Answer:** Optimize rules to bypass regex evaluations for static content or trusted IP ranges. Run WAF checks in parallel or compile critical rules using WebAssembly (Wasm) inside the gateway worker processes.

---

## Scenario 4: Offline-First Field Operations Sync Engine

### 1. Requirements Gathering
* **Functional:**
  * Offline data capture for field sales agents (e.g. Unilever distributor app).
  * Queue local mutations while offline.
  * Synchronize changes incrementally when connection is restored.
  * Server-side conflict resolution.
* **Non-Functional:**
  * Zero data loss (99% data loss reduction).
  * Minimal mobile battery and network usage.
  * Strict write ordering (e.g., customer creation must sync before order placement).

### 2. Detailed Architecture (ASCII)

```
[ SQLite Client DB ] ──► [ Local Mutation Queue Table ]
                                │
          ┌─────────────────────┴─────────────────────┐
          ▼ (No Internet Connection)                  ▼ (Internet Restored)
[ Queue Locally ]                             [ Push Sync Request (Delta JSON) ]
                                                      │
                                                      ▼
                                            [ Sync Controller ]
                                                      │
                                                      ▼
                                            [ Conflict Resolver ]
                                            (Checks record version)
                                                      │
                                                      ▼
                                            [ PostgreSQL Master ]
```

### 3. Deep Dive System Decisions
* **Sync Protocol Design:** The client uses an incremental sequence number (client version) and transaction ID for all local changes. The sync payload contains:
  ```json
  {
    "client_id": "field_agent_44",
    "last_sync_timestamp": 1782349000,
    "mutations": [
      { "id": "tx_1", "action": "CREATE", "table": "customers", "payload": "{...}" },
      { "id": "tx_2", "action": "CREATE", "table": "orders", "payload": "{...}" }
    ]
  }
  ```
  The server processes the mutations inside a transaction. If a conflict occurs, it applies a rule (e.g., *Server Wins*) and returns the conflict status along with updated database values.
* **Interviewer Follow-up Question:** *How do you resolve conflicts if two agents update the same product inventory count offline?*
  * **Answer:** Avoid overwriting field values. Instead of syncing the absolute product count, sync the delta value (e.g. `decrement inventory by 5`). The server applies the subtraction to the database using an atomic SQL query:
    `UPDATE products SET stock = stock - 5 WHERE id = 123;`

---

## Scenario 5: Distributed Task Scheduler & Notification Engine

### 1. Requirements Gathering
* **Functional:**
  * Schedule tasks to run at specific times (one-off or recurring crons).
  * Support millions of scheduled tasks (e.g., payment reminders for bKash).
  * Deliver multi-channel notifications (Email, SMS, Webhook).
* **Non-Functional:**
  * High availability: Tasks must execute even if nodes fail.
  * Near-exact execution timing (accuracy within 5 seconds).
  * Prevent duplicate notifications (at-most-once execution of the notify action).

### 2. Detailed Architecture (ASCII)

```
[ User App / API ] ──► [ Task Schedule Service ] ──► [ Task Store (Postgres) ]
                                                           │
                                                           ▼
                                               [ Distributed Coordinator ]
                                               (Redis locks / Zookeeper)
                                                           │
                                                           ▼
                                               [ Scheduler Workers ]
                                               (Pulls ready tasks)
                                                           │
                                                           ▼
                                               [ RabbitMQ Task Queues ]
                                                           │
                                                           ▼
                                               [ Notification Runners ]
                                               (Executes email/SMS/webhooks)
```

### 3. Deep Dive System Decisions
* **Data Storage:** Store task definitions in a relational database (PostgreSQL) partitioned by date. Create a composite index on `(next_execution_time, status)`.
* **Task Allocation & Locking:** Multiple scheduler worker pods run in parallel. To prevent duplicate execution:
  * Workers use a Redis distributed lock (`redlock`) or run an atomic database update query:
    ```sql
    UPDATE scheduled_tasks 
    SET status = 'RUNNING', locked_by = 'worker_pod_3'
    WHERE id IN (
        SELECT id FROM scheduled_tasks 
        WHERE next_execution_time <= NOW() AND status = 'PENDING'
        ORDER BY next_execution_time ASC 
        LIMIT 100 FOR UPDATE SKIP LOCKED
    ) RETURNING *;
    ```
    The `SKIP LOCKED` clause prevents workers from blocking on locked rows, allowing them to pull ready tasks concurrently.
* **Interviewer Follow-up Question:** *What happens if a worker pod crashes mid-task execution?*
  * **Answer:** Tasks have a `heartbeat_timestamp` updated periodically by the worker. A supervisor process runs every minute to find tasks marked as `RUNNING` with missing heartbeats, resets their status back to `PENDING`, and releases the lock for retry.

---
System Design Scenarios Guide
