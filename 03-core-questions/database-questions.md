# Database Interview Questions

> 8 architect-level questions on PostgreSQL internals, MVCC, isolation, indexing, sharding, and query optimization.
> Cross-references: [Database Masterclass](../07-databases/database-masterclass.md) · [PostgreSQL Cheatsheet](../15-cheatsheets/postgresql-cheatsheet.md)

---

## Q1: Explain PostgreSQL MVCC. How does it handle concurrent reads and writes?

### Interviewer's Expectation
Understanding of tuple versioning, transaction visibility, xmin/xmax, and why VACUUM is necessary.

### Excellent Answer

MVCC (Multi-Version Concurrency Control) allows readers and writers to operate without blocking each other. Each row version (tuple) has hidden columns: `xmin` (creating transaction) and `xmax` (deleting/updating transaction).

**How it works**:
- **INSERT**: Creates a tuple with `xmin=current_txn`, `xmax=0`
- **UPDATE**: Marks old tuple with `xmax=current_txn`, creates new tuple with `xmin=current_txn`
- **DELETE**: Marks tuple with `xmax=current_txn`
- **Read**: Only sees tuples where `xmin` is committed AND (`xmax` is not set OR `xmax` is not yet committed)

```
Transaction 100: INSERT (id=1, name='Alice')  → xmin=100, xmax=0
Transaction 200: UPDATE SET name='Bob'         → Old: xmin=100, xmax=200
                                               → New: xmin=200, xmax=0
Transaction 150 (started before 200):          → Still sees name='Alice' (snapshot isolation)
```

**Why VACUUM is critical**: Dead tuples (old versions) accumulate. VACUUM reclaims this space. Without it: table bloat, index bloat, transaction ID wraparound risk (2^31 limit). Autovacuum should NEVER be disabled.

### Common Mistakes
- Disabling autovacuum for "performance" (causes catastrophic bloat and wraparound), not understanding that UPDATE = DELETE + INSERT (doubles storage), not monitoring dead tuple ratio, confusing VACUUM with VACUUM FULL (latter locks table)

### Follow-up Questions
- "What is transaction ID wraparound and how do you prevent it?"
- "How does HOT (Heap-Only Tuples) optimization work?"
- "Compare PostgreSQL MVCC with MySQL InnoDB MVCC."

---

## Q2: Compare PostgreSQL isolation levels. When would you use Serializable?

### Excellent Answer

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|-------------------|--------------|-------------|
| Read Uncommitted | ❌ (PG treats as Read Committed) | ✅ Possible | ✅ Possible | Fastest |
| **Read Committed** (default) | ❌ Prevented | ✅ Possible | ✅ Possible | Fast |
| **Repeatable Read** | ❌ Prevented | ❌ Prevented | ❌ Prevented* | Medium |
| **Serializable** | ❌ Prevented | ❌ Prevented | ❌ Prevented | Slowest |

*PostgreSQL's Repeatable Read actually prevents phantom reads (unlike SQL standard).

**When to use Serializable**: Financial transactions where any anomaly is unacceptable. Account transfers: `BEGIN SERIALIZABLE; SELECT balance; UPDATE balance; COMMIT;` guarantees no race condition even with concurrent transfers.

**Serializable Snapshot Isolation (SSI)**: PostgreSQL uses optimistic approach — runs transactions concurrently, detects conflicts at commit time. If conflict detected → one transaction aborts with serialization failure → application must retry.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
@Retryable(value = SerializationFailureException.class, maxAttempts = 3)
public void transferFunds(UUID from, UUID to, BigDecimal amount) {
    Account source = accountRepo.findByIdForUpdate(from);
    Account target = accountRepo.findByIdForUpdate(to);
    source.debit(amount);
    target.credit(amount);
}
```

---

## Q3: Explain PostgreSQL index types. When do you use each?

### Excellent Answer

| Index Type | Data Structure | Best For | Example |
|-----------|---------------|----------|---------|
| **B-tree** (default) | Balanced tree | Equality, range, sorting, `LIKE 'prefix%'` | `CREATE INDEX idx ON orders(created_at)` |
| **Hash** | Hash table | Equality only (=) | `CREATE INDEX idx ON users USING hash(email)` |
| **GIN** | Inverted index | Full-text search, arrays, JSONB | `CREATE INDEX idx ON docs USING gin(to_tsvector('english', body))` |
| **GiST** | Generalized search tree | Geometric, range types, full-text | `CREATE INDEX idx ON locations USING gist(coordinates)` |
| **BRIN** | Block range | Large tables with natural ordering | `CREATE INDEX idx ON events USING brin(created_at)` |
| **Partial** | Any (filtered) | Subset of rows | `CREATE INDEX idx ON orders(status) WHERE status = 'PENDING'` |
| **Covering** | B-tree + included cols | Index-only scans | `CREATE INDEX idx ON orders(customer_id) INCLUDE (total, status)` |

**Architect decision**: For a FinTech platform with millions of transactions, I'd use: B-tree on `(account_id, created_at)` for account history, BRIN on `created_at` for time-range queries (minimal storage), partial index on `status = 'PENDING'` (only 1% of rows), covering index for dashboard queries.

---

## Q4-Q8: Additional Database Questions (Condensed)

**Q4: How do you optimize a slow query using EXPLAIN ANALYZE?** Read the plan bottom-up. Look for: Sequential Scans on large tables (add index), Nested Loop with high row counts (wrong join), Sort with high cost (add index for ORDER BY), actual vs estimated rows mismatch (run ANALYZE). Key metrics: actual time, rows, loops, buffers hit vs read.

**Q5: Compare sharding strategies for a payment database.** Hash sharding (even distribution by payment_id), range sharding (by date — easy archival), geographic sharding (by region — data locality). Challenges: cross-shard queries, rebalancing, distributed joins. Tools: Citus for PostgreSQL, application-level with ShardingSphere.

**Q6: Design a connection pooling strategy for high-concurrency microservices.** HikariCP per service (min=5, max=20, connection-timeout=30s). PgBouncer in front of PostgreSQL for global pooling (transaction mode). Formula: `max_connections = num_services × pool_size`. Monitor: active connections, wait time, pool utilization.

**Q7: How do you implement database replication for read scaling?** Streaming replication (async for read replicas, sync for HA). Logical replication for selective table replication. Spring Boot: `@Transactional(readOnly=true)` routes to replica via `AbstractRoutingDataSource`. Monitor replication lag (< 1s for financial systems).

**Q8: Explain table partitioning strategies in PostgreSQL.** Range (by date — most common for time-series/transactions), List (by status, region), Hash (by ID for even distribution). Partition pruning speeds queries automatically. Example: `PARTITION BY RANGE (created_at)` with monthly partitions. Maintenance: drop old partitions instead of DELETE (instant vs hours).
