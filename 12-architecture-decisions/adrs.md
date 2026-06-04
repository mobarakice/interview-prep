# Architectural Decision Records (ADRs)

> Production-ready reference containing key architectural decision records (ADRs) demonstrating structural choices, trade-offs, and design rationales
> **Key Decisions**: Database Migration (RDBMS to NoSQL), Queue Selection (Kafka vs RabbitMQ), Language Selection (Java Spring Boot vs Go)

---

## ADR 01: Migrating User Timeline Feed from PostgreSQL to Cassandra

### Status
Accepted

### Context
Our social media product user base has scaled to 10M DAU. The SQL database (`user_activities` table) is processing 50K reads/sec and 5K writes/sec. PostgreSQL read replicas are experiencing replication lag spikes of over 45 seconds, causing timeline feeds to render out-of-order or stale activities for users. The write load is hitting disk I/O limits on the primary PostgreSQL node.

### Decision
Migrate user activity timelines and feed structures from PostgreSQL to Apache Cassandra.

### Rationale
- **Write Performance**: Cassandra uses an LSM (Log-Structured Merge) tree write path, allowing append-only disk writes. This resolves our disk I/O bottleneck on ingestion.
- **Horizontal Scale**: Cassandra's ring topology allows seamless horizontal scaling by adding nodes without service disruption or primary-replica write limits.
- **Schema Mapping**: Timeline data is naturally queryable by `user_id` and sorted by time, which maps perfectly to Cassandra clustering keys.

### Consequences
- **Eventual Consistency**: Replicas synchronize asynchronously. Feed updates might take up to 2 seconds to reflect globally.
- **Query Limits**: Relational operations (like joins, grouping) are no longer supported. The application layer must pre-compute and denormalize data before writing.
- **Operations Cost**: Running a Cassandra cluster requires specialized monitoring (JVM GC tuning, SSTable compaction checks).

---

## ADR 02: Selecting Apache Kafka as Core Event Broker (Replacing RabbitMQ)

### Status
Accepted

### Context
Our financial platform is moving to an event-driven architecture. We require a messaging platform to transport transactions, auditing records, and user updates. We need to support event replayability for auditing and the ability to process events out-of-order without blocking.

### Decision
Deploy Apache Kafka as the main event bus, replacing the existing RabbitMQ clusters.

### Rationale
- **Log Persistence**: Kafka retains messages on disk according to configuration (e.g. 14 days), allowing audit engines to replay events for reconciliation. RabbitMQ deletes messages upon consumer acknowledgment.
- **Consumer Independence**: Multiple services (ledger, fraud detection, analytics) can subscribe to the same event stream independently without creating separate message queues.
- **High Throughput**: Kafka batching mechanisms allow 10x higher ingestion rates compared to RabbitMQ under equal hardware profiles.

### Consequences
- **No Wildcard Routing**: RabbitMQ's complex routing exchanges are lost. Routing must occur using partition keys or application-level filtering.
- **Consumer Management**: Consumers must track their own offsets (offsets are stored in Kafka commit logs).

---

## ADR 03: Selecting Java with Spring Boot for Payment Core Service (vs. Go)

### Status
Accepted

### Context
We are spinning off our payments engine into a standalone microservice. The development team is split between choosing Go (for its fast boot times and low memory profile) and Java/Spring Boot (for its mature ecosystem).

### Decision
Develop the Payment Core Service in Java 21 using Spring Boot 3.x.

### Rationale
- **Ecosystem Maturity**: Payments require high security and transaction isolation. The Java ecosystem offers mature libraries for encryption (JCE), database transaction management (`@Transactional`), and robust testing utilities (JUnit, Mockito).
- **PCI-DSS Compliance**: Most major payment gateway SDKs (Stripe, Adyen) maintain official, battle-tested Java SDKs. Writing custom wrappers in Go adds development risk.
- **Virtual Threads**: Java 21 introduces virtual threads (Project Loom), enabling high-concurrency request processing similar to Go routines without the complexity of async programming.

### Consequences
- **Memory Consumption**: Higher JVM memory footprint compared to native compiled Go binaries. EKS node capacities must be provisioned accordingly.
- **Start-up Latency**: Longer container startup times (3-5 seconds) compared to Go (< 100ms). We mitigate this during autoscale triggers by maintaining minimum replica thresholds.
