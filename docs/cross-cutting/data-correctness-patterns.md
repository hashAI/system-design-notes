# Data correctness patterns — how to avoid subtle, expensive bugs

Correctness is what separates “it works in a demo” from “it works in production.”

This page covers patterns you should weave into designs that involve:

- retries
- distributed systems
- async processing
- money/inventory/account state

---

## 1) Idempotency (the #1 correctness tool)

### The problem

Requests get retried:

- client retries after timeout
- gateway retries
- worker retries queue messages

If the operation is not idempotent, retries create duplicates:

- double charge
- duplicate order
- duplicate email

### The pattern

Require an idempotency key for any write that can be retried.

Example:

- `POST /orders` with header `Idempotency-Key: <uuid>`

Server behavior:

- first request:
  - execute the operation
  - store result keyed by `(client_id, idempotency_key)`
- retry request:
  - return the same result without repeating side effects

Implementation options:

- store idempotency records in DB with unique constraint
- store in Redis with TTL for short-lived operations (less durable)

Senior signal:

- define the scope (per user? per API key?) and retention (how long keys are valid).

---

## 2) Deduplication (at-least-once is normal)

Queues and distributed delivery are usually at-least-once.

That means consumers must be safe:

- process the same event twice without corrupting state

Patterns:

- store “processed event IDs” with TTL
- use unique constraints (e.g., `(order_id, event_type)` unique)
- ensure updates are conditional (compare-and-swap / version checks)

---

## 3) Exactly-once: why it’s rare

End-to-end exactly-once requires coordinating:

- message delivery
- side effects (DB writes, external calls)

Most systems choose:

- at-least-once delivery
- idempotency + dedupe

This is usually the best reliability/correctness tradeoff.

---

## 4) Atomicity and transactions (keep invariants in one place)

If you need invariants like:

- inventory never goes below 0
- balances never go negative

Try to keep the critical update inside:

- one database transaction, or
- one strongly consistent service boundary

When invariants span services, you need sagas/workflows.

---

## 5) Sagas and compensating actions (distributed transactions)

### The problem

Checkout example:

1. reserve inventory
2. charge payment
3. confirm order

If step 2 succeeds and step 3 fails, you must recover.

### The saga pattern

Instead of a single distributed transaction, you:

- execute steps
- if a later step fails, run compensations to undo earlier steps

Examples:

- if payment fails → release inventory reservation
- if shipment creation fails → cancel order or retry shipment

Important:

- compensations are not always perfect “undo” (real world has side effects)

Senior signal:

- define an order state machine and make transitions idempotent.

---

## 6) The outbox pattern (reliable event publishing)

### The problem

You want to:

- write to DB
- publish an event to Kafka

If you do:

1. write DB
2. publish event

and crash between them, you can miss events.

If you do:

1. publish event
2. write DB

and crash, consumers see an event for data that doesn’t exist.

### The outbox pattern

In the same DB transaction as your state change, write an “outbox row”:

- `outbox(event_id, type, payload, created_at, published_at NULL)`

Then a background publisher:

- reads unpublished outbox rows
- publishes to Kafka
- marks published

Benefits:

- DB is source of truth
- event publishing becomes reliable and replayable

---

## 7) Inbox pattern (consumer-side dedupe)

For consumers, keep an “inbox” table of processed events:

- `inbox(event_id PK, processed_at)`

Consumer flow:

- if event_id already in inbox → skip
- else process and insert inbox record

This is common for:

- payment webhooks
- order events

---

## 8) Versioning and optimistic concurrency control (OCC)

When multiple writers update the same record, you can lose updates.

Pattern:

- add a `version` field
- update with condition:
  - “update where version = old_version”
- if 0 rows updated → someone else won; retry or surface conflict

Useful for:

- inventory counts
- profile updates
- document edits

---

## 9) Time and ordering (don’t trust clocks blindly)

Clocks can drift, and events can arrive out of order.

Patterns:

- use monotonic sequence numbers per entity (conversation seq)
- use vector/causal metadata only if needed (advanced)
- when using timestamps, include a tie-breaker ID

---

## 10) Reconciliation (trust but verify)

Even with all patterns, discrepancies happen.

Reconciliation jobs:

- compare derived views with source of truth
- compare external systems (PSP statements) with internal ledger
- repair and alert on mismatches

This is essential for:

- payments
- inventory
- billing

---

## 11) Practical examples (how you’d say it in a design)

### Payments

- idempotency keys on all payment mutations
- webhook inbox dedupe
- append-only ledger
- reconciliation

### Notifications

- idempotent send requests
- delivery attempts table
- DLQ for poison messages

### Feeds/search indexes

- outbox pattern from source DB
- rebuildable derived views

---

## 12) Interview-ready summary

“We assume retries and at-least-once delivery, so we build idempotency into write APIs and dedupe into consumers. For reliable event publishing we use the outbox pattern so DB writes and event emission are consistent. For cross-service workflows we use a saga with idempotent state transitions and compensating actions. We avoid lost updates with optimistic concurrency control, and we run reconciliation jobs to detect and fix drift between systems.”

