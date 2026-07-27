# Software Design Patterns & Architecture Study Guide

This guide covers core design patterns, architectural paradigms (DDD, Clean Architecture, TDD), and distributed transaction patterns (SAGA, Outbox/Inbox) for senior engineering roles.

---

## 1. SOLID Principles Deep Dive

Relational code bases require clean OOP design patterns to prevent maintenance debt:

* **Single Responsibility Principle (SRP):** A class should have only one reason to change. Separate business logic from logging or presentation code.
* **Open/Closed Principle (OCP):** Software entities should be open for extension but closed for modification. Implement this using inheritance or interface-based polymorphism (e.g. strategy pattern).
* **Liskov Substitution Principle (LSP):** Subtypes must be substitutable for their base types. If class `Ostrich` extends `Bird` but throws an exception on `fly()`, it violates LSP.
* **Interface Segregation Principle (ISP):** Clients should not be forced to depend on interfaces they do not use. Split fat interfaces into smaller, cohesive ones.
* **Dependency Inversion Principle (DIP):** Depend on abstractions, not concretions. Implement this using Dependency Injection (Spring/Hilt).

---

## 2. SAGA Pattern (Distributed Transactions)

Distributed microservices cannot rely on database locks (2-Phase Commit). The SAGA pattern manages consistency across services using a series of local transactions.

### Choreography vs. Orchestration

```
Choreography (Decentralized)
[ Order Service ] ──► (OrderCreated Event) ──► [ Payment Service ]
                                                      │
                                                      ▼
[ Inventory Service ] ◄── (PaymentCompleted Event) ───┘

Orchestration (Centralized Coordinator)
[ Order Service ] ──► [ Order Saga Orchestrator ]
                            │             ▲
                      1. Pay│             │2. PaymentSuccess
                            ▼             │
                    [ Payment Service ] ──┘
```

#### 1. Choreography-Based Saga
* **Mechanics:** Services publish and consume events asynchronously without a central coordinator.
* **Advantage:** Low coupling, simple configuration for small workflows.
* **Disadvantage:** Difficult to trace execution flow as the number of services increases. Risks circular dependencies.

#### 2. Orchestration-Based Saga
* **Mechanics:** A dedicated orchestrator service coordinates the execution flow, invoking API commands on target services and handling failures.
* **Advantage:** Centralized control, clear execution visibility. Prevents circular dependencies.
* **Disadvantage:** Orchestrator can become a single point of failure and holds complex workflow logic.

#### Compensating Transactions
If a step in a SAGA fails (e.g., payment is declined), the SAGA must execute **Compensating Transactions** in reverse order (e.g., release reserved inventory, cancel pending order) to return the system to a consistent state.

---

## 3. Transactional Outbox & Inbox Patterns

These patterns ensure reliable asynchronous communication across microservices.

### Outbox Pattern (Reliable Publishing)
* **Goal:** Guarantees that database updates and message publishing succeed or fail together.
* **Mechanics:** The application service writes the business record (e.g., `orders`) and an event record to an `outbox` table in the same database within a single local transaction. A separate Change Data Capture (CDC) worker (such as Debezium) reads the outbox table and publishes events to the message broker.

### Inbox Pattern (Reliable & Idempotent Consumption)
* **Goal:** Guarantees that a message is processed exactly-once by the consumer.
* **Mechanics:** The consumer receives an event, writes the event ID to an `inbox` table, and processes the business logic in a single database transaction. If a duplicate message arrives, the `inbox` table blocks it via a unique primary key constraint (`ON CONFLICT DO NOTHING`), preventing duplicate mutations.

---

## 4. Domain-Driven Design (DDD)

DDD structures software around business domains.

### Key Tactical Patterns
* **Bounded Context:** A logical boundary within which a domain model applies. The definition of a `User` in the Identity context differs from a `User` (Customer) in the Billing context.
* **Entities:** Objects defined by a unique identity that persists across state changes (e.g. `Order` with a unique ID).
* **Value Objects:** Immutable objects defined solely by their attributes, with no identity (e.g. `Address` containing `Street` and `ZipCode`).
* **Aggregates:** A cluster of associated objects (Entities and Value Objects) treated as a single unit for data changes. The **Aggregate Root** controls all access to child elements.
* **Repositories:** Interface abstractions for data access, decoupling domain logic from database infrastructure.
* **Domain Events:** Asynchronous notifications representing business occurrences within the domain (e.g., `OrderPlacedEvent`).

---

## 5. Clean Architecture

Clean Architecture structures code in concentric layers, with dependencies pointing inward.

### Layer Hierarchy

```
   ┌────────────────────────────────────────────────────────┐
   │ [ Frameworks & Drivers ] (DB, UI, HTTP Gateway, K8s)   │
   │   ┌────────────────────────────────────────────────┐   │
   │   │ [ Interface Adapters ] (Controllers, Presenters)│   │
   │   │   ┌────────────────────────────────────────┐   │   │
   │   │   │ [ Use Cases ] (Business Rules, Actions)│   │   │
   │   │   │   ┌────────────────────────────────┐   │   │   │
   │   │   │   │ [ Entities ] (Domain Models)   │   │   │   │
   │   │   │   └────────────────────────────────┘   │   │   │
   │   │   └────────────────────────────────────────┘   │   │
   │   └────────────────────────────────────────────────┘   │
   └────────────────────────────────────────────────────────┘
```

* **The Dependency Rule:** Source code dependencies must point inward. Outer layers (Frameworks, Databases) can import from inner layers (Use Cases, Entities), but inner layers must have zero dependencies on outer layers (no SQL or HTTP imports inside the core Domain/Entities layer).
* **Benefits:** Decouples core business rules from databases, frameworks, or external UI tools, simplifying testing and future refactoring.

---

## 6. Test-Driven Development (TDD)

TDD is a software development process based on short feedback loops:

### The Red-Green-Refactor Cycle
1. **Red:** Write a failing unit test for a new feature.
2. **Green:** Write the minimal code required to pass the test.
3. **Refactor:** Clean up code duplication, improve naming, and optimize structures while keeping tests green.

### Mocking vs. Stubbing
* **Stub:** Provides canned answers to calls made during the test.
* **Mock:** Used to verify interactions and verify if specific methods were invoked.

---

### Questions & Answers: Design Patterns & Architecture

#### Q1: Give a concrete example of a violation of Liskov Substitution Principle (LSP) and how to refactor it.
**Answer:**
```java
// Violation of LSP
public class Bird {
    public void fly() { /* flying implementation */ }
}
public class Ostrich extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Ostriches cannot fly!");
    }
}
```
> "This violates LSP because a client class calling `bird.fly()` will crash if passed an instance of `Ostrich`.
> **Refactoring:** Extract behaviors into interfaces to segregate capabilities."
```java
public class Bird {
    // Shared bird attributes
}
public interface Flyable {
    void fly();
}
public class Eagle extends Bird implements Flyable {
    @Override
    public void fly() { /* flying implementation */ }
}
public class Ostrich extends Bird {
    // Ostriches do not implement Flyable
}
```

#### Q2: Contrast Orchestration vs. Choreography SAGA. How do you handle rolling back a transaction if the third step in a four-step saga fails?
**Answer:**
> "1. **Orchestration SAGA:** A central coordinator calls the APIs of the target services. If step 3 (Payment) fails, the orchestrator catches the error and issues compensating API commands to step 2 (Inventory Service - release reserved items) and step 1 (Order Service - mark order as cancelled).
> 2. **Choreography SAGA:** Services react to events asynchronously. If Step 3 fails, the Payment Service publishes a `PaymentFailed` event. The Inventory and Order Services consume this event and trigger their respective compensation logic.
> *SAGA Rollback:* Rollbacks in SAGAs are **semantic rollbacks** rather than database rollbacks. The database does not run a roll back command; instead, compensating transactions write new state records (e.g. inserting a cancellation status) to revert the system state."

---
Design Patterns & Architecture Study Guide
