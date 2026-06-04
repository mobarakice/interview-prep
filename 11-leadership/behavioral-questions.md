# Leadership & Behavioral Playbook

> Comprehensive STAR-format response guide for Lead/Principal Engineer and Architect interviews
> **Key Categories**: Managing Architectural Disputes, Conflict Resolution, Scaling Teams, Handling Outages & Failures

---

## 1. Architectural Disputes

### Situation
At a previous enterprise company, our core e-commerce system experienced major write locks on database tables during peak sales periods. We needed to refactor our checkout system, and the engineering team was divided:
- Group A proposed a CP (Strong Consistency) architecture using 2-Phase Commit transactions to ensure real-time inventory updates.
- Group B proposed an AP (Available/Eventually Consistent) model using the Saga pattern via Kafka queues.

### Task
As the Principal Architect, my job was to resolve this dispute, align both factions, and establish a design that would scale without causing inventory anomalies (like double-selling).

### Action
1. **Fact-Based Analysis**: I scheduled a collaborative whiteboard session. I listed the technical limitations of 2PC (blocking connections, slow latencies under network partitions) and the risks of Sagas (eventual consistency lag could lead to temporary over-selling).
2. **Prototype and Benchmark**: I had one engineer from each side spend 2 days building a lightweight prototype. We ran load tests simulating 10K TPS. The 2PC model crashed due to connection pool exhaustion, while the Saga model processed the load with a 15-second latency delay.
3. **Compensating Controls**: To address Group A's valid concerns about over-selling, we implemented a reserving lock in Redis before entering the Saga. If a seat or item was held, subsequent checkouts were blocked. If a checkout failed in the Saga, a compensating transaction automatically released the lock.
4. **Alignment**: By validating concerns with data and code, we reached a consensus to proceed with the AP model.

### Result
The Saga pattern refactoring reduced checkout latencies by 85% (from 2.4s to under 300ms) and successfully handled black Friday spikes with zero database connection crashes.

---

## 2. Managing Outages & Failures

### Situation
A critical billing update went live at 2 AM, and by 8 AM, our customer service team received hundreds of reports that users were being charged twice for their monthly subscriptions.

### Task
I was the On-Call Lead Engineer. My responsibility was to coordinate the response team, resolve the issue, minimize financial damage, and draft the postmortem.

### Action
1. **Triage & Rollback**: I declared a SEV-1 incident. I verified the active database logs and confirmed that duplicate database entries were generated. I immediately ordered a rollback of the billing deployment to the previous stable release.
2. **Mitigation**: I coordinated with our customer success team to flag duplicate payments on our Stripe payment gateway, initiating automated refund batches for identified users.
3. **Root Cause Analysis (RCA)**: We reviewed code changes. The new release lacked idempotency keys on the Stripe charging API route. If a webhook or API request timed out and retried, Stripe charged the card again.
4. **Implement Safeguards**: We modified our API filters to require a unique `idempotency_key` (derived from `user_id` + `billing_cycle_date`) on all payment requests.

### Result
Within 3 hours, we reverted the deployment, identified all duplicate transactions, initiated refunds, and deployed a hotfix with idempotency keys. In the postmortem, we added automated integration tests validating duplicate request handling to our CI/CD pipeline.

---

## 3. Leading & Scaling Teams

### Situation
Following a company acquisition, our engineering team doubled in size over a 3-month period. We went from a close-knit group of 8 developers to a department of 20+ engineers. Development speed began to slow down due to pull request bottlenecks, meeting fatigue, and unclear ownership boundaries.

### Task
My goal was to re-organize the engineering structure to restore developer productivity, maintain code quality standards, and establish clear division of duties.

### Action
1. **Squad Reorganization**: I divided the large engineering pool into three smaller, cross-functional squads (Core Platform, Customer Experience, Integrations). Each squad received full ownership of specific code repositories and database domains.
2. **Automate Reviews**: We defined clear ownership paths (`CODEOWNERS` files) in GitHub. Instead of waiting for any lead developer to review, PR requests were automatically routed to matching squad members.
3. **Empowerment through Mentorship**: I instituted a peer mentorship program matching senior engineers with new hires, facilitating fast onboarding.

### Result
By splitting the teams into autonomous squads, our average PR cycle time dropped from 4.5 days to under 18 hours, and developer satisfaction scores increased by 40% in monthly surveys.
