# Design an E-commerce System — full system design

E-commerce systems are a “greatest hits” of system design:

- product catalog + search
- carts and checkout
- orders and payments
- inventory correctness
- shipping and notifications

The key senior theme:

> Strong correctness for money/inventory, eventual consistency for everything else.

---

## 1) Requirements

### Functional requirements

- Browse products (catalog)
- Search products (optional; usually required in real life)
- Add to cart
- Checkout:
  - create order
  - reserve inventory
  - take payment
  - confirm order
- Order history and status tracking

### Nice-to-have

- promotions/coupons
- recommendations
- returns/refunds workflow
- multiple warehouses

### Non-functional requirements

- High availability for browsing
- Correctness for money and inventory (avoid double charge, avoid oversell)
- Low latency for product pages and cart
- Scalability for traffic spikes (sales events)
- Security and fraud prevention

---

## 2) High-level service decomposition (practical)

You can describe these as services or modules:

- **Catalog service**: product data, pricing (base)
- **Search service**: full-text search index (derived view)
- **Cart service**: user carts (ephemeral-ish)
- **Inventory service**: stock counts and reservations (correctness critical)
- **Order service**: order state machine
- **Payment service**: payment intents, captures (correctness critical)
- **Checkout orchestrator**: coordinates multi-step checkout (saga/workflow)
- **Shipping service**: shipping labels, tracking (integration)
- **Notification service**: emails/push

Senior signal:

- “We separate read-heavy browsing from correctness-critical checkout.”

---

## 3) Core data models (simplified)

### Products (catalog)

- `products(product_id, title, description, attributes, created_at, ...)`
- `prices(product_id, currency, price, updated_at)`

Catalog is read-heavy → cache + CDN.

### Cart

- `carts(user_id PK, items[], updated_at)`

Often stored in:

- Redis (fast) + optional DB for durability

### Inventory and reservations

- `inventory(sku_id PK, available_count, reserved_count, version)`
- `reservations(reservation_id, sku_id, user_id, qty, expires_at, status)`

### Orders

- `orders(order_id, user_id, status, total, created_at, ...)`
- `order_items(order_id, sku_id, qty, price_at_purchase)`

Order is a state machine (see below).

---

## 4) Checkout flow (the heart of the design)

Checkout is a multi-step distributed process. You want:

- no double charge
- no oversell
- clear recovery paths on partial failures

### The state machine approach

Order states example:

- `created`
- `inventory_reserved`
- `payment_authorized`
- `confirmed`
- `shipped`
- `delivered`
- `cancelled`

And failure states:

- `payment_failed`
- `reservation_expired`

### A saga/workflow orchestrator

Use a checkout orchestrator that does:

1. Create order (pending)
2. Reserve inventory (with expiration)
3. Authorize payment
4. Confirm order
5. Emit events for shipping/notifications

Compensating actions on failure:

- if payment fails → release inventory reservation
- if reservation fails → fail order without charging

This is the senior “distributed transaction” answer.

---

## 5) Inventory correctness (avoid oversell)

Inventory is the hardest part at scale.

### Option A: Strong reservation in a single inventory service (common)

Reserve inventory with an atomic check-and-decrement:

- if `available_count >= qty`, then decrement available and increment reserved
- create reservation with TTL (expires if checkout not completed)

Implementation:

- SQL transaction with row lock on `sku_id`
- or compare-and-swap with version number

### Option B: Partition by SKU (scaling)

Shard inventory by `sku_id` to scale writes.

### Hot SKU during flash sales

Mitigations:

- strict rate limiting/admission control on checkout
- queue “purchase attempts” (but careful with user experience)
- pre-allocate inventory tokens (advanced)

Interview-friendly:

- “For hot items, we protect inventory service with queues and rate limits, and may use a ‘virtual waiting room’.”

---

## 6) Payments integration (summary)

Use the payment system design:

- payment intents + idempotency keys
- webhook handling
- reconciliation

Order and payment must be linked via stable IDs so retries are safe.

---

## 7) Browsing and search (read-heavy)

Catalog read path:

- CDN caches product images and static assets
- API caches product details (Redis)

Search:

- use a search index (OpenSearch/Elasticsearch) built from catalog events
- treat search index as derived (rebuildable)

---

## 8) Caching strategy

Cache:

- product pages (by product_id, locale)
- category listings
- cart reads

Avoid caching:

- inventory counts as “truth” (can cache with short TTL for display, but checkout must verify)

---

## 9) Event-driven architecture (good for scale)

Emit events:

- `order_created`
- `inventory_reserved`
- `payment_captured`
- `order_confirmed`
- `order_shipped`

Consumers:

- notifications
- shipping integration
- analytics
- search index updates

Use an outbox pattern for reliable event publishing (see data correctness page).

---

## 10) Failure modes (what happens when things break)

### Payment provider slow/down

- order remains pending
- user sees “processing”
- resolve via webhook/retry/reconciliation

### Inventory service overload

- shed load (429)
- wait room for flash sales

### Duplicate requests

- idempotency keys for checkout/payment APIs prevent double charges

### Partial failures

- saga orchestration + compensations

---

## 11) Observability

Metrics:

- checkout success rate
- payment auth/capture success rate
- reservation timeouts/expirations
- inventory contention (lock wait time)
- order state transition latency

Alerts:

- spike in payment failures
- spike in oversell/negative inventory

---

## 12) Interview-ready summary (30 seconds)

“We split the system into read-heavy browsing (catalog/search) and correctness-critical checkout (orders, inventory, payments). Checkout is implemented as a saga/workflow: create order, reserve inventory with TTL, authorize/capture payment with idempotency, then confirm order and emit events. Inventory uses atomic reservations to avoid oversell and is protected with rate limits/queues during flash sales. Search and analytics are derived views built asynchronously. We cache product pages aggressively but always verify inventory and payment correctness on the critical path.”

