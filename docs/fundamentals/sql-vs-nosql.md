# SQL vs NoSQL — practical guide (when and why)

People often ask: “Should we use SQL or NoSQL?”

The better question is:

**What data do we have, what questions do we ask of it, and what correctness guarantees do we need?**

This guide uses the same “system design problem” language: start with requirements and access patterns, choose the simplest store that enforces your invariants, then add derived stores (cache/search) for speed.

---

## 0) The default choice (works for most systems)

If you’re unsure, start with:

- **SQL as source of truth** for core entities and correctness constraints.
- Add:
  - **Redis** for caching
  - **Search index** (OpenSearch/Elasticsearch) for full-text search
  - **Event log / wide-column** store for massive append-only event streams

Then evolve only when scale/queries force it.

---

## 1) The simple mental model

### SQL databases (relational)

Think “**tables with rules**.”

- Data is stored in tables (rows/columns).
- You can join tables (combine related data).
- You can enforce strong rules (constraints, transactions).

Good for:

- orders, payments, inventory
- user accounts
- anything where correctness rules matter

Examples:

- Postgres, MySQL

### NoSQL databases (several families)

Think “**data shaped for a specific access pattern**.”

NoSQL is not one thing. It includes:

- **Key-value** stores (fast get/put by key)
- **Document** stores (JSON-like objects)
- **Wide-column** stores (designed for big write throughput)
- **Graph** stores (relationships)

Good for:

- huge scale with simple queries
- flexible schemas
- event/time-series-like access patterns

Examples:

- DynamoDB (key-value)
- MongoDB (document)
- Cassandra (wide-column)

Important: “NoSQL” does not automatically mean “faster” or “better at scale.” It means “different tradeoffs.”

---

## 2) Start with access patterns (the senior way)

Before picking a database, list the top queries:

- “Get user by email”
- “List last 50 orders for user”
- “Update inventory and create order atomically”
- “Search products by text”
- “Store billions of events and query by time range”

Then choose the store that supports those queries with the needed correctness.

---

## 3) When SQL is the best default

SQL is a great default when:

- you need **transactions** (all-or-nothing updates)
- you need **strong consistency** for invariants
- you need **joins** or flexible querying
- your data is structured and relationships matter

### Example: e-commerce checkout

You often need to:

- create an order
- reserve inventory
- record payment intent

These steps have correctness constraints. SQL + transactions are a natural fit (or a workflow on top of it).

### Scaling SQL (simple path)

SQL can scale a lot with:

- good indexes
- caching
- read replicas
- partitioning/sharding (more complex, but possible)

---

## 4) When NoSQL is a better fit

### A) Key-value stores (DynamoDB / Redis-like patterns)

Pick when:

- your queries are mostly “get/put by key”
- you need massive scale and low latency
- you can model data to avoid joins

Great for:

- session data
- rate limiting counters
- user timeline inbox lists (IDs)
- simple metadata lookups

Tradeoff:

- limited ad-hoc querying (you must design around known access patterns)

### B) Document stores (MongoDB)

Pick when:

- your data is naturally a JSON document
- you want flexible schema evolution
- you often fetch “whole objects” together

Great for:

- content documents (posts/articles) with nested fields
- product catalog with varying attributes (but be careful with search)

Tradeoffs:

- embedding can cause large documents if not bounded
- joins are limited (though some exist)
- complex transactions are possible but may not be as natural as SQL

### C) Wide-column stores (Cassandra)

Pick when:

- you have huge write throughput
- you mostly query by partition key + range (time-series style)
- you accept eventual consistency tradeoffs (depending on config)

Great for:

- logs/events
- message history per conversation (if modeled right)

Tradeoff:

- you must **design by query** (schema is optimized for known queries)
- ad-hoc queries are hard

---

## 5) A common real-world pattern: “SQL + derived views”

Many systems use:

- **SQL** as the source of truth for correctness
- plus specialized stores as derived views:
  - **Redis** for caching
  - **Elasticsearch/OpenSearch** for full-text search
  - **Cassandra** or object storage for high-volume event logs

Simple memory trick:

- **SQL = truth**
- **NoSQL/search/cache = speed**

This is not always true, but it’s a safe starting mental model.

---

## 6) How to choose quickly (decision checklist)

Ask these questions:

1. **Do we need multi-row transactions and strong constraints?**  
   - Yes → SQL (usually).
2. **Are the queries simple key lookups at huge scale?**  
   - Yes → key-value NoSQL.
3. **Is the data naturally a document and read/written as a whole?**  
   - Yes → document store.
4. **Is it write-heavy time-series/event-like with predictable queries?**  
   - Yes → wide-column store.
5. **Do we need full-text search and ranking?**  
   - Use a search index as a derived system (don’t use it as the only truth for money).

---

## 6.1) Worked examples (how this shows up in real designs)

### Example A: E-commerce checkout

Needs:

- inventory can’t go negative
- payments can’t double-charge
- orders must be auditable

Pick:

- **SQL** for orders/inventory/payment intents (transactions + constraints)
- **Redis** for caching product pages and carts
- **Search index** for product search (derived from catalog)

### Example B: Chat message history

Needs:

- fast “fetch last N messages by conversation”
- high write throughput
- ordering per conversation

Pick:

- **wide-column** store (Cassandra-like) or sharded SQL with a schema designed for that query
- **SQL** for conversation metadata and memberships

### Example C: Rate limiting counters

Needs:

- low-latency increments with TTL
- high QPS

Pick:

- **Redis** (atomic `INCR` + TTL or Lua scripts)

### Example D: Logging/telemetry events

Needs:

- extremely high write throughput
- time-based queries, retention, low cost

Pick:

- Kafka + object storage (raw)
- search index for hot window (optional)

---

## 7) Beginner mistakes (and what to say instead)

### Mistake: “NoSQL scales better”

Reality: SQL can scale very far; NoSQL often scales *differently* (at the cost of constraints/joins).

Say:

- “We choose based on access patterns and invariants. Start with SQL for correctness; add specialized stores as needed.”

### Mistake: “We’ll use MongoDB so schema is flexible”

Flexibility can also mean “harder to keep data clean.”

Say:

- “We’ll define a schema contract at the application layer and enforce it with validation.”

### Mistake: ignoring operational complexity

Each extra datastore adds:

- deployment/monitoring
- backup/restore plan
- migrations
- expertise requirement

Senior signal: mention “keep the system simple unless scale forces complexity.”

---

## 8) Interview-ready summary

“SQL is the default for strong correctness, transactions, and flexible queries. NoSQL is a family of databases optimized for certain access patterns—key-value for huge simple lookups, document for JSON-shaped objects, wide-column for write-heavy event/time-series workloads. A common design is SQL as source of truth with caches/search indexes as derived views for speed.”

