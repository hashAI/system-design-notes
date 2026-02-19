# Design a Distributed Cache — full system design

A distributed cache is a shared in-memory system used to reduce latency and protect databases.

This design covers:

- cache cluster architecture (client-side hashing vs proxy)
- replication vs sharding
- hot keys, stampedes, and tail latency
- operational concerns (rehash, failover)

---

## 1) Requirements

### Functional requirements

- Basic operations:
  - `GET(key)`
  - `SET(key, value, ttl)`
  - `DELETE(key)`
- Low latency reads (single-digit ms)
- Shared cache for many app instances/services
- Supports TTLs and eviction

### Non-functional requirements

- High availability (cache outages should degrade gracefully)
- Predictable tail latency (p99 matters)
- Horizontal scalability (add nodes)
- Operational simplicity (membership changes, rebalancing)

### Out of scope (unless asked)

- Strong consistency (most caches are best-effort)
- Complex query language (caches are key-based)

---

## 2) Back-of-the-envelope scale

Assume:

- 200k QPS reads at peak
- average value size 1 KB

Bandwidth:

- ~200 MB/s read bandwidth

Key implications:

- network and p99 latencies matter
- consistent distribution across nodes is critical

---

## 3) Basic architecture: sharded cache cluster

Most distributed caches are **sharded**:

- each node holds a subset of keys

You need a routing strategy to map key → node.

---

## 4) Routing strategy options

### Option A: Client-side consistent hashing (common)

Clients compute:

- `node = hash(key) on a ring`

Pros:

- no extra hop (lower latency)
- high throughput

Cons:

- clients must know membership
- membership changes require rebalancing logic in clients

Key improvement:

- use **virtual nodes** to smooth distribution

### Option B: Proxy-based routing

Clients talk to proxies; proxies route to cache nodes.

Pros:

- simpler clients
- centralized policy

Cons:

- extra hop (latency)
- proxy can become bottleneck (must scale)

Interview-friendly:

- “Client-side hashing for performance; proxy if client simplicity is needed.”

---

## 5) Replication (optional) and durability

Most caches are not the source of truth, so durability isn’t the goal.

Replication helps availability for hot keys:

- store two copies on different nodes

Tradeoffs:

- more memory usage
- more complex writes/invalidation

Many systems prefer:

- no replication in cache (simple)
- rely on DB as truth
- degrade gracefully on cache misses

If a cache is used for sessions or rate limiting, you may want replication/failover.

---

## 6) Hot keys (the #1 distributed cache problem)

Hot key = one key gets huge traffic.

Symptoms:

- one node becomes overloaded
- p99 latency spikes

Mitigations:

- replicate hot key to multiple nodes (read spreading)
- cache hot key in-process in app servers (local LRU)
- shard the value (for counters: sharded counters)
- use CDN/edge caching when applicable

---

## 7) Cache stampedes (dogpile)

When a key expires, many clients miss and hit the DB.

Mitigations:

- TTL jitter
- single-flight refresh (only one client rebuilds)
- soft TTL + background refresh
- request coalescing at proxy (if proxy-based)

---

## 8) Membership changes and rebalancing

When you add/remove nodes, keys move.

Consistent hashing reduces movement, but some movement still happens.

Operational approaches:

- add nodes gradually (virtual nodes help)
- throttle rewarming traffic to protect DB
- pre-warm hot keys where possible

---

## 9) Eviction and memory management

Caches evict when full using policies:

- LRU/LFU

Operational details:

- monitor eviction rate
- set maximum item size (avoid huge values)
- compress values if needed (trade CPU vs memory)

---

## 10) Multi-region caches

If you have multiple regions:

- you usually run a cache per region (locality)
- don’t try to keep caches strongly consistent across regions

Approach:

- each region caches locally
- DB replication handles truth

---

## 11) Failure modes

### Cache node failure

- keys on that node are lost
- clients remap keys to other nodes
- DB sees increased load

Mitigation:

- add circuit breakers and rate limits to protect DB
- consider “stale cache” fallback for non-critical reads

### Full cache outage

- all traffic hits DB

Mitigation:

- load shedding
- degrade features
- serve stale from CDN if possible

---

## 12) Observability

Metrics:

- hit rate and miss rate
- p95/p99 latency
- eviction rate
- memory usage
- hot keys (top N)
- error rate/timeouts

These metrics help you catch issues before the DB melts.

---

## 13) Interview-ready summary (30 seconds)

“We use a sharded distributed cache to reduce DB load and latency. Keys are routed via client-side consistent hashing with virtual nodes (or proxies if we want simpler clients). We support TTLs and eviction (LRU/LFU) and design for hot keys and stampedes using TTL jitter and single-flight refresh. Cache failures degrade to DB, so we protect downstream with circuit breakers and load shedding. We monitor hit rate, p99 latency, evictions, and hot keys.”

