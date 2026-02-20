# Bloom filters and probabilistic structures — practical guide

When you need to answer "does this exist?" or "how many unique items?" at massive scale with bounded memory, **probabilistic data structures** are the tool. They trade exactness for space and speed.

This guide covers Bloom filters, HyperLogLog, and Count-Min Sketch — the three you'll encounter in system design interviews.

---

## 0) The default use cases (when to reach for these)

- **Bloom filter**: "Does this key exist in the set?" — avoid expensive lookups (DB, disk) when the answer is "no."
- **HyperLogLog**: "How many unique items?" — cardinality estimation with ~1.5% error in kilobytes.
- **Count-Min Sketch**: "How often does X appear?" — frequency estimation for heavy hitters, top-K.

All are **probabilistic**: small chance of false positives (Bloom) or bounded error (HLL, CMS). No false negatives for Bloom.

---

## 1) Bloom filter — "might be there" vs "definitely not"

### What it does

A Bloom filter answers: "Is this element in the set?"

- **No** → definitely not in the set (no false negatives).
- **Yes** → might be in the set (small chance of false positive).

### How it works (simplified)

- Bit array of size m.
- k hash functions map each element to k bit positions.
- **Add**: set all k bits to 1.
- **Query**: check if all k bits are 1. If any is 0 → definitely not present. If all 1 → probably present.

### Why it's useful

- **Tiny memory**: 1 billion items with 1% false positive rate ≈ 1.2 GB (vs billions of bytes for storing keys).
- **Fast**: O(k) for add and query; k is small (often 7–10).
- **No deletions** (standard Bloom): clearing bits could affect other elements. Use Counting Bloom Filter for deletions.

### When to use

- **Cache / DB filter**: "Check Bloom before hitting DB. If Bloom says no, skip the lookup."
- **Distributed systems**: "Does this chunk/file exist?" before fetching.
- **Spell check**: "Is this word in the dictionary?" (classic use case).
- **Deduplication**: "Have we seen this event ID?" before processing.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Very small memory | False positives (can't avoid) |
| No false negatives | Can't delete (standard version) |
| Fast add/query | Can't list elements |
| No storage of actual keys | Must tune m, k for desired FP rate |

### What to say in interviews

"We'll use a Bloom filter in front of the cache/DB. On cache miss, check Bloom first. If Bloom says the key doesn't exist, we skip the DB lookup entirely. We accept a small false positive rate (e.g., 1%) to save memory and reduce load."

---

## 2) HyperLogLog — "how many unique?"

### What it does

Estimates the **cardinality** (number of unique elements) of a stream using very little memory.

- ~1.5% standard error.
- Fixed memory (e.g., 12 KB for billions of distinct items).

### How it works (simplified)

- Uses hash of each element.
- Tracks patterns in hash bits (e.g., longest run of leading zeros).
- Combines estimates from multiple "registers" for accuracy.

### When to use

- "How many unique visitors today?" (billions of events, limited memory).
- "How many distinct user IDs in this log?"
- "Cardinality of a set" when you can't store the set.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Constant memory | Approximate only |
| Mergeable (different HLLs can be combined) | Can't list elements |
| Handles massive streams | ~1.5% error |

### What to say in interviews

"For unique visitor count at scale, we use HyperLogLog. We can estimate cardinality in the billions with ~12 KB. It's mergeable, so we can combine per-region HLLs for a global count."

---

## 3) Count-Min Sketch — "how often?"

### What it does

Estimates the **frequency** of elements in a stream. Good for finding "heavy hitters" (most frequent items).

### How it works (simplified)

- 2D array of counters: d rows × w columns.
- d hash functions map each element to one column per row.
- **Add**: increment counters at (row, hash(element)) for each row.
- **Query**: take minimum of the d counters (min reduces impact of collisions).

### When to use

- "Top K most frequent items" (trending topics, hot keys).
- "How many times did user X do Y?"
- Frequency estimation for rate limiting, anomaly detection.

### Tradeoffs

| Pro | Con |
|-----|-----|
| Sublinear space | Overestimates (never underestimates) |
| Fast updates | Approximate |
| Good for heavy hitters | Less accurate for rare items |

### What to say in interviews

"For finding hot keys or trending items, we use Count-Min Sketch. It gives us frequency estimates with bounded error and lets us identify heavy hitters without storing full counts for every key."

---

## 4) Quick comparison

| Structure | Question | Memory | Error |
|-----------|----------|--------|-------|
| **Bloom filter** | Is X in the set? | O(n) bits for n items | False positives only |
| **HyperLogLog** | How many unique? | ~12 KB typical | ~1.5% |
| **Count-Min Sketch** | How often is X? | O(w×d) | Overestimates |

---

## 5) Real-world usage

| System | Use |
|--------|-----|
| **Cassandra, HBase** | Bloom filters to avoid disk lookups for non-existent keys |
| **Redis** | `BF.ADD`, `BF.EXISTS` (RedisBloom module) |
| **BigQuery, Spark** | `APPROX_COUNT_DISTINCT` often uses HyperLogLog |
| **Datadog, monitoring** | Cardinality estimation for metrics |

---

## 6) Common interview questions (and answers)

**Q: When would you use a Bloom filter?**  
A: When we need to avoid expensive lookups (DB, disk, network) for keys that probably don't exist. Example: before checking cache, check Bloom; if Bloom says no, skip cache/DB. We accept rare false positives.

**Q: Can a Bloom filter have false negatives?**  
A: No. If the element was added, all k bits will be 1. So "no" from a Bloom filter is always correct.

**Q: How do you reduce false positives in a Bloom filter?**  
A: Use more bits (larger m) or more hash functions (k). There's a formula: for n elements and desired FP rate p, m ≈ -n ln(p) / (ln 2)².

**Q: What's the difference between HyperLogLog and Bloom filter?**  
A: Bloom answers "is X in the set?"; HyperLogLog answers "how many unique items?" HLL doesn't support membership queries; Bloom doesn't support cardinality.

---

## 7) Interview-ready summary

"Bloom filters answer 'is this in the set?' with no false negatives and a small false positive rate — great for avoiding expensive lookups. HyperLogLog estimates unique count with ~1.5% error in constant memory. Count-Min Sketch estimates frequencies for heavy hitter detection. All trade exactness for space and speed."
