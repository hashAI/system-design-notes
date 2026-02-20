# Disaster recovery: RTO and RPO — practical guide

When systems fail — region outage, data center fire, ransomware — you need a plan. **RTO** and **RPO** are the two numbers that define how much downtime and data loss you can tolerate.

---

## 0) The default mental model (use in every design)

- **RPO (Recovery Point Objective)**: How much data loss is acceptable? "We can lose at most 1 hour of data."
- **RTO (Recovery Time Objective)**: How long can we be down? "We must be back within 4 hours."

These drive your backup, replication, and failover strategy. Lower RPO/RTO = more cost and complexity.

---

## 1) RPO — Recovery Point Objective

### What it means

**RPO** = maximum acceptable amount of data loss measured in time.

- RPO of 1 hour → you can lose up to 1 hour of writes.
- RPO of 0 (zero) → no data loss acceptable (requires synchronous replication).

### How it drives design

| RPO | Typical strategy |
|-----|------------------|
| **Hours** | Periodic backups (e.g., daily), async replication |
| **Minutes** | Frequent backups, async replication with short lag |
| **Seconds** | Async replication with near-real-time sync |
| **Zero** | Synchronous replication (both regions confirm before commit) |

### Tradeoff

- Lower RPO → more replication, more sync, higher latency, higher cost.
- Synchronous replication across regions adds significant latency.

---

## 2) RTO — Recovery Time Objective

### What it means

**RTO** = maximum acceptable downtime.

- RTO of 4 hours → you have 4 hours to restore service.
- RTO of minutes → you need hot standby, automatic failover.

### How it drives design

| RTO | Typical strategy |
|-----|------------------|
| **Hours/days** | Restore from backup, manual failover |
| **Minutes** | Hot standby in another region, manual or semi-automated failover |
| **Seconds** | Active-active or active-passive with automatic failover |

### Tradeoff

- Lower RTO → more redundancy, automation, testing, cost.
- Automatic failover requires careful design (split-brain, data consistency).

---

## 3) Backup strategies

### Full backup

- Copy entire dataset periodically.
- Simple, but slow and storage-heavy.
- Restore = restore last full backup + any incrementals.

### Incremental backup

- Only backup what changed since last backup.
- Faster, less storage.
- Restore = full + all incrementals (slower restore).

### Continuous replication

- Stream changes to replica(s) in real time.
- Lowest RPO.
- Used with failover for low RTO.

### Snapshot

- Point-in-time copy (e.g., EBS snapshot, DB snapshot).
- Good for "restore to this moment" before a bad migration.

---

## 4) Multi-region strategies

### Active-passive (hot standby)

- Primary region serves traffic.
- Standby region has replicas, ready to take over.
- Failover: DNS/load balancer switches to standby.
- RTO: minutes (with automation) to hours (manual).

### Active-active

- Both regions serve traffic.
- Lowest RTO (no failover — both already up).
- Harder: conflict resolution, data consistency, routing.

### Pilot light

- Minimal resources in standby (e.g., DB replicas, no app servers).
- On disaster, scale up app tier in standby.
- Lower cost than hot standby; higher RTO.

---

## 5) What to test

- **Backup restore**: Regularly restore from backup to verify it works.
- **Failover drills**: Run failover to standby; verify data and traffic.
- **Chaos engineering**: Kill regions, corrupt data — practice recovery.

Senior signal: "We test our disaster recovery quarterly. Last drill, we failed over to the standby region in 12 minutes."

---

## 6) Common interview questions (and answers)

**Q: What's the difference between RTO and RPO?**  
A: RPO is how much data loss we accept (recovery point). RTO is how long we can be down (recovery time). They drive backup frequency and failover strategy.

**Q: How do you achieve RPO of zero?**  
A: Synchronous replication: both primary and standby must confirm before commit. Adds latency. Used for critical data (payments, ledger).

**Q: How do you achieve low RTO?**  
A: Hot standby in another region, automated failover, regular failover drills. Active-active gives lowest RTO but is complex.

**Q: What's in your disaster recovery plan?**  
A: RTO and RPO targets, backup strategy (full + incremental, retention), replication to standby region, failover runbook, and quarterly DR drills.

---

## 7) Interview-ready summary

"RPO defines acceptable data loss; RTO defines acceptable downtime. We use async replication and backups for moderate RPO; sync replication for zero data loss. For low RTO, we use hot standby and automated failover. We test DR regularly with restore and failover drills."
