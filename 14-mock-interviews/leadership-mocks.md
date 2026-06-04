# Engineering Leadership Mock Interviews

> Mock interview transcripts and scenarios focusing on team building, engineering management, conflict resolution, and architectural prioritization
> **Mocks Included**: 1. Resolving Technical Debt Conflicts · 2. Post-Mortem Accountability & Culture

---

## Mock 1: Prioritizing Technical Debt vs. Feature Delivery

### Interview Setting
- **Interviewer**: Director of Engineering / VP of Product
- **Candidate**: Engineering Lead / Principal Architect candidate
- **Goal**: Evaluate candidate's ability to balance technical health (refactoring, upgrading systems) with business goals (feature releases).

### Transcript Excerpt

**Interviewer**: The product team wants to release a new loyalty program feature next month to hit a key marketing window. However, the engineering team reports that the core user registration module is full of technical debt, lacks unit tests, and refactoring it will take at least 3 weeks. If they build the new feature on top of the old code, it will add more technical debt. How do you handle this?

**Candidate**:
- **Goal**: I want to avoid two extreme positions: blindly agreeing to build features on broken code, or demanding a complete stop to business delivery for a full refactor. I prefer a **compromise strategy (Incremental Refactoring)**.
- **Step 1: Impact Assessment**: I will work with the lead engineer to analyze *how* the technical debt impacts the new feature. Will it slow down development? Will it introduce security risks or latency?
- **Step 2: Scoped Refactoring**: I will propose refactoring *only* the specific code path that the new loyalty feature interacts with, rather than rewriting the entire registration module. This might add 5 days to the timeline instead of 3 weeks.
- **Step 3: Define a Technical Debt Budget**: I will negotiate with the product manager to allocate a static portion of every sprint (e.g. 20% of engineering bandwidth) to resolve technical debt. This ensures the system improves continuously without halting feature delivery.

**Interviewer**: What if the Product Manager refuses to extend the deadline by even 5 days, stating the marketing date is unmovable?

**Candidate**:
- If the date is immovable, I will suggest scoping down the loyalty feature itself. Can we launch a minimal viable product (MVP) first?
- If we must release the full feature on the legacy code, I will accept it but log the technical debt in our tracker with a pre-approved agreement from the product manager to spend the entire following sprint *only* on code cleanup. I will document this decision to ensure transparency.

---

## Evaluation Rubric

- **Collaboration (Critical)**: Candidates must demonstrate empathy for business goals. Avoid purely dogmatic technical arguments.
- **Pragmatism**: Look for structured planning, risk management, and the ability to negotiate compromises.
- **Tracking Practices**: Understanding the value of documenting technical debt logs and Architectural Decision Records (ADRs).
