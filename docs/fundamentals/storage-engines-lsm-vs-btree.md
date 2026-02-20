# Storage engines: LSM vs B-Tree — practical guide

Databases and storage systems use different internal structures. The two dominant designs are **B-Tree** (and variants) and **LSM (Log-Structured Merge-tree)**. Understanding the tradeoffs helps you choose the right store and debug performance.

---

## 0) The default mental model (when to use which)

- **B-Tree**: Best when you need predictable read latency, point lookups, and range scans. Common in PostgreSQL, MySQL, SQL Server.
- **LSM**: Best when writes dominate, you can tolerate occasional read amplification, and you want high write throughput. Common in Cassandra, RocksDB, LevelDB, ScyllaDB.

In interviews: "We chose LSM for write-heavy workloads like event ingestion; B-Tree for transactional workloads where read latency matters."

---

## 1) B-Tree — the classic structure

### What it is

- Balanced tree where each node has many children (e.g., hundreds).
- Data is sorted; leaves hold actual key-value pairs.
- Depth is logarithmic (e.g., 3–4 levels for millions of rows).

### How reads work

- Start at root, follow pointers to the correct leaf.
- O(log n) disk seeks for a point lookup.
- Range scans: traverse leaf nodes in order (efficient).

### How writes work

- Find the leaf, update in place (or insert).
- May need to split nodes (and propagate splits up).
- Writes can be **random** (scattered across disk) → slower on spinning disks.

### Pros

- Predictable read latency.
- Efficient range queries and ordering.
- In-place updates (no rewrite of entire structure).
- Mature, well-understood.

### Cons

- Writes cause random I/O (unless using SSDs).
- Write amplification from node splits and rebalancing.
- Fragmentation over time (vacuum, rebuild).

### Used by

PostgreSQL, MySQL, SQL Server, Oracle, MongoDB (WiredTiger default).

---

## 2) LSM (Log-Structured Merge-tree) — write-optimized

### What it is

- Writes go to an in-memory structure (memtable) first.
- When full, memtable is flushed to disk as an **immutable sorted file** (SSTable).
- Multiple SSTables exist; background **compaction** merges them.

### How writes work

- Append-only: write to memtable (in-memory, fast).
- Flush to disk as new SSTable (sequential write — fast).
- No in-place updates → very high write throughput.
- **Write amplification** happens during compaction (reading + rewriting files).

### How reads work

- Check memtable first.
- Then check SSTables (newest to oldest) until key is found.
- May need to read multiple SSTables → **read amplification**.
- Bloom filters reduce disk lookups for non-existent keys.

### Compaction

- Background process merges SSTables.
- Removes duplicates (keeps newest), reclaims space.
- Tradeoff: more compaction = less read amplification, more write amplification.

### Pros

- Very high write throughput (sequential writes).
- Good for write-heavy, append-heavy workloads.
- Compression is easier (immutable files).
- Bloom filters make "key not found" fast.

### Cons

- Read amplification (multiple SSTables to check).
- Write amplification (compaction rewrites data).
- Compaction can cause latency spikes.
- Space amplification (multiple copies before compaction).

### Used by

Cassandra, RocksDB, LevelDB, ScyllaDB, HBase, DynamoDB (under the hood).

---

## 3) Side-by-side comparison

| Aspect | B-Tree | LSM |
|--------|--------|-----|
| **Write pattern** | In-place, random I/O | Append-only, sequential |
| **Write throughput** | Lower (random writes) | Higher (sequential writes) |
| **Read latency** | Predictable, low | Can spike (many SSTables) |
| **Read amplification** | Low | Higher (multiple levels) |
| **Write amplification** | Moderate (splits) | Higher (compaction) |
| **Space amplification** | Lower | Higher (before compaction) |
| **Range scans** | Excellent | Good (but may touch many files) |
| **Compaction** | N/A (vacuum instead) | Core to design |

---

## 4) Write amplification (both have it)

**Write amplification** = bytes written to disk / bytes in logical write.

- **B-Tree**: Node splits, rebalancing, WAL (write-ahead log).
- **LSM**: Compaction rewrites data multiple times as it moves down levels.

High write amplification → more disk wear (SSDs), more I/O, slower writes.

---

## 5) Compaction strategies (LSM)

Different LSM systems use different compaction:

- **Leveled**: Each level is 10× larger; merge into next level. Better read performance, more compaction.
- **Tiered**: Merge SSTables of similar size. Less compaction, more read amplification.
- **Hybrid**: LevelDB uses leveled; Cassandra offers both.

---

## 6) When to choose which

### Choose B-Tree when

- Read latency is critical (user-facing queries).
- Workload is read-heavy or balanced.
- You need strong consistency and transactions.
- Range scans and ordering are important.

### Choose LSM when

- Write throughput is the bottleneck.
- Append-heavy (logs, events, time-series).
- You can tolerate occasional read latency spikes.
- You're on SSDs (random writes less painful, but LSM still wins on throughput).

---

## 7) Common interview questions (and answers)

**Q: Why is LSM better for writes?**  
A: Writes are append-only (memtable + sequential SSTable flush). No random I/O. B-Tree does in-place updates, which cause random disk writes.

**Q: What is compaction and why does LSM need it?**  
A: Compaction merges SSTables, removes duplicates, reclaims space. Without it, reads would check many files and get slower. Compaction trades write amplification for read performance.

**Q: What is write amplification?**  
A: The ratio of actual bytes written to disk vs logical bytes written. Both B-Tree and LSM have it; LSM can have more due to compaction.

**Q: How do you reduce read amplification in LSM?**  
A: Bloom filters (skip SSTables that don't contain the key), more aggressive compaction, or tiered storage (hot data in fewer levels).

---

## 8) Interview-ready summary

"B-Tree gives predictable read latency and efficient range scans; writes are in-place and can be slower. LSM is write-optimized: append-only writes, high throughput, but read amplification and compaction. We choose B-Tree for read-heavy transactional workloads, LSM for write-heavy event/log ingestion."
