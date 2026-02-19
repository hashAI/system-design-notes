# Replication & sharding — simple, deep guide

When a database (or any storage system) hits limits, there are two big tools:

- **Replication**: keep *copies* of the same data.
- **Sharding (partitioning)**: *split* data across multiple machines.

They solve different problems:

- Replication helps **availability** and **read scaling**.
- Sharding helps **write scaling** and **storage scaling**.

---

## 0) The default scaling path (what you’ll do in many designs)

If you’re not sure what to propose, this is a safe progression:

1. **Index and query tuning** (often the biggest win).
2. **Replication** (read replicas) to scale reads and improve availability.
3. **Sharding** when:
   - the dataset no longer fits on a single node, or
   - the write rate is too high for a single leader.

Then call out the two classic gotchas:

- replication causes **lag** (stale reads)
- sharding causes **cross-shard complexity** (queries/transactions/migrations)

---

## 1) Replication (copies of the same data)

### What replication gives you

- **Higher availability**: if one node dies, another copy can serve.
- **More read capacity**: you can read from replicas.
- **Disaster recovery**: copies in another AZ/region reduce risk.

### What replication costs you

- complexity (failover, consistency)
- **replication lag** (replicas can be behind)
- extra storage cost

Key point:

- replication increases availability and read capacity, but introduces lag and failover complexity.

---

## 2) The most common setup: leader–follower

Also called primary–replica.

### How it works

- **Leader (primary)** receives writes.
- **Followers (replicas)** copy changes from the leader.
- Reads can go to leader or replicas depending on needs.

### Why it’s popular

- Easy mental model.
- Good for scaling reads.

### The big tradeoff: replication lag

If you read from a replica right after a write, you may not see your write yet.

This is fine for:

- feeds
- analytics
- “view count”

It’s dangerous for:

- money balances
- inventory
- permissions

### Read-your-writes (common requirement)

If a user just changed something, route their reads to:

- the leader, or
- a replica that has caught up (if you track it).

In interviews, saying “read-your-writes” is a strong signal.

---

## 3) Failover (what happens when the leader dies)

Leader failure is normal. The system must promote a replica.

Typical failover steps:

1. Detect leader is unhealthy (health checks / consensus).
2. Choose a replica to promote.
3. Update routing so writes go to new leader.

Key pitfalls:

- **split brain**: two leaders accept writes (data corruption risk).
- **lost writes**: the old leader had writes not replicated yet.

How systems reduce risk:

- synchronous replication for critical writes (slower)
- quorum/consensus systems (more coordination)
- careful fencing/leases to prevent two leaders

For interviews, keep it simple:

- “We accept small risk of lost last-second writes for non-critical data; for critical data we use stronger replication or transaction logs.”

---

## 4) Multi-leader replication (multiple writers)

Useful when:

- you need writes in multiple regions (low latency).

Hard part:

- **conflicts** (two regions update same record).

You need a conflict resolution strategy:

- last-write-wins (simple, can lose updates)
- merge rules (domain-specific)
- CRDT-like approaches (advanced)

Beginner-friendly summary:

- Multi-leader improves write locality but makes correctness harder.

---

## 5) Quorum replication (Dynamo-style idea)

Some systems talk in terms of `N, R, W`:

- `N`: number of replicas
- `W`: how many replicas must confirm a write
- `R`: how many replicas you read from

If `R + W > N`, reads and writes overlap, which improves consistency.

Practical takeaway:

- You can tune “more consistent” vs “more available/faster.”

---

## 6) Sharding (splitting data)

Sharding means your data is divided across multiple “shards” so no single machine holds everything.

Key point:

- sharding scales writes and storage, but increases operational and query complexity.

### What sharding gives you

- **Write scaling**: writes go to different shards in parallel.
- **Storage scaling**: total data fits across many machines.
- **Isolation**: one shard’s load doesn’t always melt the whole cluster (unless you have a hot shard).

### What sharding costs you

- cross-shard queries are harder
- cross-shard transactions are harder
- rebalancing/resharding is hard
- hot shards can happen

---

## 7) Choosing a shard key (the most important decision)

A shard key decides which shard holds a row.

Good shard keys:

- spread load evenly
- match your access patterns
- keep related data together when needed

Bad shard keys:

- create hotspots
- make common queries fan out across all shards

### Common strategies

#### A) Hash-based sharding

- Compute `hash(key) % num_shards`.
- Great for even distribution.
- Bad for range queries (e.g., time ranges).

Use for:

- user ID lookups
- URL short codes

#### B) Range-based sharding

- Shard by ranges (e.g., `user_id 1–1M` on shard 1).
- Good for range scans.
- Risk: hotspots if new data always lands in newest range (time-based ranges).

Use for:

- time-series data (with care)

#### C) Directory-based sharding

- A lookup service maps key → shard.
- Flexible, but adds another dependency.

Use when:

- you need to move keys between shards without changing key format

---

## 8) Hot shards and hot keys (the classic sharding failure)

Even if you have 100 shards, one shard can melt.

### Why hot shards happen

- one celebrity user
- one product during a flash sale
- time-based shard where “current time” is hottest

### Common mitigations

- split hot shard (more shards, finer partitions)
- isolate the celebrity/high-traffic keys
- replicate hot items (cache and/or multi-read copies)
- “shuffle sharding” for noisy tenants
- use rate limiting/admission control for abuse

Senior signal: always say “we must plan for skew.”

---

## 9) Resharding (what happens when you add shards)

Adding shards means moving data. That’s risky and expensive.

Common approaches:

- **consistent hashing with virtual nodes** (reduces movement when nodes change)
- background rebalancing with throttling
- dual writes/reads during migration (for correctness)

Interview-friendly statement:

- “We’ll pick a shard key early and use consistent hashing + virtual nodes so adding capacity doesn’t require moving most keys.”

---

## 10) Replication + sharding together (how real systems do it)

Most real systems do both:

- Data is split into shards.
- Each shard is replicated.

Example:

- 20 shards × 3 replicas each = 60 nodes.

Reads can go to replicas; writes go to leaders (per shard) depending on architecture.

---

## 11) Interview-ready summary

“Replication keeps copies of data for availability and read scaling, but introduces replication lag and failover complexity. Sharding splits data across nodes for write/storage scaling; the shard key is the critical design choice, and we must plan for hotspots and resharding. Real systems usually combine sharding + replication: each shard is replicated for durability and availability.”

