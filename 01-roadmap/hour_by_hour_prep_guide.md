# 24-Hour Hour-by-Hour Master Interview Preparation Guide
**Target Role:** Senior Full-Stack Product Engineer | **Company:** Intelligent Machines (IM), Banani, Dhaka  
**Candidate:** Mobarak Hosen (10+ Years Experience | Software Architect & CSM®)

---

## 📅 HOUR-BY-HOUR PREPARATION ROADMAP

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ BLOCK 1: STRATEGY, COMPANY POSITIONING & AI WORKFLOW (HOURS 1–4)                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ Hour 1 : Company Intelligence (IM, bKash/Unilever SaaS Pivot) & Positioning Pitch      │
│ Hour 2 : AI-Native Workflow Preparation & Live Engineering Demo Setup                    │
│ Hour 3 : Cloud Cost Control & Infrastructure Optimization (The 70-80% Case Study)       │
│ Hour 4 : Multi-Tenant SaaS Architecture & Data Isolation Deep Dive                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ BLOCK 2: SYSTEM DESIGN & DEVOPS MASTERCLASS (HOURS 5–10)                                │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ Hour 5 : System Design 1 — Multi-Tenant B2B SaaS Platform with Billing & API Quotas     │
│ Hour 6 : System Design 2 — Enterprise API Gateway, Rate Limiting & Coraza WAF Security │
│ Hour 7 : System Design 3 — Real-Time High-Throughput Data Pipeline (bKash/Unilever)   │
│ Hour 8 : System Design 4 — Offline-First Mobile & Field Operations Sync Engine          │
│ Hour 9 : Kubernetes, Docker, CI/CD Pipelines & Cloud Infrastructure (AWS ➔ GCP)        │
│ Hour 10: PostgreSQL Deep Dive, Query Plan Optimization & Row-Level Security (RLS)      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ BLOCK 3: FULL-STACK TECHNICAL CORE (HOURS 11–14)                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ Hour 11: Backend Core — Node.js/TypeScript Event Loop vs Java/Spring Boot Concurrency   │
│ Hour 12: Microservices Architecture, API Protocols (gRPC/REST/GraphQL) & Resilience    │
│ Hour 13: Frontend Core — React 18/19, Next.js App Router, SSR vs Client Components      │
│ Hour 14: State Management (Zustand/React Query) & Core Web Vitals Performance           │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ BLOCK 4: SLEEP & COGNITIVE CONSOLIDATION (HOURS 15–18)                                  │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ Hours 15–18: Mandatory Sleep, Mental Recovery & Memory Retention Break                  │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ BLOCK 5: BEHAVIORAL & RESUME INTERROGATION (HOURS 19–21)                                │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ Hour 19: Behavioral Stories — Founder Collaboration, Speed vs Quality & Trade-offs     │
│ Hour 20: Behavioral Stories — Production Incident Response & Engineer Mentorship        │
│ Hour 21: Resume Bullet Deep Dive — Technical Defense for Every Resume Bullet           │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ BLOCK 6: MOCK SIMULATION & FINAL CHEAT SHEET (HOURS 22–24)                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│ Hour 22: Mock Interview Simulation — Screening & Technical Deep Dive Rounds            │
│ Hour 23: Mock Interview Simulation — System Design & Founder Final Rounds              │
│ Hour 24: Final 1-Hour Warmup, Metric Flashcards & Logistics Check                       │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ───────── BLOCK 1: STRATEGY, COMPANY POSITIONING & AI WORKFLOW ─────────

### 🕒 HOUR 1: Company Intelligence & 2-Minute Elevator Pitch
* **Focus Area:** Aligning 10+ years of full-stack engineering with Intelligent Machines' transition from custom enterprise software (for **bKash** and **Unilever**) to launching proprietary B2B SaaS products in late 2026.
* **Key Objective:** Master a crisp, non-generic 2-minute elevator pitch highlighting architecture ownership, hands-on building, cloud cost control, and team leadership.

#### Mock Question 1: "Tell me about yourself and why you are the right fit to anchor the product engineering team at Intelligent Machines."
* **Senior Model Answer:**
  > "I am a Senior Full-Stack Product Engineer and Certified Scrum Master with over 10 years of experience architecting, building, and scaling enterprise systems and SaaS platforms. 
  > In my recent role as Software Architect at TechnoNext, I owned the technical architecture of a high-performance API Gateway handling 10M+ daily requests across 10+ microservice platforms. Crucially for a growing SaaS company, I designed in-house caching and routing solutions that reduced annual cloud infrastructure costs by 70–80% while maintaining 99.9% uptime. I also designed the multi-tenant architecture for the Map Phantom SaaS project, building services for tenant user management, API-key metering, and consumption billing.
  > Prior to that at BJIT, I led teams of 4–8 engineers shipping global applications like Pocketalk Translation—scaling it from scratch to 50k+ downloads and winning a Google Play Award—and migrating Rakuten Beauty's monolith to microservices with a 60% performance gain.
  > What excites me about Intelligent Machines is your pivot toward proprietary enterprise B2B SaaS products. You need someone who stays 100% hands-on writing code weekly, owns cloud infrastructure and cost optimization end-to-end, embeds AI-native workflows into the team, and can collaborate directly with leadership to ship fast. That exact combination of builder mindset and architectural leadership is what I've delivered throughout my career."

#### Mock Question 2: "IM works heavily with enterprise giants like bKash and Unilever. How does your past experience translate to these high-stakes enterprise environments?"
* **Senior Model Answer:**
  > "High-stakes enterprise clients like bKash or Unilever demand three non-negotiable qualities: absolute data security/tenant isolation, zero tolerance for unannounced downtime, and cost-effective scaling under peak loads. 
  > During my time delivering solutions for clients like Rakuten Japan and global enterprise IoT platforms like Pocketalk Ventena 2, I operated under strict enterprise SLAs (99.9% uptime). I designed robust multi-tenant security layers using Coraza WAF and OAuth2/HMAC authorization, implemented zero-downtime deployment pipelines in Kubernetes, and built resilient messaging pipelines using Kafka and Redis. I understand how to design software that meets strict enterprise compliance while keeping engineering velocity fast."

---

### 🕒 HOUR 2: AI-Native Engineering Workflow & Live Demo Strategy
* **Focus Area:** Demonstrating real AI-assisted engineering workflows. JD explicitly states: *"We will ask you to show your workflow."*
* **Key Objective:** Articulate an AI-assisted SDLC that uses tools like GitHub Copilot, Claude Code, and AGY/Gemini tools to accelerate delivery without sacrificing code quality or security.

#### Mock Question 1: "Walk us through your daily AI-assisted development workflow. Where does AI add maximum value, and where does it fail?"
* **Senior Model Answer:**
  > "I treat AI tools as a high-speed junior pair-programmer under my strict senior supervision. My workflow follows a 4-step pipeline:
  > 1. **Prototyping & Boilerplate:** I use AI to generate initial OpenAPI 3.0 specs, PostgreSQL DDL table schemas, and TypeScript/Java DTO boilerplate. This saves 40% of initial setup time.
  > 2. **Test-Driven Generation:** I prompt AI to write unit test suites (Jest/JUnit) including edge cases (null payloads, duplicate tenant keys, concurrency races) *before* implementing business logic.
  > 3. **AI Code Review & Linting:** Before opening a PR, I run AI security checks over my diffs to catch unhandled promise rejections, missing SQL index hints, or improper input sanitization.
  > 4. **Where AI Fails:** AI fails at complex domain boundary design, multi-tenant isolation semantics, distributed transaction boundaries (e.g. Saga vs Outbox patterns), and subtle memory leaks in event loops. I own 100% of architectural choices, security boundaries, and code correctness."

#### Mock Question 2: "If we ask you to open your IDE right now and build a feature using AI tools, how will you structure your prompts and live workflow?"
* **Senior Model Answer:**
  > "I start by giving the AI explicit context: the tech stack (e.g. Next.js 14 App Router, TypeScript, Prisma/PostgreSQL), the business constraints (multi-tenant filtering required), and coding standards (functional, strict type safety, input validation with Zod). 
  > Instead of asking AI to write an entire feature in one prompt, I break it down into modular steps: 
  > First: *'Draft the Zod schema and TypeScript types for a Tenant API Key creation payload.'* 
  > Second: *'Write a Jest test suite checking valid keys, expired keys, and missing tenant headers.'* 
  > Third: *'Implement the service function using Prisma with parameterized queries.'* 
  > This iterative approach ensures total control over output quality and test coverage."

---

### 🕒 HOUR 3: Cloud Cost Control & Infrastructure Optimization
* **Focus Area:** Deep dive into your **70–80% annual cloud infrastructure cost reduction** at TechnoNext. FinOps principles for Kubernetes, AWS, and Docker.
* **Key Objective:** Explain how to build cost-aware architecture from Day 1 for IM's SaaS products.

#### Mock Question 1: "How did you achieve a 70–80% reduction in API Gateway and infrastructure costs at TechnoNext?"
* **Senior Model Answer:**
  > "At TechnoNext, our microservices platform (serving Foodi, CartUp, TripLover, etc.) processed 10M+ daily API requests. The platform was originally relying on expensive third-party managed gateway solutions whose per-request licensing and cloud compute overhead were scaling exponentially.
  > I led the architecture of an in-house API Gateway using open-source Apache APISIX combined with Coraza WAF for security. To minimize compute resource requirements, I designed a 2-tier caching system: L1 in-memory LRU cache inside the gateway pods for high-frequency token checks, backed by an optimized Redis L2 cluster. 
  > We containerized the gateway on Docker and Kubernetes with Horizontal Pod Autoscaling (HPA) driven by request concurrency rather than just CPU usage. This eliminated expensive third-party licensing fees, optimized container resource requests/limits, and reduced annual cloud infrastructure costs by 70–80% while decreasing average API latency by 35–50%."

#### Mock Question 2: "How do you design a B2B SaaS platform to be cloud-cost-aware from Day 1?"
* **Senior Model Answer:**
  > "Cost control starts in system design, not in post-billing audits. I enforce four practices:
  > 1. **Multi-Tenant Resource Efficiency:** Using shared compute and row-level isolated database instances for lower-tier tenants, reserving dedicated database instances only for top-tier enterprise clients.
  > 2. **Aggressive Multi-Tier Caching:** Caching tenant authorization tokens, rate limits, and configuration data in Redis to prevent expensive database disk I/O.
  > 3. **Autoscaling Based on Custom Metrics:** Configuring Kubernetes KEDA/HPA to scale pods down during off-peak hours based on queue length or active requests.
  > 4. **Storage Lifecycle Policies:** Offloading cold audit logs and telemetry data from PostgreSQL to cheap cloud object storage (AWS S3 / GCS) compressed as Parquet files."

---

### 🕒 HOUR 4: Multi-Tenant SaaS Architecture & Data Isolation
* **Focus Area:** Multi-tenancy patterns, Map Phantom Project experience, tenant data separation strategies.
* **Key Objective:** Master the trade-offs of Database-per-tenant vs Schema-per-tenant vs Discriminator Column (RLS).

#### Mock Question 1: "Compare Database-per-tenant, Schema-per-tenant, and Shared DB with Discriminator Column. How do you choose for IM's SaaS platform?"
* **Senior Model Answer:**
  > "The choice comes down to Isolation Level vs Infrastructure Cost:
  > 1. **Database-per-Tenant:** Maximum security, easiest backup/restore per client, highest cost and operational overhead (managing 500 DB connections/migrations). Best for high-paying enterprise clients (e.g. bKash).
  > 2. **Schema-per-Tenant (PostgreSQL Schemas):** Medium isolation, single database instance with separate schemas. Good isolation, but schema migrations across 1,000 schemas become slow.
  > 3. **Shared DB with Discriminator Column (`tenant_id` + Row-Level Security):** Lowest cost, highest density, fastest cross-tenant reporting. Requires strict PostgreSQL RLS policies to prevent data leaks.
  > **My Recommendation for IM:** A **Hybrid Multi-Tenant Model**. Use a shared PostgreSQL database with RLS (`tenant_id`) for standard B2B SaaS tiers to keep compute costs minimal, while architecting the connection pooling layer so enterprise clients (like Unilever) can be seamlessly provisioned on an isolated Database-per-tenant instance."

#### Mock Question 2: "How did you design tenant management and billing tracking in the Map Phantom Project?"
* **Senior Model Answer:**
  > "In the Map Phantom Project, Map Core Service access was monetized based on API consumption. I designed four core services:
  > - **User/Tenant Service:** Handled tenant organization onboarding and role-based access control.
  > - **API Key Management Service:** Issued HMAC-signed API keys tied to tenant IDs and subscription tiers. Keys were cached in Redis with metadata (rate limits, active quotas).
  > - **Usage Metering Pipeline:** When requests passed through the gateway, an asynchronous event was published to Kafka (`tenant_id`, `api_key`, `endpoint`, `timestamp`).
  > - **Billing Consumer Service:** Aggregated raw Kafka usage events into 5-minute time windows, updated PostgreSQL usage tables, and enforced monthly package limits. When a tenant hit 90% quota, automated webhooks notified the client."

---

## ───────── BLOCK 2: SYSTEM DESIGN & DEVOPS MASTERCLASS ─────────

### 🕒 HOUR 5: System Design 1 — Multi-Tenant B2B SaaS Platform
* **Focus Area:** End-to-end design of an Enterprise SaaS platform with tenant provisioning, API key rate-limiting, and billing.

#### System Architecture Blueprint (ASCII)
```
[ Enterprise User App ] ──► [ APISIX / Node.js API Gateway ] 
                                     │ (Injects X-Tenant-ID)
                                     ▼
                      ┌──────────────────────────────┐
                      │  Redis Cluster (L2 Cache)    │
                      │  - API Key Metadata          │
                      │  - Tenant Token Bucket Rates │
                      └──────────────┬───────────────┘
                                     ▼
                      [ Multi-Tenant App Microservices ]
                      (Row-Level Security Enforcement)
                                     │
                 ┌───────────────────┴───────────────────┐
                 ▼                                       ▼
    [ PostgreSQL Shared DB ]              [ Apache Kafka Event Bus ]
    (RLS: tenant_id filter)               (Topic: tenant-usage-events)
                                                         │
                                                         ▼
                                          [ Billing & Metering Engine ]
```

#### Mock Question 1: "Design a Multi-Tenant B2B SaaS platform handling 100M daily API events with quota enforcement."
* **Senior Model Answer Key Points:**
  * **Gateway Auth & Context Injection:** API Gateway intercepts incoming calls, validates API key in Redis (< 2ms), extracts `tenant_id` & `tier`, and injects an immutable `X-Tenant-ID` header downstream.
  * **Tenant Isolation:** Use PostgreSQL Row-Level Security (`ALTER TABLE orders ENABLE ROW LEVEL SECURITY`). App connection sets `SET LOCAL app.current_tenant_id = 'tenant_123'`.
  * **Asynchronous Metering:** Gateway uses non-blocking async logger to push usage payload to Kafka (`tenant-usage-events`). Dedicated consumer updates timeseries usage tables.
  * **Quota Enforcement:** Redis atomic decrement (`DECRBY`) checks remaining monthly API allowance. If 0, Gateway immediately returns HTTP `429 Too Many Requests`.

---

### 🕒 HOUR 6: System Design 2 — Enterprise API Gateway, Rate Limiting & WAF
* **Focus Area:** High-throughput API gateway design, distributed rate limiting, Coraza WAF security integration.

#### Mock Question 1: "How do you implement a distributed Rate Limiter in Redis that handles 10,000+ RPS without race conditions?"
* **Senior Model Answer:**
  > "A naive `GET` and `SET` in Redis causes race conditions under high concurrency. I use a **Redis Token Bucket algorithm executed via an atomic Lua script**.
  > The Lua script takes `key` (e.g. `ratelimit:{tenant_id}:{minute}`), `max_capacity`, and `refill_rate`. Inside Redis memory, the script fetches the current bucket state, calculates tokens generated since the last timestamp, checks if tokens >= 1, decrements token count, and updates the timestamp atomically. 
  > Because Redis executes Lua scripts single-threaded, the token check and update are 100% atomic with 0 race conditions and execution overhead under 1 millisecond."

---

### 🕒 HOUR 7: System Design 3 — Real-Time High-Throughput Analytics Pipeline
* **Focus Area:** Ingesting 20,000 events/sec for enterprise clients (bKash/Unilever) with Kafka, Redis, and PostgreSQL.

#### Mock Question 1: "How do you handle message ordering, backpressure, and exactly-once processing across Kafka partitions?"
* **Senior Model Answer:**
  > "1. **Message Ordering:** Kafka guarantees ordering *only within a single partition*. I route events using a consistent partition key—such as `tenant_id` or `user_id`—ensuring all events for a given tenant land on the exact same partition in order.
  > 2. **Backpressure:** Consumers pull messages from Kafka based on their processing capacity. If database write latency increases, consumers slow down their poll rate without dropping messages.
  > 3. **Exactly-Once Processing:** Achieved using the **Transactional Outbox Pattern** on the producer side, and **Idempotent Consumers** on the consumer side. Consumers track processed `message_id`s in a Redis bloom filter / DB table using `INSERT INTO processed_events (event_id) VALUES (...) ON CONFLICT DO NOTHING`."

---

### 🕒 HOUR 8: System Design 4 — Offline-First Mobile & Field Sync Engine
* **Focus Area:** Offline-first architecture (Field Info GmbH experience), mobile synchronization, conflict resolution.

#### Mock Question 1: "Design an offline-first data sync engine for field sales agents operating in rural Bangladesh with low connectivity."
* **Senior Model Answer Key Points:**
  * **Client Persistence:** Local SQLite database storing mutation queues (`pending_sync_table` holding `{id, action, entity, payload, client_timestamp, status}`).
  * **Delta Sync Protocol:** When network resumes, client sends `POST /sync` with payload containing `last_synced_at` and `pending_mutations` array. Server processes array inside a DB transaction.
  * **Conflict Resolution:** Server uses **Optimistic Concurrency Control (OCC)** with `version` fields. If server version > client version, server executes business conflict rules (e.g. *Field Agent edit wins for stock count, Server wins for price*) and returns updated state + new `sync_timestamp`.

---

### 🕒 HOUR 9: Kubernetes, Docker, CI/CD & DevOps Infrastructure
* **Focus Area:** End-to-end production infrastructure ownership. Containerization, Kubernetes HPA, GitOps CI/CD, AWS to GCP mapping.

#### Cloud Provider Mapping Table (AWS ➔ GCP)
```
+----------------──────────────+----------------──────────────+
| AWS Service (Your Resume)    | GCP Equivalent (IM Preferred)|
+----------------──────────────+----------------──────────────+
| EC2 (Virtual Machines)       | Compute Engine               |
| EKS (Elastic Kubernetes)     | GKE (Google Kubernetes Engine)|
| S3 (Object Storage)          | Cloud Storage (GCS)          |
| RDS (PostgreSQL/MySQL)       | Cloud SQL                    |
| AWS Lambda (Serverless)      | Cloud Functions / Cloud Run  |
+----------------──────────────+----------------──────────────+
```

#### Mock Question 1: "How do you structure a zero-downtime CI/CD deployment pipeline for Kubernetes microservices?"
* **Senior Model Answer:**
  > "I use a GitOps-driven CI/CD pipeline (e.g. GitHub Actions / Jenkins + ArgoCD):
  > 1. **Build & Test Stage:** On git push to `main`, run automated unit and integration tests. Build multi-stage Docker images tagged with the git commit SHA. Run vulnerability scanning (Trivy/Snyk).
  > 2. **Push Artifacts:** Push container image to AWS ECR or GCP Artifact Registry.
  > 3. **Kubernetes Deployment:** Update Kubernetes manifest repository. ArgoCD syncs the cluster.
  > 4. **Zero-Downtime Strategy:** Use **Kubernetes Rolling Updates** with `maxSurge: 25%` and `maxUnavailable: 0`. Configure explicit `readinessProbe` (checks HTTP `/healthz` & DB connectivity) and `livenessProbe`. K8s routes traffic to new pods only after readiness probes pass."

---

### 🕒 HOUR 10: PostgreSQL Deep Dive, Query Optimization & RLS
* **Focus Area:** PostgreSQL indexing strategies, `EXPLAIN ANALYZE`, connection pooling, Row-Level Security.

#### Mock Question 1: "Walk through how you analyze and optimize a PostgreSQL query taking 8 seconds to run on a 50M row table."
* **Senior Model Answer:**
  > "1. **Inspect Query Execution Plan:** Run `EXPLAIN (ANALYZE, BUFFERS)` to see actual runtime, index usage, and disk I/O.
  > 2. **Identify Red Flags:** Look for `Seq Scan` (Sequential Scan), `Sort Method: external merge Disk` (spilling to disk due to low `work_mem`), or high `Shared Read Blocks`.
  > 3. **Indexing Optimization:** If filtering on `tenant_id` and `created_at`, add a composite B-Tree index: `CREATE INDEX CONCURRENTLY idx_tenant_created ON orders (tenant_id, created_at DESC)`.
  > 4. **Query Rewriting & Partitioning:** Avoid functions on indexed columns (`WHERE DATE(created_at) = ...` breaks index usage). For 50M+ rows, implement **PostgreSQL Table Partitioning** by range (e.g. monthly partitions)."

---

## ───────── BLOCK 3: FULL-STACK TECHNICAL CORE ─────────

### 🕒 HOUR 11: Backend Core — Node.js/TypeScript vs Java/Spring Boot
* **Focus Area:** Comparing runtime architectures, event loop internals, multi-threading, memory management.

#### Mock Question 1: "Compare Node.js event loop with Java Spring Boot concurrency. When would you use each?"
* **Senior Model Answer:**
  > "Node.js runs on a single-threaded event loop powered by `libuv`. Non-blocking I/O operations (file, DB, network) are delegated to background OS threads or libuv worker pools. It excels in I/O-heavy microservices, API gateways, and real-time streaming due to low memory overhead and high concurrent connection handling.
  > Java Spring Boot historically used a multi-threaded thread-per-request model (Tomcat thread pool), though modern Java uses Project Loom virtual threads (`VirtualThreadPerTaskExecutor`). Java excels in CPU-intensive computations, complex domain models with heavy OOP abstractions, and multi-threaded parallel processing.
  > For IM's SaaS products and API proxies, Node.js with TypeScript provides fast iteration, low memory footprint, and excellent async I/O performance."

---

### 🕒 HOUR 12: Microservices Patterns, API Protocols & Resilience
* **Focus Area:** gRPC vs REST vs GraphQL, Circuit Breaker pattern, Transactional Outbox pattern.

#### Mock Question 1: "When do you choose gRPC over REST or GraphQL in a microservices architecture?"
* **Senior Model Answer:**
  > "I choose **gRPC** for internal inter-service communication between microservices where low latency, high throughput, and strict contract safety are critical. gRPC uses Protocol Buffers (compact binary serialization) over HTTP/2 (multiplexing), making it up to 5-10x faster than REST/JSON.
  > I choose **REST** for public-facing client APIs, third-party integrations, and web applications due to universal browser support and simple caching.
  > I choose **GraphQL** when frontend clients (like complex SaaS dashboards) require flexible, selective data fetching to avoid over-fetching across deep object graphs."

---

### 🕒 HOUR 13: Frontend Core — React 18/19, Next.js App Router & SSR
* **Focus Area:** Next.js App Router, Server Components vs Client Components, rendering strategies.

#### Mock Question 1: "Explain Server Components (RSC) vs Client Components (`'use client'`) in Next.js App Router. How do you decide which to use?"
* **Senior Model Answer:**
  > "Server Components (default in App Router) execute exclusively on the server during request time or build time. They render to raw HTML/RSC payload, meaning zero JavaScript bundle size is sent to the client. They can access backend databases and file systems directly.
  > Client Components (`'use client'`) are hydrated on the client browser. They are required whenever interactive browser features are used: `useState`, `useEffect`, event listeners (`onClick`), or browser APIs (`localStorage`).
  > **My Strategy:** Keep the core layout, data fetching, and static UI as Server Components. Push `'use client'` down to the smallest possible leaf components (e.g. interactive button or chart widget) to keep client JS bundles small."

---

### 🕒 HOUR 14: State Management & Core Web Vitals Performance
* **Focus Area:** Zustand vs Redux, TanStack Query (React Query), LCP/INP/CLS Web Vitals optimization.

#### Mock Question 1: "Why prefer Zustand + TanStack Query over Redux Toolkit for modern enterprise web apps?"
* **Senior Model Answer:**
  > "Modern React applications separate **Server State** from **Client UI State**:
  > 1. **TanStack Query** handles Server State: async data fetching, caching, deduplication, polling, and background revalidation. This eliminates 80% of manual state management code previously written in Redux thunks.
  > 2. **Zustand** handles transient Client UI State (e.g. sidebar toggle, modal visibility, active filters). It is unopinionated, has boilerplate-free hooks, zero context provider wrapping, and a tiny 1KB bundle size compared to Redux Toolkit."

---

## ───────── BLOCK 4: SLEEP & RECOVERY ─────────

### 🕒 HOURS 15–18: Mandatory Sleep & Memory Consolidation
* **Action:** 6 hours of uninterrupted rest. Research proves cognitive performance, problem-solving agility, and interview recall drop by 40% under sleep deprivation.

---

## ───────── BLOCK 5: BEHAVIORAL & RESUME INTERROGATION ─────────

### 🕒 HOUR 19: Behavioral STAR Stories — Leadership, Speed & Trade-offs
* **Focus Area:** Demonstrating startup mindset, choosing working software over endless polish, founder collaboration.

#### Mock Question 1: "Describe a situation where you had to choose speed and working software over architectural perfection."
* **Senior Model Answer (STAR):**
  * **Situation:** At BJIT while building Pocketalk Translation, we faced a tight competitive deadline to launch on the Google Play Store before a major international trade event.
  * **Task:** Deliver a fully functional translation MVP within 6 weeks without risking app crashes.
  * **Action:** I simplified the initial architecture. Instead of building a complex offline translation engine in V1, I architected a hybrid model relying on lightweight cloud API translation endpoints with aggressive local caching. I deferred complex multi-language custom settings to V2 and focused on rock-solid core voice/camera translation.
  * **Result:** We shipped on schedule, achieved 50k+ downloads, won a Google Play Award, and refactored the offline pipeline iteratively in subsequent release cycles."

---

### 🕒 HOUR 20: Behavioral STAR Stories — Incidents & Engineer Mentorship
* **Focus Area:** Production incident response, mentoring mid-level engineers, enforcing standards.

#### Mock Question 1: "Walk me through how you mentor mid-level engineers to raise the team's engineering bar."
* **Senior Model Answer:**
  > "Mentorship happens through three actionable feedback loops:
  > 1. **Constructive PR Reviews:** I don't just point out errors; I explain *why* a pattern matters (e.g. showing how a missing index causes database locks under load).
  > 2. **Pair Programming on Critical Modules:** When tackling complex microservices or Kubernetes setups, I pair with mid-level engineers to walk through domain modeling and debugging strategies.
  > 3. **Establishing Engineering Standards:** At TechnoNext and BJIT, I authored team guidelines for clean architecture, automated testing standards, and CI/CD pipelines, giving engineers clear benchmarks for quality."

---

### 🕒 HOUR 21: Resume Bullet Deep Dive (Technical Defenses)

#### Resume Bullet: "Migrated Rakuten Beauty monolith to Java microservices with 60% improved performance."
* **Interviewer Question:** How did you execute this migration without breaking existing traffic, and how was 60% measured?
* **Technical Defense:**
  > "We used the **Strangler Fig Pattern**. We deployed an API Gateway in front of the legacy monolith. We broke off domain modules (e.g. Reservation Service, Salon Catalog Service) one by one into independent Spring Boot microservices.
  > Traffic was routed incrementally (10% -> 50% -> 100%) using canary deployments at the gateway. 
  > The 60% performance improvement was measured by tracking average response latency (p95 drop from 850ms to 340ms) and throughput capacity per server under JMeter load testing."

---

## ───────── BLOCK 6: MOCK SIMULATION & CHEAT SHEET ─────────

### 🕒 HOUR 22: Mock Interview Round 1 & 2 (Screening & Tech Deep Dive)
* **Self-Practice Instructions:** Practice answering out loud within 90-second limits per response. Ensure clear transitions between technical reasoning, business context, and career metrics.

### 🕒 HOUR 23: Mock Interview Round 3 & 4 (System Design & Founder Fit)
* **Self-Practice Instructions:** Draw system design diagrams on paper or digital whiteboard. Practice explaining trade-offs clearly: *"Option A gives us lower latency, but Option B cuts cloud costs by 50%."*

### 🕒 HOUR 24: Final 1-Hour Warmup & Metric Flashcards
* **Final Pre-Interview Checklist:**
  - [ ] 70-80% cost reduction story ready.
  - [ ] 10M+ daily requests API Gateway metrics ready.
  - [ ] AI workflow 4-stage explanation ready.
  - [ ] Camera, mic, lighting, internet backup tested.
  - [ ] 3 questions prepared to ask the founder.

---
*Hour-by-Hour Guide prepared for Mobarak Hosen | Intelligent Machines (IM) Senior Full-Stack Product Engineer Interview*
