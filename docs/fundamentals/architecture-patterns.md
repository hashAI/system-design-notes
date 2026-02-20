# Architecture patterns — CQRS, Event Sourcing, Saga, Microservices

Senior engineers and architects must know when to apply these patterns and their tradeoffs. This guide covers CQRS, Event Sourcing, Saga, and Microservices vs Monolith.

---

## 0) The default mental model (when to use which)

- **CQRS**: Separate read and write models when they have different shapes, scale, or consistency needs.
- **Event Sourcing**: Store changes as events when you need full history, audit, or replay.
- **Saga**: Coordinate multi-step workflows across services without distributed transactions.
- **Microservices vs Monolith**: Start with monolith; split when team size, deployment, or scale demands it.

---

## 1) CQRS (Command Query Responsibility Segregation)

### What it is

Separate **read** and **write** models. Commands (writes) and queries (reads) use different data structures and possibly different stores.

### Why use it

- **Different shapes**: Write model is normalized; read model is denormalized for fast queries (e.g., feed view).
- **Different scale**: Writes are low; reads are 100×. Scale read model independently.
- **Different consistency**: Writes need strong consistency; reads can be eventually consistent.
- **Different stores**: Write to SQL; read from Elasticsearch or a cache layer.

### How it works

- **Command side**: Validates, applies business rules, writes to write store. Publishes events.
- **Query side**: Consumes events (or polls), updates read model(s). Serves queries from optimized read store.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Optimize read and write independently | Eventual consistency on reads |
| Scale reads without touching writes | More moving parts |
| Flexible read models (multiple views) | Complexity in sync and failure handling |

### When to use

- Feed/timeline (write = post; read = merged feed from many sources).
- Dashboard with complex aggregations.
- When read and write patterns diverge significantly.

### What to say in interviews

"We use CQRS for the feed: writes go to a normalized store and publish events; a separate process builds denormalized read models optimized for feed queries. Reads are eventually consistent, which we accept for this use case."

---

## 2) Event Sourcing

### What it is

Store **all changes as an immutable log of events**, not just the current state. Current state is derived by replaying events.

### Why use it

- **Full history**: Audit trail, time travel, debugging.
- **Replay**: Rebuild state from events; add new read models by replaying.
- **Event-driven**: Events naturally feed CQRS, analytics, notifications.

### How it works

- Instead of `UPDATE balance = 100`, store `Deposited(50)`, `Withdrew(30)`.
- To get current balance: replay events for that account.
- Snapshots can speed up replay (periodic saved state + events after).

### Tradeoffs

| Pro | Con |
|-----|-----|
| Complete audit trail | Event schema evolution is hard |
| Replay for new features | Query "current state" requires replay or projection |
| Natural fit for event-driven | Storage can grow (compaction helps) |

### When to use

- Financial systems (audit requirement).
- Collaboration tools (replay for conflict resolution).
- When you need to reason about "what happened" not just "what is."

### What to say in interviews

"We use event sourcing for the payment ledger: every debit and credit is an event. We can replay to get balance, audit, or build new projections. We use snapshots to avoid replaying from the beginning."

---

## 3) Saga pattern (distributed transactions without 2PC)

### What it is

A **saga** is a sequence of local transactions, each with a **compensating action** if a later step fails.

### Why use it

- Distributed transactions (2PC) are complex, block, and don't scale.
- Sagas avoid 2PC by accepting "eventual consistency" and handling failures with compensation.

### How it works

**Choreography**: Each service listens for events and acts; publishes its own events. No central coordinator.

**Orchestration**: A central orchestrator calls each service, tracks state, and triggers compensations on failure.

Example (order → payment → inventory → shipping):

- Step 1: Reserve inventory. Compensate: release reservation.
- Step 2: Charge payment. Compensate: refund.
- Step 3: Create shipment. Compensate: cancel shipment.

If step 3 fails, run compensations for 2 and 1 in reverse order.

### Tradeoffs

| Pro | Con |
|-----|-----|
| No 2PC, scales better | Compensation logic can be complex |
| Works across heterogeneous systems | Temporary inconsistencies |
| Each step is a local transaction | Must design for partial failure |

### When to use

- Multi-step workflows across services (checkout, onboarding).
- When you can't use a single DB transaction.

### What to say in interviews

"For checkout across Order, Payment, and Inventory services, we use a saga. Each step has a compensating action. If we charge the card but inventory fails, we refund. We make compensations idempotent and track saga state for recovery."

---

## 4) Microservices vs Monolith

### Monolith

- Single codebase, single deployment unit.
- Shared database (or a few DBs).
- Simple to develop, deploy, and debug.

### Microservices

- Many small services, each owns a domain.
- Deploy independently.
- Communicate via APIs or events.

### When to start with monolith

- Early-stage product, small team.
- Unclear boundaries.
- Need to move fast and iterate.

### When to split into microservices

- **Team scale**: Multiple teams need to deploy independently.
- **Different scaling needs**: One component needs 10× the scale of others.
- **Different tech stacks**: ML pipeline vs CRUD app.
- **Fault isolation**: One component's failure shouldn't take down everything.
- **Organizational**: Conway's Law — structure mirrors organization.

### Tradeoffs

| Monolith | Microservices |
|----------|---------------|
| Simple ops, single deploy | Complex ops, many deploys |
| Easy debugging, local calls | Harder debugging, network calls |
| Shared DB, strong consistency | Distributed data, eventual consistency |
| Coupled release | Independent release |
| Single point of failure | Fault isolation |

### What to say in interviews

"We start with a monolith for speed. We'll split when we have multiple teams, different scaling requirements, or need fault isolation. We design with clear domain boundaries so extraction is easier later. We avoid splitting too early — distributed systems are hard."

---

## 5) How these patterns combine

- **CQRS + Event Sourcing**: Events are the write model; read models are projections. Common in event-sourced systems.
- **Saga + Event Sourcing**: Saga steps emit events; compensation can be event-driven.
- **Microservices + Saga**: Sagas coordinate across microservices.
- **Microservices + CQRS**: Each service may use CQRS internally; services communicate via events.

---

## 6) Common interview questions (and answers)

**Q: When would you use CQRS?**  
A: When read and write models differ (e.g., feed: writes are posts, reads are merged timelines), or when we need to scale reads independently with different consistency.

**Q: What's the downside of event sourcing?**  
A: Schema evolution of events, querying current state requires replay or projections, and storage growth. We use snapshots and compaction to manage.

**Q: How do sagas handle failures?**  
A: Each step has a compensating action. On failure, we run compensations in reverse order. Compensations must be idempotent.

**Q: Monolith vs microservices — when to split?**  
A: Split when we have multiple teams, different scaling needs, or need fault isolation. Don't split too early; distributed systems add complexity.

---

## 7) Interview-ready summary

"CQRS separates read and write models for independent scaling and optimization. Event sourcing stores changes as events for audit and replay. Sagas coordinate multi-step workflows across services with compensating actions. We prefer monolith early and split to microservices when team size, scale, or fault isolation demands it."
