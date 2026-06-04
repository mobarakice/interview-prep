# Distributed Systems Cheatsheet

> Quick-reference guide for consistency models, distributed transaction patterns, rate limiting algorithms, and reliability strategies

---

## 1. Consistency & Isolation Paradigms

- **PACELC Theorem**:
  - **P**artition active -> Trade **A**vailability or **C**onsistency.
  - **E**lse (Normal operation) -> Trade **L**atency or **C**onsistency.
- **Consensus Protocols**:
  - **Raft / Paxos**: Strong consistency. Replicated state machine using majority quorum nodes (`(N/2)+1`).

---

## 2. Distributed Transaction Patterns

- **2-Phase Commit (2PC)**:
  - **Mechanism**: Coordinator asks participants to prepare, then commits.
  - **Problem**: Synchronous blocking. Single point failure at coordinator causes locks to persist.
- **Saga Pattern (Compensation Transactions)**:
  - **Mechanism**: Sequential microservice step runs. Step failure triggers reverse compensation steps (Refund, Release).
  - **Trade-off**: Eventual consistency. Requires idempotent APIs.

---

## 3. Rate Limiting Algorithms

| Algorithm | Pros | Cons | Best For |
|---|---|---|---|
| **Token Bucket** | Handles traffic bursts smoothly | Requires state memory | API gateway general limiting |
| **Leaky Bucket** | Guarantees constant egress output rate | Delaying responses | Egress queue traffic shaping |
| **Sliding Window Log** | High accuracy | High memory cost | Low-volume sensitive APIs |
| **Fixed Window** | Low memory footprint | Bound spikes (double limit at window border) | Basic IP block limiting |

---

## 4. Key Reliability Checklist

- **Idempotency**: Use a unique transaction correlation token to discard duplicate incoming API calls.
- **Circuit Breaker**: Detect service latency spikes and fail-fast instantly without calling degraded downstream databases.
- **Backpressure**: Consumers throttle intake rate, alerting upstreams or buffering logs in Kafka queues to prevent crashing under sudden spikes.
