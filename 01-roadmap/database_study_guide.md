# Database Architecture, Schema Design, & SQL Optimization Guide

This guide covers relational database internals, comparison of PostgreSQL and MySQL, partitioning, sharding, multi-tenancy isolation models, query profiling, and Change Data Capture (CDC) pipelines.

---

## 1. ACID Properties & Isolation Levels

Relational databases guarantee consistency through ACID transactions:
* **Atomicity:** All operations in a transaction succeed, or all fail (rollback).
* **Consistency:** Prevents database corruption by enforcing constraints (foreign keys, unique indexes).
* **Isolation:** Controls how transaction changes are visible to other concurrent transactions.
* **Durability:** Committed data is written to non-volatile storage (via the Write-Ahead Log - WAL / Redo Log) before transaction success is acknowledged.

### Transaction Isolation Levels & Anomalies
Standard SQL isolation levels define which read anomalies are permitted:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
| :--- | :---: | :---: | :---: | :---: |
| **Read Uncommitted** | Allowed | Allowed | Allowed | Allowed |
| **Read Committed** (Default) | Prevented | Allowed | Allowed | Allowed |
| **Repeatable Read** | Prevented | Prevented | Allowed / Prevented* | Allowed |
| **Serializable** | Prevented | Prevented | Prevented | Prevented |

* *Read Anomaly Definitions:*
  * **Dirty Read:** Reading uncommitted writes from another transaction.
  * **Non-Repeatable Read:** Reading the same row twice inside a transaction yields different values because another transaction committed an update in between.
  * **Phantom Read:** Re-executing a query inside a transaction returns new matching rows (phantoms) inserted by another transaction in between.
  * *Note:* PostgreSQL's Repeatable Read isolation level prevents Phantom Reads using MVCC snapshots, which differs from standard SQL specifications.

---

## 2. PostgreSQL vs. MySQL Internals

### Architectural Comparison

```
+------------------------------------------+------------------------------------------+
| PostgreSQL                               | MySQL (InnoDB Engine)                    |
+------------------------------------------+------------------------------------------+
| - Process-per-connection model.          | - Thread-per-connection model.           |
| - MVCC: Appends new tuples (row versions)| - MVCC: Modifies row in-place; stores    |
|   directly to the table page (heap).     |   old versions in Undo Log segments.     |
| - Requires VACUUM to reclaim space.      | - Purges Undo Logs automatically.        |
| - Index points directly to Heap Tuple ID.| - Clustered Index (Primary Key holds     |
| - Rich JSONB support with GIN indexes.   |   row data; secondary indexes point to PK)|
+------------------------------------------+------------------------------------------+
```

### Key Differences
1. **MVCC Implementation:**
   * *PostgreSQL:* When a row is updated, a new version of the row is created and inserted into the main table page. The older row remains until it is cleaned up by the **autovacuum** process.
   * *MySQL:* Rows are updated in-place inside the page. The previous state is written to the **Undo Log** space. Readers traverse the undo logs to reconstruct historical snapshots.
2. **Indexing:**
   * *PostgreSQL:* All indexes contain direct pointers to physical row locations (Tuple IDs).
   * *MySQL (InnoDB):* Uses a **Clustered Index** structure where the primary key index leaf nodes contain the physical row data. Secondary indexes store the primary key value as a pointer, requiring a double lookup if a query cannot be satisfied by the secondary index alone.

---

## 3. Schema Design & Multi-Tenancy Models

When designing a B2B SaaS platform for clients like Unilever or bKash, selecting the right multi-tenancy model affects scalability and cost control:

```
┌────────────────────────────────────────────────────────────────────────┐
│ Multi-Tenancy Architecture Options                                     │
├───────────────────┬─────────────────────────────┬──────────────────────┤
│ 1. Database Pool  │ 2. Schema-per-Tenant        │ 3. Shared DB (RLS)   │
│ [DB A]   [DB B]   │ [DB 1]                      │ [DB 1]               │
│                   │ ├─ Schema Tenant A          │ └─ Single Schema     │
│                   │ └─ Schema Tenant B          │    (tenant_id col)   │
└───────────────────┴─────────────────────────────┴──────────────────────┘
```

1. **Database-per-Tenant (Isolated):** Highest security, separate backup strategies. Connection pooling and database resource overhead scale linearly with the number of tenants.
2. **Schema-per-Tenant (Semi-Shared):** A single database instance containing separate logical schemas. Good database boundary isolation. However, executing schema migrations across 1,000 schemas requires complex orchestration.
3. **Shared Database with Row-Level Security (RLS):** All tenants share the same tables. Tenant data is isolated by a `tenant_id` column.
   * *Implementation:* Use PostgreSQL RLS policies to prevent cross-tenant data leaks at the database level:
     ```sql
     ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;
     
     CREATE POLICY tenant_isolation_policy ON transactions
         USING (tenant_id = current_setting('app.current_tenant_id'));
     ```

---

## 4. Triggers, Partitioning, & Query Analysis

### Partitioning
Partitioning divides a large table into smaller physical tables (partitions) while presenting a single logical interface.
* **Range Partitioning:** Splitting by date ranges (e.g. `orders_2026_01`, `orders_2026_02`). Highly effective for historical timeseries data.
* **List Partitioning:** Splitting by explicit values (e.g. partition by region: `north`, `south`).
* **Hash Partitioning:** Splitting by a hash value of a key (e.g. hash partitioning on `tenant_id` to distribute load evenly across 10 partitions).

### Query Execution Analysis (`EXPLAIN ANALYZE`)
To optimize queries, you must read query plans:
* **Seq Scan:** Sequential table scan. Check if an index is missing, or if the index cardinality is too low.
* **Index Scan:** Traversing an index tree, then fetching the corresponding row from the heap.
* **Index Only Scan:** Fetching columns directly from the index page without accessing the table heap (requires index to include all columns in the `SELECT` clause).
* **Buffers output:** `EXPLAIN (ANALYZE, BUFFERS)` reveals memory hits. Look for `shared read` (disk reads) vs `shared hit` (cache memory hits). Minimize disk reads.

---

### Questions & Answers: Optimization & Partitioning

#### Q1: Explain when you would write a Database Trigger vs. performing the action in Application Services.
**Answer:**
> "I prefer executing business logic inside Application Services to maintain testability, clear observability, and easy deployment. However, Database Triggers are appropriate for:
> 1. **Data Auditing & Compliance:** Writing changed records to an audit table to guarantee the change is captured, regardless of whether the update came from the application or a manual DBA command.
> 2. **Enforcing Strict Invariants:** Auto-calculating aggregate totals or setting default states when absolute data consistency is required at the storage layer.
> *Trade-offs:* Triggers hide execution flow, are difficult to unit test, and add CPU load to database instances that cannot easily scale horizontally."

#### Q2: Write a DDL script setting up PostgreSQL Table Partitioning by Range on an `orders` table based on a `created_at` timestamp.
```sql
-- Create parent table
CREATE TABLE orders (
    id UUID NOT NULL,
    tenant_id VARCHAR(50) NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id, created_at) -- Partition key must be part of primary key
) PARTITION BY RANGE (created_at);

-- Create active monthly partitions
CREATE TABLE orders_2026_06 PARTITION OF orders
    FOR VALUES FROM ('2026-06-01 00:00:00') TO ('2026-07-01 00:00:00');

CREATE TABLE orders_2026_07 PARTITION OF orders
    FOR VALUES FROM ('2026-07-01 00:00:00') TO ('2026-08-01 00:00:00');
```

---

## 5. Change Data Capture (CDC) with Debezium & Kafka

CDC streams real-time database modifications (inserts, updates, deletes) to downstream systems without affecting application performance.

### Ingestion Pipeline Architecture

```
[ PostgreSQL / WAL ] ──► [ Debezium Connector ] ──► [ Kafka Connect ] ──► [ Kafka Topic ]
```

1. **Write-Ahead Log (WAL) Ingestion:** PostgreSQL writes transaction updates to the WAL. Debezium uses logical replication slots to read the WAL stream directly.
2. **Kafka Connect:** Runs the Debezium engine, transforming raw WAL binary data into structured JSON/Avro events.
3. **Kafka Topic Delivery:** Events are published to topics matching the database tables (e.g. `db.public.orders`), enabling downstream services (billing engines, elasticsearch) to consume updates asynchronously.

---

### Questions & Answers: CDC & Kafka Connect

#### Q1: How do you handle schema changes (DDL migrations) when using Debezium CDC in production?
**Answer:**
> "Schema changes can break downstream consumers if they receive payloads with unexpected fields. I manage this by:
> 1. **Schema Registry:** Enforcing schema serialization using Apache Avro with Confluent Schema Registry. This validates backward/forward compatibility.
> 2. **Coordinated Migration Steps:**
>    - Step A: Execute DDL changes to add columns as nullable in the database.
>    - Step B: Debezium captures the schema change, registers the new schema version in the registry, and continues streaming.
>    - Step C: Deploy updated microservice consumers that can parse the new fields.
>    - Step D: Update the application producers to write data into the new columns."

---

## 6. Database Sharding

Sharding is horizontal partitioning across separate physical database servers.

### Sharding Strategies
* **Range-Based Sharding:** Route data to shards based on ranges of a key (e.g. IDs 1–1M go to Shard 1, 1M–2M go to Shard 2). Focuses hotspots on the newest shard.
* **Directory-Based Sharding:** A lookup service maps keys to shard IDs. Introduces a single point of failure and lookup latency.
* **Hash-Based Sharding:** Route using a hash algorithm:
  $$\text{Shard ID} = \text{hash}(\text{sharding\_key}) \pmod{\text{Number of Shards}}$$
  This distributes data evenly, but resizing the cluster requires rehashing and moving a significant volume of data.

---

### Questions & Answers: Sharding

#### Q1: What are the main engineering challenges introduced when you shard a database? How do you mitigate them?
**Answer:**
> "Sharding introduces substantial system complexity:
> 1. **Cross-Shard Joins:** SQL joins cannot cross physical network boundaries.
>    * *Mitigation:* Denormalize data, copy lookup tables (like configurations) to every shard, or perform joins inside the application service layer.
> 2. **Distributed Transactions:** Ensuring atomic updates across two shards requires distributed transaction coordinators (2-Phase Commit), which add high latency and lock overhead.
>    * *Mitigation:* Design the system so all related entities (e.g. a tenant's orders, items, and billing) share the same sharding key (`tenant_id`), ensuring they reside on the same physical shard.
> 3. **Re-sharding under load:** Adding shards requires moving millions of rows.
>    * *Mitigation:* Implement **Consistent Hashing** algorithms to minimize the volume of data that needs to migrate when shards are added."

---
Database & SQL Optimization Study Guide
