# System Design Interview Framework

> **Target Role**: Software Architect / Principal Engineer (10+ YOE)
> **Philosophy**: Demonstrate structured thinking, trade-off analysis, and production-grade awareness

---

## 🎯 The RESHADED Framework

Use this systematic approach for every system design interview:

| Step | Focus | Time |
|------|-------|------|
| **R**equirements | Clarify functional & non-functional requirements | 3-5 min |
| **E**stimation | Back-of-envelope calculations (users, TPS, storage) | 3-5 min |
| **S**torage Schema | Database design, data models, indexes | 5 min |
| **H**igh-Level Design | Architecture diagram with major components | 5-8 min |
| **A**PI Design | REST/gRPC endpoints, request/response contracts | 5 min |
| **D**etailed Design | Deep dive into 2-3 critical components | 10-15 min |
| **E**valuation | Trade-offs, bottlenecks, failure modes | 3-5 min |
| **D**istinctive Features | What makes YOUR design special / production-ready | 2-3 min |

---

## 📐 Scale Reference Numbers

### Latency Numbers Every Engineer Should Know

| Operation | Latency |
|-----------|---------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory reference | 100 ns |
| SSD random read | 150 μs |
| HDD random read | 10 ms |
| Send 1 KB over 1 Gbps network | 10 μs |
| Read 1 MB sequentially from memory | 250 μs |
| Read 1 MB sequentially from SSD | 1 ms |
| Read 1 MB sequentially from HDD | 20 ms |
| Round trip within same datacenter | 500 μs |
| Round trip CA to Netherlands | 150 ms |

### Power of Two Reference

| Power | Exact Value | Approx | Bytes |
|-------|-------------|--------|-------|
| 10 | 1,024 | 1 Thousand | 1 KB |
| 20 | 1,048,576 | 1 Million | 1 MB |
| 30 | 1,073,741,824 | 1 Billion | 1 GB |
| 40 | 1,099,511,627,776 | 1 Trillion | 1 TB |
| 50 | 1,125,899,906,842,624 | 1 Quadrillion | 1 PB |

### Quick Estimation Helpers

| Metric | Value |
|--------|-------|
| Seconds in a day | ~86,400 ≈ ~100K |
| Seconds in a year | ~31.5M ≈ ~30M |
| QPS from daily active users | DAU × avg_requests / 86,400 |
| Peak QPS | ~2-3× average QPS |
| Storage per year | daily_data × 365 |

---

## 🏗️ Architecture Patterns Reference

### Communication Patterns

| Pattern | When to Use | Trade-offs |
|---------|-------------|------------|
| **Synchronous (REST/gRPC)** | Request-response, low latency needed | Tight coupling, cascading failures |
| **Asynchronous (Message Queue)** | Decoupling, eventual consistency OK | Complexity, debugging difficulty |
| **Event Streaming (Kafka)** | Event sourcing, audit trails, replay | Storage cost, ordering complexity |
| **WebSocket** | Real-time bidirectional communication | Connection management, scaling |
| **Server-Sent Events** | One-way real-time updates | Limited to one direction |

### Data Storage Patterns

| Storage | Best For | Not For |
|---------|----------|---------|
| **PostgreSQL** | ACID transactions, complex queries, relational data | High-write throughput at extreme scale |
| **MongoDB** | Flexible schema, document-oriented, rapid iteration | Complex joins, strong consistency |
| **Cassandra** | Write-heavy, time-series, geo-distributed | Complex queries, strong consistency |
| **Redis** | Caching, sessions, leaderboards, pub/sub | Primary data store, complex queries |
| **Elasticsearch** | Full-text search, log analytics | Primary data store, transactions |
| **DynamoDB** | Key-value at scale, single-digit ms latency | Ad-hoc queries, complex joins |
| **S3/Blob Storage** | Files, images, videos, backups | Structured queries |

### Consistency Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Strong Consistency** | All reads reflect latest write | Banking, inventory |
| **Eventual Consistency** | Reads may be stale temporarily | Social feeds, analytics |
| **Read-Your-Writes** | User sees own writes immediately | User profile updates |
| **Causal Consistency** | Causally related ops in order | Chat messages |

### Distributed Transaction Patterns

| Pattern | Description | Pros | Cons |
|---------|-------------|------|------|
| **2PC** | Coordinator-based commit | Strong consistency | Blocking, single point of failure |
| **Saga (Choreography)** | Events trigger next step | Loose coupling | Hard to debug |
| **Saga (Orchestration)** | Central orchestrator | Clear flow, easier debugging | Single point of coordination |
| **TCC** | Try-Confirm-Cancel | Flexible | Complex to implement |
| **Outbox Pattern** | Write to DB + outbox table | Reliable messaging | Additional complexity |

---

## 🔧 Non-Functional Requirements Checklist

### Performance
- [ ] P50, P95, P99 latency targets defined
- [ ] Throughput requirements (QPS/TPS) calculated
- [ ] Batch vs real-time processing decision made

### Availability
- [ ] SLA target defined (99.9%, 99.99%, etc.)
- [ ] Failure domains identified (single DC, multi-region)
- [ ] Graceful degradation strategy defined
- [ ] Health check and circuit breaker patterns

### Scalability
- [ ] Horizontal scaling strategy for each component
- [ ] Database sharding/partitioning strategy
- [ ] Caching layer design (L1/L2 caches)
- [ ] CDN strategy for static content
- [ ] Auto-scaling policies defined

### Security
- [ ] Authentication mechanism (OAuth2, JWT)
- [ ] Authorization model (RBAC, ABAC)
- [ ] Data encryption (at rest, in transit)
- [ ] Input validation and sanitization
- [ ] Rate limiting and DDoS protection
- [ ] Audit logging

### Observability
- [ ] Metrics collection (Prometheus/CloudWatch)
- [ ] Distributed tracing (Jaeger/Zipkin)
- [ ] Centralized logging (ELK/CloudWatch Logs)
- [ ] Alerting thresholds defined
- [ ] Dashboard design for key SLIs

---

## 💡 Interview Pro Tips

### For Architect-Level Interviews

1. **Lead the discussion** - Don't wait for the interviewer to guide you. Drive the conversation.
2. **State assumptions explicitly** - "I'm assuming 100M DAU and 10:1 read-to-write ratio"
3. **Discuss trade-offs** - Never just pick a technology. Explain WHY and what you're giving up.
4. **Think about operations** - Deployment, monitoring, debugging, on-call considerations
5. **Consider cost** - Cloud costs, infrastructure efficiency, cost optimization strategies
6. **Address failure modes** - What happens when X goes down? How do you detect and recover?
7. **Evolution story** - How would this design evolve from MVP to production scale?

### Common Follow-Up Areas
- "How would you handle a sudden 10× traffic spike?"
- "What happens if your primary database goes down?"
- "How would you debug a latency regression?"
- "How would you migrate from monolith to this architecture?"
- "What would you change for a global deployment?"

---

## 📚 System Design Case Studies

The following detailed case studies apply this framework to real-world systems:

| # | System | Key Concepts | File |
|---|--------|-------------|------|
| 1 | Payment Gateway | Idempotency, PCI-DSS, settlement, reconciliation | [payment-gateway.md](./payment-gateway.md) |
| 2 | AI Subscription Manager | Usage billing, churn prediction, lifecycle | [ai-subscription-manager.md](./ai-subscription-manager.md) |
| 3 | SaaS Billing Platform | Multi-tenant billing, invoicing, dunning | [saas-billing-platform.md](./saas-billing-platform.md) |
| 4 | API Gateway | Rate limiting, routing, circuit breaking | [api-gateway.md](./api-gateway.md) |
| 5 | Authentication Platform | OAuth2/OIDC, SSO, MFA, RBAC/ABAC | [auth-platform.md](./auth-platform.md) |
| 6 | Notification System | Multi-channel, templates, delivery tracking | [notification-system.md](./notification-system.md) |
| 7 | Ride Sharing Platform | Matching, real-time tracking, surge pricing | [ride-sharing.md](./ride-sharing.md) |
| 8 | Food Delivery Platform | Real-time tracking, geofencing, delivery assignment | [food-delivery.md](./food-delivery.md) |
| 9 | E-commerce Platform | Saga pattern, inventory locking, recommendations | [ecommerce.md](./ecommerce.md) |
| 10 | URL Shortener | Base62 encoding, read-heavy optimization, sharding | [url-shortener.md](./url-shortener.md) |
| 11 | Social Media Feed | Fan-out strategies, timeline generation, ranking | [social-media-feed.md](./social-media-feed.md) |
| 12 | Chat System | WebSocket, message ordering, E2E encryption | [chat-system.md](./chat-system.md) |
| 13 | Video Streaming | Transcoding pipeline, adaptive bitrate, CDN | [video-streaming.md](./video-streaming.md) |
| 14 | File Storage | Chunked upload, dedup, sync, conflict resolution | [file-storage.md](./file-storage.md) |
| 15 | Banking Platform | ACID transactions, KYC/AML, audit trail | [banking-platform.md](./banking-platform.md) |
| 16 | Loan Processing System | State machine, credit scoring, disbursement | [loan-processing.md](./loan-processing.md) |
| 17 | Fraud Detection Platform | Real-time scoring, ML serving, rule engine | [fraud-detection.md](./fraud-detection.md) |
| 18 | Event Ticketing System | Distributed locking, flash sales, waiting room | [event-ticketing.md](./event-ticketing.md) |
| 19 | AI Agent Platform | Agent orchestration, MCP, multi-model routing | [ai-agent-platform.md](./ai-agent-platform.md) |
| 20 | Enterprise RAG Platform | Document ingestion, vector search, citations | [enterprise-rag.md](./enterprise-rag.md) |

---

*Each case study follows the RESHADED framework above. Start with the framework, then deep-dive into the specific system.*
