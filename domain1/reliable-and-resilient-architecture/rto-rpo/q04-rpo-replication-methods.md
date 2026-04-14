# Q04: RPO Impact of Different Replication Methods

## Question

A company needs to replicate an Amazon RDS MySQL database (500 GB) from us-east-1 to eu-west-1 for disaster recovery. The company evaluates three replication approaches and wants to understand the **RPO implications** of each.

**Approach 1:** Automated snapshots with cross-Region copy scheduled every 6 hours
**Approach 2:** Cross-Region read replica with asynchronous replication
**Approach 3:** AWS DMS with continuous replication (CDC) to a target RDS instance

Rank the approaches from **best (lowest) RPO** to **worst (highest) RPO**.

## Options

- **A.** Approach 2 (best) → Approach 3 → Approach 1 (worst)
- **B.** Approach 3 (best) → Approach 2 → Approach 1 (worst)
- **C.** Approach 1 (best) → Approach 2 → Approach 3 (worst)
- **D.** All three provide approximately the same RPO since they all replicate to eu-west-1.

## Answers

### A. Read Replica → DMS CDC → Snapshots — ✅ Correct

**Approach 2 — Cross-Region Read Replica: Best RPO (seconds)**
Native MySQL replication uses binary log streaming. The read replica is typically seconds behind the primary. RPO is measured in seconds under normal conditions.

**Approach 3 — DMS with CDC: Good RPO (seconds to minutes)**
DMS continuous replication (Change Data Capture) reads from the source database's transaction log and applies changes to the target. It's typically seconds to low minutes behind. Slightly higher latency than native replication because DMS adds an intermediary processing layer.

**Approach 1 — Snapshots every 6 hours: Worst RPO (up to 6+ hours)**
With 6-hour snapshot intervals, data written after the last snapshot is lost. Additionally, cross-Region copy adds transfer time. Worst case RPO = 6 hours + copy duration.

### B. DMS CDC → Read Replica → Snapshots — ❌ Incorrect

DMS CDC adds an intermediary layer (the DMS replication instance) that introduces slightly more latency than native read replica replication. Native replication uses the database engine's built-in binary log streaming, which is more tightly integrated and lower latency.

### C. Snapshots → Read Replica → DMS — ❌ Incorrect

Snapshots provide the worst RPO because they're point-in-time captures at scheduled intervals. Any data written between snapshots is lost.

### D. Same RPO — ❌ Incorrect

The three approaches use fundamentally different replication mechanisms with very different lag characteristics. "Replicating to the same Region" doesn't mean the same RPO — the replication method matters.

## Recommendations

- **RPO hierarchy** for RDS MySQL (best to worst):
  1. Aurora Global Database (~1s replication lag)
  2. Native cross-Region read replica (seconds)
  3. DMS CDC (seconds to minutes)
  4. Automated snapshots + cross-Region copy (hours)
- **Read replicas vs. DMS CDC:** Use read replicas when source and target are the same engine. Use DMS when migrating between different engines or when you need data transformation during replication.
- **Snapshot RPO formula:** RPO = snapshot interval + copy time + restore time (for RTO). Always add cross-Region copy time.
- On the exam, if a question mentions "continuous replication," associate it with seconds-level RPO. If it mentions "scheduled snapshots," associate it with hours-level RPO.

## Relevant Links

- [RDS MySQL Read Replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_MySQL.Replication.ReadReplicas.html)
- [AWS DMS CDC](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Task.CDC.html)
- [RDS Automated Backups](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html)
- [Cross-Region Snapshot Copy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CopySnapshot.html)
