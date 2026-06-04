# System Design Whiteboard Playbook

> Playbook for leading and structuring system design whiteboard interviews for Principal/Staff Engineer and Software Architect roles
> **Key Frameworks**: The RESHADED Framework, Communication Strategies, Board Management

---

## 1. The RESHADED Framework
To systematic structure your thoughts during a 45-minute whiteboard session:

| Step | Phase | Action |
|---|---|---|
| **R** | **Requirements** | Clarify scope. Outline functional (user actions) and non-functional goals (TPS, availability, latency). |
| **E** | **Estimations** | Calculate storage volume, read/write QPS, and network bandwidth limits. |
| **S** | **Storage** | Choose database paradigm (SQL vs NoSQL), schema structures, and indexing strategies. |
| **H** | **High-Level Design** | Sketch the primary data flow blocks (Client, Gateway, Load Balancer, Services, DBs). |
| **A** | **APIs** | Define core REST endpoints, parameters, and payloads. |
| **D** | **Detailed Design** | Deep dive into 2-3 complex bottlenecks (e.g., concurrency locks, caching details, scaling). |
| **E** | **Evaluation** | Review the design against requirements. Discuss edge cases and failover recovery plans. |
| **D** | **Diagrams** | Keep diagrams updated and labeled clearly throughout the session. |

---

## 2. Advanced Communication Playbook

### 1. Drive the Conversation
Never remain silent. Verbalize your thought process continuously ("Think-Aloud Protocol").
* **Good**: *"We have a write-heavy profile of 10K writes/sec. If I write to a standard relational database, I risk I/O bottleneck. Let's explore using Cassandra because of its LSM write path, allowing us to flush sequential writes to disk."*
* **Bad**: *[Drawing silently for 3 minutes]*

### 2. Proactively Propose Trade-offs
Avoid absolute answers like "this is the best database." Frame choices around trade-offs:
* **Latency vs Consistency**: *"By using Redis caching, we reduce P99 latency to 5ms, but we accept eventual consistency for about 2 seconds until database updates sync. For a profile dashboard, this is acceptable."*
* **AP vs CP**: *"If a network partition occurs, I prefer AP (Available) for our social feed, allowing users to scroll cached posts, rather than returning a 500 server error."*

---

## 3. Whiteboard Board Layout Pattern
Divide the whiteboard space into organized, logical zones before you begin:

```
┌─────────────────────────┬─────────────────────────┐
│ ZONE 1: REQUIREMENTS     │ ZONE 3: DATA SCHEMAS    │
│ - Functional Goals      │ - SQL/NoSQL Tables      │
│ - Non-Functional SLA    │ - Indexes & Shard Keys  │
├─────────────────────────┼─────────────────────────┤
│ ZONE 2: CALCULATIONS    │ ZONE 4: ARCHITECTURE    │
│ - QPS (Read/Write)      │ - High Level Design     │
│ - Storage/Bandwidth     │ - Detailed Component Flow│
└─────────────────────────┴─────────────────────────┘
```
- **Zone 1 (Top-Left)**: Write down requirements. Keep them visible so you can reference them during the review.
- **Zone 2 (Bottom-Left)**: Run calculations. Keeps numbers isolated from diagrams.
- **Zone 3 (Top-Right)**: Schema definitions.
- **Zone 4 (Bottom-Right)**: High-level architectural diagram.
