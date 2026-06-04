# System Design Reference Cheatsheet

> Quick-reference checklist for system design calculations, components selection, and scale metrics

---

## 1. Scale Conversion Guide

- **Requests per Second (QPS)**:
  - 100K requests per day $\approx$ 1.15 req/sec
  - 1M requests per day $\approx$ 12 req/sec
  - 10M requests per day $\approx$ 116 req/sec
  - 100M requests per day $\approx$ 1,160 req/sec (1.2K QPS)
  - 1B requests per day $\approx$ 11,600 req/sec (12K QPS)
- **Time Duration**:
  - 1 Day = 86,400 seconds
  - 1 Month = 2,592,000 seconds (2.6M)

---

## 2. Dynamic Performance Estimations

### Network Round-Trip Time (Latency)
- L1 cache reference: 0.5 ns
- L2 cache reference: 7 ns
- Main memory reference: 100 ns
- Read 1MB sequentially from SSD: 1,000,000 ns (1 ms)
- Round trip within same datacenter: 500,000 ns (0.5 ms)
- Ping across ocean (NY to London): 150 ms

### Storage Math
- 1 write/sec with 1KB record size $\approx$ 86.4MB per day.
- 100 writes/sec with 1KB record size $\approx$ 8.6GB per day.
- 10,000 writes/sec with 1KB record size $\approx$ 864GB per day ($\approx$ 315TB per year).

---

## 3. Tech Stack Selectors

| Problem | Best Solution | Alternative |
|---|---|---|
| **Real-Time Timelines** | Redis ZSET | Cassandra |
| **Highly-Reliable Ledger** | CockroachDB / SQL Serializable | LedgerDB |
| **Full-text search** | Elasticsearch | Postgres pg_trgm |
| **Dynamic Surge Pricing** | Flink Stream Analytics | Spark Streaming |
| **Raw Media Files** | AWS S3 object store | MinIO |
| **Configuration KV** | Etcd / Consul | ZooKeeper |
