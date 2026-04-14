# Q01: Understanding RPO vs. RTO Trade-offs

## Question

A solutions architect is designing a DR plan for a company that runs an inventory management system. The system uses Amazon EC2 instances and an Amazon RDS for SQL Server database with 2 TB of data. The business has stated:

- "We can tolerate losing up to 1 hour of data."
- "The system must be back online within 4 hours."
- "We need to minimize DR costs."

During a recent DR drill, the team discovered that restoring the RDS database from a cross-Region snapshot took 2 hours and launching/configuring EC2 instances took 1 hour. The remaining time was consumed by DNS propagation and application health checks.

The CTO is concerned that **future data growth** will increase the database restore time beyond the allowed window. Which solution BEST addresses this concern while maintaining cost efficiency?

## Options

- **A.** Switch from snapshot-based recovery to an RDS cross-Region read replica. During failover, promote the read replica (minutes) instead of restoring from a snapshot (hours).
- **B.** Increase the RTO to 8 hours to accommodate future data growth. This avoids additional infrastructure costs.
- **C.** Pre-restore the latest snapshot in the DR Region every hour and keep a warm RDS instance always running with the latest data restored.
- **D.** Migrate from RDS SQL Server to Amazon Aurora with Aurora Global Database for faster cross-Region replication and sub-minute failover.
- **E.** Compress the database backups before copying them to the DR Region to reduce restore time.

## Answers

### A. RDS cross-Region read replica — ✅ Correct

A cross-Region read replica is continuously synchronized with the primary, so promotion takes only minutes regardless of database size. This decouples recovery time from data volume, directly addressing the CTO's concern about data growth. The additional cost is running a read replica in the DR Region, but this is still significantly cheaper than multi-site active/active while keeping recovery well within the 4-hour RTO.

### D. Aurora Global Database — ❌ Incorrect (technically valid but not best)

While Aurora Global Database offers superior cross-Region replication (sub-second lag, sub-minute failover), it requires **migrating from SQL Server to Aurora** — a significant project involving application code changes, testing, and validation. The question asks to address the specific concern about growing restore times, not re-architect the database layer. If the question said "which provides the best long-term solution," this could be correct.

### B. Increase RTO — ❌ Incorrect

Changing business requirements to match technical limitations is not a valid architectural solution. The 4-hour RTO represents a business need (e.g., contractual SLA). A solutions architect should solve the technical constraint, not weaken the requirements.

### C. Pre-restored warm RDS — ❌ Incorrect

Running a full RDS instance continuously in the DR Region is expensive and complex. An hourly pre-restore means the RPO could drift to 2 hours (1 hour of data loss + 1 hour since last restore). More importantly, the restored instance becomes stale immediately after restoration — you'd still need to apply transaction logs to reach the desired RPO. A read replica (option A) is simpler and cheaper.

### E. Compress backups — ❌ Incorrect

RDS automated backups and snapshots are managed by AWS — you cannot control their compression. Even if you could, compression reduces transfer time but **not restore time**, which is the bottleneck. The database engine must decompress and apply data pages regardless of transfer format.

## Recommendations

- **RPO** = acceptable data loss (measured in time). **RTO** = acceptable downtime (measured in time).
- When restore time becomes the bottleneck, shift from **snapshot-based** recovery to **replication-based** recovery (read replicas or Global Database).
- **Read replicas decouple RTO from data size** — promotion time is constant regardless of database volume.
- On the SAP exam, watch for answers that "change the requirements" to match limitations — these are almost always wrong.
- Evaluate migration-heavy answers carefully: if the question focuses on a specific concern, prefer targeted solutions over platform migrations.

## Relevant Links

- [RDS Read Replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
- [RDS Backup and Restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html)
- [RTO and RPO AWS Guidance](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/business-continuity-plan-bcp.html)
