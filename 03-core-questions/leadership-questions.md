# Leadership Interview Questions

> 5 architect-level questions on technical vision, mentoring, decision-making, alignment, and technical debt.
> Cross-references: [Behavioral Questions](../11-leadership/behavioral-questions.md) · [Whiteboard Guide](../13-whiteboard-playbook/whiteboard-guide.md)

---

## Q1: How do you define and communicate a technical vision for a platform?

### Interviewer's Expectation
Ability to articulate long-term direction, align teams, and translate business goals into technical strategy.

### Excellent Answer

**My framework for technical vision**:

1. **Understand business context**: Where is the company going in 2-3 years? What capabilities does the product need? What scale do we need to support?

2. **Assess current state**: Architecture audit, tech debt inventory, team capabilities, operational maturity. Where are the biggest gaps?

3. **Define target state**: Target architecture diagrams, technology radar (adopt/trial/assess/hold), quality attributes (latency, availability, scalability targets).

4. **Create the roadmap**: Sequence migrations and improvements. Quick wins first (build momentum), foundational changes next (enable future features), transformative changes last.

5. **Communicate relentlessly**:
   - **Engineers**: Architecture Decision Records, tech talks, pairing sessions
   - **Engineering managers**: Quarterly architecture reviews with progress metrics
   - **Product/Business**: Translate to business impact ("This migration reduces time-to-market for new features from 6 weeks to 2 weeks")

**Example**: When I defined the vision for moving from monolith to microservices:
- Created a 12-month roadmap with 4 phases
- Presented to CTO with business impact metrics
- Held weekly architecture office hours for teams
- Published a living Architecture Principles document
- Measured progress: deployment frequency, lead time, change failure rate

### Common Mistakes
- Vision without execution plan (just a dream), not connecting technical vision to business value, not adapting vision as context changes, creating vision in isolation (ivory tower), vision too detailed (micromanaging) or too vague (meaningless)

### Follow-up Questions
- "How do you get buy-in from engineers who disagree with the vision?"
- "How do you balance long-term vision with short-term delivery pressure?"
- "Give an example where you had to significantly change your technical vision."

---

## Q2: How do you mentor and grow senior engineers into architects?

### Excellent Answer

**Three-layer mentoring approach**:

1. **Expand thinking scope**: Shift from "how to build this feature" to "how should this system evolve." Assign cross-cutting tasks: RFC writing, ADR documentation, tech debt assessment.

2. **Structured growth activities**:
   - **Architecture reviews**: Invite them to present their designs, guide with questions not answers
   - **ADR writing**: Co-author decisions, explain trade-off analysis
   - **Production incidents**: Walk through root cause analysis → architectural implications
   - **Book club**: "Designing Data-Intensive Applications", "Building Evolutionary Architectures"

3. **Progressive responsibility**:
   - Month 1-3: Own technical design of a single service
   - Month 3-6: Lead cross-team technical initiative
   - Month 6-12: Drive architecture decision for a new capability
   - Month 12+: Present to architecture review board independently

**Key insight**: The biggest gap between senior engineers and architects isn't technical — it's the ability to **think in trade-offs**, **influence without authority**, and **communicate decisions to diverse audiences**.

---

## Q3: How do you make architecture decisions when stakeholders disagree?

### Excellent Answer

**Decision-making framework**: 1. Ensure all perspectives are heard (avoid HiPPO — Highest Paid Person's Opinion). 2. Define evaluation criteria upfront (performance, cost, team capability, time-to-market). 3. Create a decision matrix with weighted criteria. 4. Document in ADR — the decision AND the alternatives considered. 5. Disagree and commit — once decided, everyone supports it.

**Escalation model**: Team consensus first → Architect recommendation → Engineering leadership tiebreaker. But most decisions should be made by the team closest to the problem.

**Type 1 vs Type 2 decisions**: Type 1 (irreversible, high-impact) → slow, careful, inclusive. Type 2 (reversible, low-impact) → fast, delegate, iterate.

---

## Q4: How do you build alignment across multiple engineering teams?

### Excellent Answer

**Alignment mechanisms**: Architecture Principles document (10 principles everyone follows), Technology Radar (what to use, what to avoid), Golden Path (template repos, starter kits), Guild/CoP (cross-team architecture community), RFC process (anyone can propose, open discussion).

**Communication cadence**: Weekly architecture office hours, monthly architecture newsletter, quarterly architecture review with all tech leads, annual technology radar refresh.

---

## Q5: How do you balance feature delivery with technical debt reduction?

### Excellent Answer

**The 20% rule**: Reserve 20% of capacity for non-feature work (tech debt, tooling, resilience). Don't ask permission — embed it in estimates.

**Communicate in business terms**: ❌ "We need to refactor the payment module." ✅ "The payment module causes 2 incidents/month averaging $30K each. A 3-sprint investment eliminates this recurring cost."

**Strategic approaches**: Boy Scout Rule (leave code better than you found it), Tech Debt Sprints (quarterly focused sprints), Bundling (attach tech debt work to related features), Sunset obsolete systems (stop maintaining, redirect resources).
