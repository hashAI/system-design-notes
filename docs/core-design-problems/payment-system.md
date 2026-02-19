# Design a Payment System (high-level flow) — full system design

Payment systems are about **correctness first**. You can’t “eventually” charge someone correctly.

In interviews, you’re not expected to implement a payment processor, but you *are* expected to show:

- idempotency (no double charges)
- a clear state machine
- a ledger model
- handling timeouts, webhooks, and reconciliation

---

## 1) Requirements

### Functional requirements

- Create a payment intent (user wants to pay)
- Authorize payment (card authorization)
- Capture payment (complete the charge)
- Refund payment (full/partial)
- Handle asynchronous provider callbacks (webhooks)
- Provide payment status to clients

### Non-functional requirements

- Strong correctness and auditability
- Idempotency for all writes (client and internal retries)
- High availability, but correctness is prioritized over availability
- Security and compliance (PCI considerations)

### Out of scope (unless asked)

- Full PCI implementation details (tokenization, secure card vault)
- Fraud/ML scoring engines (mention as integration)
- Multi-currency FX engines

---

## 2) Key concepts (what you must say)

### A) Idempotency keys

Every client-initiated mutation must be idempotent:

- create payment intent
- authorize
- capture
- refund

Why:

- network timeouts happen
- clients retry
- you must not double charge

### B) State machine

Payments follow a lifecycle with legal transitions.

Example states:

- `created` → `authorized` → `captured` → `settled`
- plus failure/cancel paths:
  - `failed`, `cancelled`, `refunded`, `chargeback`

### C) Ledger (double-entry)

A ledger records money movement as append-only entries.

Why:

- auditability
- easy reconciliation
- avoids “updating balances in place” errors

Simple model:

- every movement is two entries:
  - debit one account
  - credit another account

---

## 3) High-level architecture

### Components

- **Payments API**: client-facing endpoints (create/confirm/status/refund)
- **Payments DB**: stores intents, charges, refunds, state transitions
- **Ledger service/store**: append-only ledger entries (can be in same DB initially)
- **PSP adapter**: integrates with provider (Stripe/Adyen/etc.)
- **Webhook handler**: verifies and processes provider callbacks
- **Event bus**: publish payment events to other systems (orders, notifications)
- **Reconciliation job**: compares internal state with PSP statements

### Principle: separate “request acceptance” from “final settlement”

Payments are naturally asynchronous:

- user requests payment
- provider may take time
- final settlement can happen later

Your system should handle “pending” states gracefully.

---

## 4) APIs (example)

### Create payment intent

`POST /v1/payment_intents`

Request:

- `idempotency_key`
- `amount`, `currency`
- `customer_id`
- `payment_method_token` (tokenized card, not raw PAN)
- `order_id` (optional but common)

Response:

- `payment_intent_id`
- `status`

### Confirm/authorize

`POST /v1/payment_intents/{id}:confirm`

Response:

- `status = authorized | requires_action | failed`

### Capture

`POST /v1/payment_intents/{id}:capture`

### Refund

`POST /v1/charges/{charge_id}:refund`

Each should accept an idempotency key.

---

## 5) Data model (simplified)

### payment_intents

- `payment_intent_id` (PK)
- `idempotency_key` (unique)
- `order_id`
- `amount`, `currency`
- `customer_id`
- `status`
- `created_at`, `updated_at`

### charges

- `charge_id` (PK)
- `payment_intent_id`
- `provider` (Stripe/Adyen)
- `provider_charge_id`
- `status` (authorized/captured/failed)
- `authorized_at`, `captured_at`

### refunds

- `refund_id` (PK)
- `charge_id`
- `provider_refund_id`
- `amount`
- `status`

### ledger_entries (append-only)

- `entry_id` (PK)
- `transaction_id` (groups entries)
- `account_id`
- `direction` (debit/credit)
- `amount`, `currency`
- `created_at`
- `reference_type/id` (intent/charge/refund)

Senior signal:

- “We keep ledger entries append-only; balances are derived, not updated in place.”

---

## 6) The main flow: authorize + capture (with real failure handling)

### Step 1: Create intent (idempotent)

- Insert `payment_intent` with unique `idempotency_key`.
- If key already exists, return existing intent.

### Step 2: Confirm intent (call PSP)

- Create a `charge` record in `pending` state.
- Call provider to authorize.

Possible outcomes:

- success: mark charge authorized, intent authorized
- requires_action: return to client (3DS flow), remain pending
- failure: mark failed

### Step 3: Capture (idempotent)

- Call provider capture
- mark captured
- write ledger entries for the capture

### The timeout problem (classic)

If you call PSP and your request times out:

- PSP may have succeeded
- or may not have

Correct approach:

- do **not** blindly retry without idempotency
- query PSP by idempotency key or provider charge id
- reconcile state based on PSP truth

---

## 7) Webhooks (untrusted but essential)

PSPs send webhooks like:

- `payment_succeeded`
- `payment_failed`
- `charge_dispute`

Rules:

- **verify signature**
- treat as **at-least-once** (dedupe by webhook event id)
- webhooks can be out of order

Implementation:

- store `webhook_event_id` processed set
- update state machine only via valid transitions

---

## 8) Reconciliation (how you ensure correctness long-term)

Even with idempotency and webhooks, discrepancies happen:

- bugs
- provider delays
- partial outages

Reconciliation job:

- pulls daily statement from PSP
- compares with internal ledger/charges
- flags mismatches
- performs corrective actions (manual or automated)

Senior signal:

- “Reconciliation is non-negotiable for payments.”

---

## 9) Consistency, availability, and CAP choices

Payments typically choose CP-style behavior:

- if DB or ledger is unavailable, fail the request rather than risk double charge or wrong state

But you can still be user-friendly:

- return “pending” and resolve via webhook/reconciliation

---

## 10) Security and compliance (high level)

- Never store raw card numbers unless you are fully PCI compliant
- Use tokenization (PSP gives a payment method token)
- Encrypt sensitive data at rest
- Strict access controls and auditing
- Separate duties: only certain services can access payment secrets

Fraud:

- integrate risk checks before capture
- rate limit attempts

---

## 11) Observability

Metrics:

- auth success rate, capture success rate
- time-to-authorize, time-to-capture
- webhook lag
- reconciliation mismatch count

Logs/traces:

- correlation IDs: intent_id, charge_id, provider_charge_id

Alerts:

- provider error rate spikes
- webhook failures

---

## 12) Interview-ready summary (30 seconds)

“We model payments as a state machine with idempotent APIs for intent creation, authorization, capture, and refunds. We integrate with a PSP via an adapter and handle timeouts by using provider idempotency and reconciliation instead of blind retries. We store an append-only double-entry ledger for auditability and correctness, and process provider webhooks as untrusted, at-least-once events with signature verification and dedupe. A reconciliation job compares PSP statements to internal records to catch and fix discrepancies.”

