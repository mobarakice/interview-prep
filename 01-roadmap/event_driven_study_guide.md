# Event-Driven Design: Kafka, RabbitMQ, & SQS Study Guide

This guide covers event-driven architecture principles, distributed messaging technologies (Kafka, RabbitMQ, SQS), ordering guarantees, and common architectural patterns (Outbox, CQRS, Idempotency).

---

## 1. Core Technology Comparison

To design scalable systems for clients like bKash and Unilever, you must select the right messaging infrastructure:

| Feature | Apache Kafka | RabbitMQ | AWS SQS (Standard / FIFO) |
| :--- | :--- | :--- | :--- |
| **Architecture** | Distributed Commit Log | AMQP Broker (Exchanges & Queues) | Managed Queue Service |
| **Model** | Pull-based (Consumer pulls) | Push-based (Broker pushes) | Pull-based (Polling/Long polling) |
| **Ordering** | Guaranteed *within a partition* | Guaranteed *within a queue* | FIFO: Strict ordering; Standard: Best-effort |
| **Message Lifespan** | Durable (Retained for set TTL) | Deleted upon acknowledgment | Deleted upon acknowledgment |
| **Delivery Guarantee** | At-least-once, Exactly-once (Transactional) | At-least-once, At-most-once | At-least-once (FIFO: Exactly-once) |
| **Scalability** | Ultra-high throughput (millions/sec) | Medium throughput (10k-50k/sec) | High scale (elastic managed scaling) |

---

## 2. Deep Dive: Apache Kafka

Kafka is a distributed streaming platform modeled as a replicated append-only log.

### Architecture Components

```
[ Producer ] ──► [ Topic ] ──► [ Partition 0 ] (Log Append)
                           ──► [ Partition 1 ] (Log Append)
                                     │
                                     ▼
                      [ Consumer Group (Parallelism) ]
                      - Consumer A reads Partition 0
                      - Consumer B reads Partition 1
```

* **Partitions:** Topics are divided into partitions. Partitions allow Kafka to scale horizontally across brokers.
* **Consumer Groups:** Dynamic consumer groups distribute partitions among consumers. Each partition is consumed by only one consumer inside a consumer group at any given time, guaranteeing serial ordering for that partition.
* **Replication Factor & ACKS:**
  * `acks=0`: Producer does not wait for acknowledgment. High throughput, high risk of data loss.
  * `acks=1`: Producer waits for the partition leader to write the log.
  * `acks=all` (or `-1`): Leader waits for all in-sync replicas (ISRs) to write the log. Highest durability.

---

## 3. Deep Dive: RabbitMQ

RabbitMQ is a smart broker based on the AMQP (Advanced Message Queuing Protocol) standard.

### Routing Architecture

```
[ Producer ] ──► [ Exchange ] (Routes using bindings)
                      │
         ┌────────────┴────────────┐
         ▼ (Routing Key)           ▼ (Fanout/Direct)
    [ Queue A ]               [ Queue B ]
         │                         │
         ▼                         ▼
   [ Consumer A ]            [ Consumer B ]
```

* **Exchanges:** Receives messages from producers and routes them to queues:
  * **Direct:** Routes based on exact routing key matches.
  * **Fanout:** Broadcasts to all bound queues.
  * **Topic:** Routes based on wildcard routing key matches (e.g. `order.*.completed`).
* **Acknowledgment Mechanics:** RabbitMQ retains messages in memory/disk until the consumer returns an explicit acknowledgment (`basic.ack`). If the connection drops before ack, RabbitMQ returns the message to the queue.

---

## 4. Deep Dive: AWS SQS

AWS SQS is a fully managed message queuing service.

* **Standard Queues:** Offers near-unlimited throughput. However, it does not guarantee strict ordering, and messages can occasionally be delivered more than once (At-least-once).
* **FIFO Queues:** Guarantees strict ordering and exactly-once processing (up to 3,000 messages/sec with batching). Requires deduplication IDs.
* **Visibility Timeout:** When a consumer pulls a message, SQS marks it as invisible for a set duration (e.g. 30s) to prevent other consumers from processing it. The consumer must process the message and delete it from the queue before the timeout expires; otherwise, the message becomes visible again.

---

## 5. Event-Driven Patterns

### Transactional Outbox Pattern
Enforces atomicity when updating a database and publishing events to a message broker, eliminating the risk of split-brain failures (e.g. DB updates succeed, but network drops before publishing to Kafka).

### CQRS (Command Query Responsibility Segregation)
Separates read operations (Queries) from write operations (Commands) to optimize performance:
* **Write Database (PostgreSQL):** Handles normalization and transactions. Emits mutation events.
* **Read Database (Elasticsearch / Redis):** Denormalized data optimized for rapid query fetches. Updated asynchronously by consuming write events.

```
[ User Action ] ──► [ Write API ] ──► [ Write DB (Postgres) ]
                                             │
                                             ▼ (WAL CDC / Outbox)
                                      [ Event Broker ]
                                             │
                                             ▼
[ User Query  ] ◄── [ Read API  ] ◄── [ Read DB (Elasticsearch) ]
```

### Idempotent Consumer Pattern
Ensures that processing the same message multiple times does not change the system state.
* *Implementation:* Use **Unique Message IDs**. The consumer checks a database or Redis cache before execution:
  ```sql
  INSERT INTO processed_messages (message_id) VALUES ('msg_123')
  ON CONFLICT DO NOTHING;
  ```
  If the insert inserts 0 rows, the consumer skips processing to avoid duplicate mutations.

---

### Questions & Answers: Event-Driven Design

#### Q1: What is a "Poison Pill" message, and how do you handle it in a production consumer application?
**Answer:**
> "A **Poison Pill** is a malformed message (e.g. invalid JSON syntax or missing critical fields) that a consumer cannot parse. 
> If not handled:
> 1. In **Kafka:** The consumer throws an exception, fails to commit its offset, and retries the same message indefinitely, blocking the entire partition.
> 2. In **RabbitMQ:** The message is returned to the queue and retried in a continuous loop, causing CPU spikes.
> **Mitigation:**
> Wrap the consumer deserializer in a try-catch block. If parsing fails, catch the error, log the invalid payload, and publish the raw message to a dedicated **Dead Letter Queue (DLQ)** topic. Immediately commit the offset / acknowledge the message to allow the consumer to proceed with the next message."

#### Q2: How do you prevent and resolve consumer lag in a production Kafka deployment?
**Answer:**
> "Consumer lag occurs when the rate of messages published to a topic exceeds the rate at which the consumers can process them.
> **Prevention & Resolution:**
> 1. **Horizontal Scaling:** Increase the number of partitions in the Kafka topic and add more consumer instances to the consumer group (up to a 1:1 ratio of consumers to partitions).
> 2. **Process in Batches:** Configure the consumer to poll messages in batches and process them concurrently using an internal worker thread pool, committing offsets only after the entire batch is completed.
> 3. **Tune Fetch Configurations:** Adjust `max.poll.interval.ms` (to prevent Kafka from thinking the consumer is dead during long executions) and increase `max.partition.fetch.bytes`."

---
Event-Driven Design Study Guide
