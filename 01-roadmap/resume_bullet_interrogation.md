# Resume Bullet Interrogation & Technical Defense Guide

This guide contains realistic, probing interview questions and quantitative model answers to help you defend every key achievement on your resume.

---

## 1. TechnoNext: API Gateway & Cost Optimization

### Bullet 1
> *"Architected and led a high-performance API Gateway serving 10+ microservice platforms (Foodi, CartUp, FirstTrip, TripLover, etc.), reducing third-party gateway cost by 70–80% annually."*

* **Interviewer Probe:** *Why did you build a custom gateway instead of using cloud-native solutions like AWS API Gateway? What was the exact licensing cost difference?*
* **Technical Defense:**
  > "AWS API Gateway charges \$3.50 per million REST API calls, plus data transfer fees. At 10M daily requests (300M monthly calls), that amounts to \$1,050/month per platform, or \$10,000+ monthly across 10 platforms. This does not include costs for advanced rate limiting and WAF integrations.
  > By building our gateway on open-source Apache APISIX containerized on a 3-node Kubernetes cluster (EC2 `t3.medium` instances), our hosting costs were under \$300/month. This replaced licensing fees and reduced our annual gateway infrastructure costs by 70–80%."

### Bullet 2
> *"Designed optimized caching layers (in-memory + Redis), resulting in 35–50% lower latency and 2x throughput improvement under peak load."*

* **Interviewer Probe:** *How did you benchmark the 35–50% latency drop? What were the average latency values before and after?*
* **Technical Defense:**
  > "Before optimization, our average end-to-end latency under peak load (1,500 RPS during lunch rushes on Foodi) was 120ms (p95 was 350ms), bottlenecked by database read queries for user sessions and API key permissions.
  > I implemented a 2-tier cache: L1 local LRU memory cache inside the gateway pods with a 10s TTL, and a Redis cluster L2 cache. This cache-aside setup intercepted 85% of read queries.
  > Under the same peak load, average response latency dropped to 65ms (a ~45% reduction), and our gateway's maximum throughput capacity doubled from 1,000 to 2,000 RPS without database CPU exhaustion."

### Bullet 3
> *"Built custom gateway features—Authentication, Authorization, API-Key Validation, Dynamic Rate Limiting, Unique Device Identification & Throttling, URI Blocking—processing 10M+ daily API requests reliably."*

* **Interviewer Probe:** *How did you implement unique device identification? How did you protect against IP spoofing?*
* **Technical Defense:**
  > "We generated a unique device fingerprint on the mobile client by hashing device attributes (e.g. UUID, advertising ID, platform details) and stored it in secure storage. The client sent this fingerprint in a custom header: `X-Device-Signature`.
  > To protect against IP spoofing, we avoided relying on the `X-Forwarded-For` header. The gateway validated client IP signatures using HMAC-signed payloads and compared device identification states directly against a Redis database containing device registration records."

---

## 2. TechnoNext: Map Phantom Multi-Tenant SaaS

### Bullet
> *"Architected the Map Phantom Project (Map Portal)... User Service, API-Key Management Service, Billing and Reporting Service, and Billing Consumer Service."*

* **Interviewer Probe:** *How did you isolate tenant billing data inside the Billing Consumer Service to ensure a billing update for Tenant A never wrote to Tenant B's balance?*
* **Technical Defense:**
  > "The Billing Consumer read usage metrics from a partitioned Kafka topic where the partition key was set to `tenant_id`. This guaranteed that all events for a given tenant were processed sequentially by a single worker thread.
  > When updating balances in PostgreSQL, the update executed inside a transaction using **Row-Level Security (RLS)**. Before executing the update, the connection set the active tenant ID:
  > `SET LOCAL app.current_tenant_id = 'tenant_A';`
  > The update query then targeted the balance row:
  > `UPDATE tenant_balances SET balance = balance - 0.05 WHERE tenant_id = 'tenant_A';`
  > If the worker thread attempted to write to Tenant B's data, the database RLS policy blocked the transaction."

---

## 3. BJIT: Monolith Migration & IoT Systems

### Bullet 1
> *"Migrated Rakuten Beauty monolith to Java microservices with 60% improved performance."*

* **Interviewer Probe:** *How did you split the database schema without breaking foreign key relationships in the monolith?*
* **Technical Defense:**
  > "We identified the Booking domain as a candidate for migration. In the monolith, the `bookings` table had foreign keys referencing the `salons` and `users` tables.
  > 1. We kept the shared database during the initial migration phase but separated the tables logically into two schemas: `booking_schema` and `core_schema`.
  > 2. We removed the database-level foreign key constraints, replacing them with application-level validation checks.
  > 3. To load booking details along with salon metadata, the Booking service queried the Salon service via REST (cached using Redis) instead of running a SQL JOIN.
  > 4. Once the service boundaries were stable, we migrated the tables to separate database instances."

### Bullet 2
> *"Designed and led delivery of Pocketalk Ventena 2, a distributed IoT system with 99.9% uptime."*

* **Interviewer Probe:** *How did you design for high availability in the IoT data pipeline? How did you handle 99.9% uptime constraints?*
* **Technical Defense:**
  > "To ensure 99.9% uptime (no more than 8.7 hours of downtime per year), we designed a redundant pipeline:
  > 1. Devices connected to a cluster of MQTT brokers load-balanced by AWS NLB.
  > 2. The MQTT brokers wrote messages to a partitioned Kafka topic with a replication factor of 3 and `acks=all`.
  > 3. We deployed our ingestion and processing services across multiple AWS Availability Zones (AZs) using Kubernetes pod anti-affinity rules, ensuring nodes were distributed.
  > 4. If an entire AZ went offline, the load balancer redirected device traffic to the active AZs, and Kafka reassigned partition leaders within 10 seconds."

---

## 4. Field Info: Offline Data Sync

### Bullet
> *"Launched Android apps (IRDF, NKG Bloom) with custom offline syncing and real-time location modules... Achieved 99% reduction in data loss."*

* **Interviewer Probe:** *How did you calculate the 99% data loss reduction? What were the previous failure metrics?*
* **Technical Defense:**
  > "Previously, field agents used a mobile web form that sent data directly to the server. If the network dropped during transmission, the request failed, and agents lost their inputs. The error rate was ~15% (150 out of 1,000 form submissions failed).
  > I refactored the app to use an offline-first architecture. We added a local SQLite database to store mutations in a queue table.
  > When connection was restored, the sync service retried transmissions using exponential backoff. In our pilot testing of 10,000 form submissions, only 2 forms failed due to unresolvable client data corruption. This reduced the error rate from 15% to 0.02% (a ~99% reduction in data loss)."

---
Resume Defense Guide
