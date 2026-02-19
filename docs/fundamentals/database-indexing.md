# Database indexing — practical deep dive

Indexes are one of the most important “make it fast” tools in system design.

If caching is like keeping a copy at the front desk, **indexing is like adding a table of contents to a book**:

- Without an index, the database may have to scan many rows (“read the whole book”).
- With an index, it can jump straight to the right place (“use the table of contents”).

---

## 0) The default indexing plan (useful in most designs)

When you’re designing a system and you’re not sure what indexes you need, start here:

- **List the top 3–5 queries** (the queries that happen the most and/or must be fast).
- Add indexes to support:
  - the **filter** (`WHERE ...`)
  - the **sort** (`ORDER BY ...`)
  - the **pagination cursor** (if you paginate)
- Keep the number of indexes small at first (indexes slow writes).
- Validate with `EXPLAIN` once you have a real query.

This is the same “requirements → hot queries → design” mindset you use in the core design problems.

---

## 1) What is an index?

An **index** is an extra data structure that lets the database find rows faster.

You pay for that speed with:

- extra storage
- extra work on writes (because the index must be updated)

So the tradeoff is:

- **Faster reads** vs **slower writes** + **more storage**

---

## 2) The most common index: B-Tree (what it’s good at)

Most relational databases (Postgres, MySQL, etc.) commonly use B-Tree indexes.

B-Tree indexes are great for:

- equality lookups: `WHERE user_id = 123`
- ranges: `WHERE created_at >= ... AND created_at < ...`
- ordering: `ORDER BY created_at DESC`
- prefix matches on composite keys (more on that below)

Simple rule:

- If you filter/sort by a column often, it may need an index.

---

## 3) Why indexes speed things up

Without an index, many queries require scanning large portions of a table.

- Without an index on `email`, a query like `WHERE email = 'a@b.com'` might scan a huge portion of the table.
- With an index on `email`, the DB can find the row location quickly.

This is why “add an index” is often the first scaling step **before** adding replicas or sharding.

---

## 4) The most important index concepts

### A) Selectivity (how unique a column is)

Indexes help most when the column narrows down results a lot.

- High selectivity: `user_id`, `email`, `order_id` → great index candidates.
- Low selectivity: `is_active` (true/false) → often not useful by itself.

But low-selectivity columns can still be useful **as part of a composite index**.

### B) Composite indexes (multi-column indexes)

An index can be on multiple columns, like `(user_id, created_at)`.

Rule: **leftmost prefix**

- Index on `(user_id, created_at)` helps:
  - `WHERE user_id = ...`
  - `WHERE user_id = ... AND created_at > ...`
  - `ORDER BY created_at` *within a user’s rows*
- It generally does **not** help much for:
  - `WHERE created_at > ...` (without user_id)

So order matters: put the most common filter first.

### C) Covering indexes

A **covering index** contains all the columns needed to answer a query.

Example:

- Query: `SELECT created_at, status FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 20`
- If an index on `(user_id, created_at, status)` exists, DB may avoid reading the full table rows.

Result: faster queries with fewer disk reads.

---

## 5) Indexes are not free (write cost and storage)

Every insert/update/delete must update:

- the table data
- every index on that table

So:

- too many indexes can slow writes a lot
- indexes take space
- indexes increase maintenance work (vacuum/compaction, reindexing)

Simple “balance” rule:

- Start with the indexes that match your most important read queries.
- Add indexes only when you can justify them with measured slow queries.

---

## 6) How to decide what to index (access patterns first)

In system design, you should always say:

1. “What are the top queries?”
2. “What filters/sorts do those queries use?”
3. “What indexes support those queries?”

Examples:

- “Fetch latest 50 messages in a conversation” → index on `(conversation_id, message_id)` or `(conversation_id, created_at)`
- “Find user by email” → unique index on `email`
- “List a user’s orders newest first” → `(user_id, created_at DESC)`

---

## 7) The classic trap: OFFSET pagination

Many beginners paginate like:

- `LIMIT 50 OFFSET 50000`

This can become slow because DB must walk through many rows to reach the offset.

Better: **cursor pagination**.

Example:

- “Give me orders older than `(created_at, id)` cursor” using an index that matches that ordering.

This stays fast at large scale.

---

## 8) Primary key vs secondary indexes

- **Primary key index**: built-in index on the primary key (often clustered in some DBs).
- **Secondary index**: additional index on other columns.

Senior signal: say which query uses the primary key vs which needs a secondary index.

---

## 9) How seniors debug slow queries (simple checklist)

If something is slow:

- Look at the query plan (`EXPLAIN`).
- Ask: is it doing a full table scan?
- Ask: is it using the right index?
- Check:
  - missing index
  - wrong composite index order
  - low selectivity
  - too many rows being returned
  - large joins without proper keys

In interviews you don’t need deep SQL, but you should mention:

- “We’ll use `EXPLAIN` to confirm indexes are used.”

---

## 9.1) Worked examples (what you’d say in a design)

### Example A: “List a user’s orders, newest first”

Query:

- `SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 50`

Index:

- `(user_id, created_at DESC)` (or `(user_id, created_at, id)` for stable cursor paging)

Why:

- filters by `user_id`
- sorts by `created_at`
- supports efficient cursor pagination

### Example B: “Find user by email”

Query:

- `SELECT * FROM users WHERE email = ?`

Index:

- unique index on `email`

Why:

- high selectivity
- correctness benefit (enforces uniqueness)

### Example C: “Search by status within a user”

Query:

- `SELECT * FROM orders WHERE user_id = ? AND status = ? ORDER BY created_at DESC`

Index:

- `(user_id, status, created_at DESC)`

Why:

- `status` alone has low selectivity, but combined with `user_id` it can be useful.

---

## 10) Interview-ready summary

“Indexes speed up reads by avoiding full scans, but they cost storage and slow writes. We choose indexes based on the top access patterns—especially filters and sort order. Composite indexes follow the leftmost-prefix rule, and covering indexes can avoid extra table reads. For pagination at scale we prefer cursor pagination over OFFSET.”

