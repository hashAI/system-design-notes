# Logical clocks — Lamport and vector clocks (practical guide)

In distributed systems, nodes don't share a single clock. **Logical clocks** order events without relying on physical time. They're used for distributed ordering, causality, and conflict detection.

---

## 0) The default mental model (when to use which)

- **Lamport clocks**: Order events across nodes with a single counter. "Happened-before" for causal ordering.
- **Vector clocks**: Detect concurrent (causally independent) events. Used for conflict detection in distributed systems.

In interviews: "We use Lamport timestamps to order events in the distributed log. For conflict detection in multi-leader replication, we use vector clocks."

---

## 1) The problem: why physical time fails

- Clocks drift (NTP helps but isn't perfect).
- Different regions, different skew.
- You can't rely on "event A at 10:00:00.001" vs "event B at 10:00:00.002" to determine order across machines.

**Logical clocks** order events based on **causality** (happened-before), not wall-clock time.

---

## 2) Happened-before (causality)

Event A **happened-before** event B if:

- A and B are on the same process and A occurred first, or
- A is "send" and B is "receive" of the same message, or
- There's a chain of such relationships.

If neither A happened-before B nor B happened-before A, they are **concurrent** (causally independent).

---

## 3) Lamport timestamps

### How it works

- Each node has a counter, initially 0.
- **On local event**: increment counter, attach to event.
- **On send**: increment, attach timestamp to message.
- **On receive**: set counter = max(local counter, received timestamp) + 1.

### Property

- If A happened-before B, then Lamport(A) < Lamport(B).
- **Reverse is false**: Lamport(A) < Lamport(B) does NOT imply A happened-before B. (Could be concurrent.)

### Use cases

- Order events in a distributed log (Kafka, event sourcing).
- Distributed locking (tie-break by Lamport timestamp).
- Replication ordering.

### Limitation

- Cannot detect concurrency. Two events with timestamps 5 and 7 might be concurrent — you can't tell.

---

## 4) Vector clocks

### How it works

- Each node has a vector of length N (one entry per node).
- Node i's vector: [c1, c2, ..., cN] where cj = number of events from node j that i knows about.
- **On local event at node i**: increment V[i].
- **On send**: attach vector to message.
- **On receive**: merge vectors: V[j] = max(local[j], received[j]) for all j; then increment V[self].

### Properties

- **V(A) < V(B)** (component-wise, at least one strict): A happened-before B.
- **V(A) and V(B) incomparable** (neither < the other): A and B are concurrent.

### Use cases

- **Conflict detection**: In multi-leader replication, if two writes have incomparable vector clocks, they're concurrent → conflict.
- **Dynamo-style systems**: Vector clocks attached to versions; concurrent versions require conflict resolution.
- **Causal consistency**: Ensure reads respect causality.

### Limitation

- Vector size = number of nodes. Can grow; pruning strategies exist (e.g., dotted version vectors).

---

## 5) Lamport vs vector clocks

| Aspect | Lamport | Vector |
|--------|---------|--------|
| **Size** | Single number | N numbers (N = nodes) |
| **Detects concurrency?** | No | Yes |
| **Ordering** | Partial (happened-before) | Same + concurrency |
| **Use case** | Distributed ordering | Conflict detection |
| **Overhead** | Low | Grows with cluster size |

---

## 6) Practical applications

### Event sourcing / Kafka

- Lamport timestamps can order events across producers.
- Or use hybrid logical clocks (HLC) for physical + logical ordering.

### Multi-leader replication (Dynamo, Cassandra)

- Vector clocks (or variants) on each version.
- Concurrent writes → multiple versions → client or server resolves.

### Distributed databases

- Version vectors for conflict detection.
- Last-write-wins using timestamps when no concurrency.

---

## 7) Hybrid Logical Clocks (HLC) — bonus

- Combines physical time and logical counter.
- Keeps logical clock close to physical time (bounded drift).
- Used in CockroachDB, MongoDB for distributed timestamps.
- Good when you want "real-time-ish" ordering with causality.

---

## 8) Common interview questions (and answers)

**Q: What's the difference between Lamport and vector clocks?**  
A: Lamport gives a single number; if A happened-before B, Lamport(A) < Lamport(B). It can't detect concurrency. Vector clocks have a per-node counter; if two vectors are incomparable, the events are concurrent. Used for conflict detection.

**Q: When would you use vector clocks?**  
A: In multi-leader replication when we need to detect concurrent writes. Two writes with incomparable vector clocks conflict; we need to resolve (merge, last-write-wins, or present both to user).

**Q: Why not use physical timestamps?**  
A: Clocks drift and can be wrong. Two events might get the same timestamp or wrong order. Logical clocks order by causality, which is what we need for correctness.

---

## 9) Interview-ready summary

"Lamport clocks order events by causality with a single counter; they can't detect concurrency. Vector clocks add per-node counters to detect concurrent events for conflict resolution. We use Lamport for distributed log ordering; vector clocks for multi-leader conflict detection."
