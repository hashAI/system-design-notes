# Consensus and leader election — practical guide

When multiple replicas must agree on a single leader or a single value, you need **consensus**. This is how distributed databases, coordination services, and message brokers stay correct under failures.

This guide covers what senior engineers and architects need to know: leader election, Paxos/Raft at a high level, split-brain, and quorum.

---

## 0) The default mental model (use in most designs)

- **Leader-follower** systems need a way to pick exactly one leader when the current one fails.
- **Consensus** ensures all nodes agree on who the leader is (or what value was chosen).
- **Quorum** (majority of nodes) is required so that during a partition, at most one side can make progress.
- **Split-brain** (two leaders) is the failure you must prevent — it corrupts data.

In interviews: "We use Raft (or similar) for leader election. A quorum must agree before a new leader is elected, which prevents split-brain."

---

## 1) Why consensus exists

In a replicated system:

- Writes go to the leader.
- If the leader dies, a new one must be chosen.
- If two nodes both think they're the leader, they can accept conflicting writes → **data corruption**.

Consensus solves: "How do we agree on a single leader (or a single value) when nodes can fail and the network can partition?"

---

## 2) Leader election (the problem)

When the leader fails:

1. **Detect** the failure (timeouts, heartbeats).
2. **Elect** a new leader.
3. **Reconfigure** clients and replicas to use the new leader.

The hard part: ensuring only one leader exists at any time, even during network partitions.

---

## 3) Split-brain (the failure to avoid)

**Split-brain** means two (or more) nodes believe they are the leader and both accept writes.

### How it happens

- Network partition splits the cluster into two groups.
- Each group cannot see the other.
- Without coordination, each group might elect its own leader.
- Both leaders accept writes → conflicting data.

### How to prevent it

- **Quorum**: only a majority can elect a leader. During a partition, at most one side has a majority.
- **Fencing tokens**: each leader gets a monotonically increasing token; older leaders are rejected.
- **Lease-based leadership**: leader holds a time-bound lease; expires if heartbeats stop.

Senior signal: "We prevent split-brain by requiring a quorum for leader election. During a partition, the minority side cannot elect a leader."

---

## 4) Quorum (majority rules)

**Quorum** = majority of nodes. With N nodes, quorum = ⌊N/2⌋ + 1.

### Why it works

- With 5 nodes, quorum = 3.
- Any two groups that can both make progress must overlap (share at least one node).
- So at most one group can have a quorum during a partition.

### R, W, N (read/write quorum)

- **N**: total replicas
- **W**: write quorum — must receive write from W nodes to consider it committed
- **R**: read quorum — must read from R nodes and merge/compare

If **R + W > N**, then read and write sets overlap → you read at least one up-to-date replica → strong consistency.

Examples:

- N=3, W=2, R=2 → R+W=4 > 3 ✓
- N=3, W=1, R=1 → R+W=2 ≤ 3 → eventual consistency (no overlap guarantee)

---

## 5) Paxos (high-level — what to know)

Paxos is the classic consensus algorithm. You don't need to implement it; you need to explain the idea.

### The idea

- Nodes propose values and vote.
- A value is "chosen" when a majority (quorum) accepts it.
- Once chosen, no other value can be chosen for that round.

### Roles (simplified)

- **Proposer**: proposes a value
- **Acceptor**: votes on proposals
- **Learner**: learns the chosen value

### Why it's hard to explain

- Multiple rounds, complex failure cases, subtle edge cases.
- In practice, systems use Raft (easier to understand) or Paxos variants (Multi-Paxos, etc.).

### What to say in interviews

"Paxos ensures distributed consensus: a quorum must agree on a value. It's correct but complex; Raft is a simpler alternative that achieves the same goal."

---

## 6) Raft (high-level — what to know)

Raft is a consensus algorithm designed to be understandable. Used by etcd, Consul, CockroachDB, and many others.

### The idea

- **Leader-based**: one leader, others are followers.
- Leader handles all client requests; replicates log entries to followers.
- When leader fails, followers elect a new leader via **leader election**.

### Key concepts

1. **Log replication**
   - Leader appends entries to its log.
   - Replicates to followers.
   - Entry is "committed" when a majority have it.

2. **Leader election**
   - Followers have a timeout; if no heartbeat from leader, they become candidates.
   - Candidate requests votes from other nodes.
   - Candidate that gets majority becomes leader.
   - **Term**: monotonically increasing; each election increments it.

3. **Safety**
   - Only nodes with up-to-date logs can become leader.
   - Prevents committed entries from being overwritten.

### What to say in interviews

"Raft uses leader-based replication. The leader replicates log entries to followers; a majority must acknowledge before commit. On leader failure, followers run an election; the candidate with the most up-to-date log wins. This prevents split-brain because only one side can have a majority."

---

## 7) When systems use consensus

| System | Use case |
|--------|----------|
| **etcd, Consul** | Distributed config, service discovery, leader election |
| **Kafka** | Controller (leader) election for partition assignment |
| **ZooKeeper** | Coordination, leader election, distributed locks |
| **CockroachDB, TiKV** | Replicated log for distributed transactions |
| **MongoDB (replica set)** | Primary election |

---

## 8) Failure detection (how do you know the leader is dead?)

Consensus depends on knowing when the leader has failed.

### Timeout-based

- Leader sends heartbeats.
- Followers start election if no heartbeat for X ms.
- **Tradeoff**: short timeout = fast failover, more false positives (network hiccup). Long timeout = slow failover, fewer false positives.

### Lease-based

- Leader holds a lease (e.g., 10 seconds).
- Must renew before expiry.
- If leader crashes, lease expires and followers can elect a new one.
- **Tradeoff**: failover time ≥ lease duration.

### What to say

"We use heartbeat timeouts for failure detection. We tune the timeout to balance failover speed vs false positives from network jitter."

---

## 9) Common interview questions (and answers)

**Q: What is split-brain and how do you prevent it?**  
A: Split-brain is when two nodes both act as leader and accept writes, causing data corruption. We prevent it by requiring a quorum for leader election; during a partition, at most one side has a majority.

**Q: What's the difference between Paxos and Raft?**  
A: Both achieve consensus. Paxos is older and more abstract; Raft is leader-based and designed to be easier to understand and implement. Raft is widely used (etcd, Consul).

**Q: Why do we need a quorum?**  
A: So that during a network partition, at most one side can make progress. Two disjoint groups cannot both have a majority.

**Q: What happens if we have 5 nodes and 3 go down?**  
A: No quorum (need 3). The system cannot elect a leader or commit new writes. It correctly refuses to make progress rather than risk split-brain.

---

## 10) Interview-ready summary

"Consensus ensures all nodes agree on a single leader or value. We use Raft (or similar): leader-based replication, quorum for commits, and leader election on failure. Quorum prevents split-brain — during a partition, only the majority side can elect a leader. We detect failures via heartbeats and timeouts."
