# Apache Kafka Cheatsheet

> Quick-reference guide for Kafka topic configuration, consumer behaviors, partition balancing, and durability settings

---

## 1. Producer Performance Tuning

- **`acks` Settings**:
  - `acks=0`: High throughput. No server acknowledgment. Risk of high data loss.
  - `acks=1` (Default): Lead partition writes to disk and acknowledges. Message can be lost if leader crashes before replica syncs.
  - `acks=all` (or `-1`): Lead waits for min.insync.replicas acknowledgment. Maximum durability.
- **Batching & Compression**:
  - Batch size: `batch.size=65536` (64KB batch writes).
  - Wait window: `linger.ms=20` (buffer writes for 20ms before flushing).
  - Compression: `compression.type=zstd` (high compression ratio, low CPU overhead).

---

## 2. Consumer Offset Commits & Group Tuning

- **Offset Reset**:
  - `auto.offset.reset=latest`: Start reading from new events. Old events are ignored.
  - `auto.offset.reset=earliest`: Read topic history from offset zero.
- **Consumer Rebalance Trigger**:
  - Occurs when a consumer leaves or joins the group.
  - Caused by: Thread blockage or processing takes longer than `max.poll.interval.ms` (causes broker to assume consumer is dead, triggers group rebalance).

---

## 3. Durability & Topic Settings

| Setting | Recommendation | Rationale |
|---|---|---|
| `replication.factor` | `3` | Survives loss of 2 brokers without data loss. |
| `min.insync.replicas` | `2` | Guarantees that at least one replica has written the event when `acks=all` is set. |
| `retention.ms` | `604800000` (7 Days) | Balance disk utilization with event recovery requirements. |
| `unclean.leader.election.enable` | `false` | Prevents non-synchronized replicas from being elected leader, protecting consistency. |
