# Q02: RDS Multi-AZ Instance vs Cluster vs Read Replicas and Redshift Cross-Region DR

## Question

A company runs two database workloads and needs HA/DR solutions for both:

**Workload 1 — OLTP Database (RDS MySQL 8.0)**
- Serves a customer-facing application with strict latency requirements (< 10 ms query response)
- Current: Single-AZ db.r6g.2xlarge instance
- Requirements:
  - Automatic failover to a standby on AZ failure with < 35 seconds of downtime
  - The standby must be readable during normal operation (offload reporting queries)
  - RPO = 0 (zero data loss on failover)

**Workload 2 — Data Warehouse (Redshift ra3.4xlarge, 4-node cluster)**
- Runs nightly analytics queries, dashboards refresh every morning
- Current: Single-Region in us-east-1
- Requirements:
  - The warehouse must be recoverable in eu-west-1 if us-east-1 is completely unavailable
  - RPO < 24 hours (can tolerate losing last day of data)
  - RTO < 4 hours (dashboards must be available within 4 hours)
  - Minimize standby cost — no running Redshift cluster in eu-west-1 during normal operations

Which configuration meets both workloads' requirements?

## Options

- **A.** Workload 1: RDS MySQL Multi-AZ DB **cluster** (2 readable standby instances in different AZs). Workload 2: Redshift automated snapshots with cross-Region snapshot copy enabled — copy snapshots to eu-west-1 automatically. On disaster, restore a Redshift cluster from the latest snapshot in eu-west-1.
- **B.** Workload 1: RDS MySQL Multi-AZ DB **instance** (1 non-readable standby in a different AZ). Workload 2: Redshift cross-Region datasharing for real-time access from eu-west-1.
- **C.** Workload 1: RDS MySQL read replica in a different AZ, with manual promotion on failure. Workload 2: AWS Backup for Redshift with cross-Region copy.
- **D.** Workload 1: RDS MySQL Multi-AZ DB cluster. Workload 2: Run a second Redshift cluster in eu-west-1, keep it in sync with Redshift data sharing.

## Answers

### A. Multi-AZ DB Cluster (readable standby) + Redshift cross-Region snapshot copy — ✅ Correct

**Workload 1 — RDS Multi-AZ DB Cluster:**

- **Multi-AZ DB Cluster** (different from Multi-AZ DB Instance):
  - Deploys a primary DB instance + **2 readable standby instances** across 3 AZs.
  - **Readable standbys**: Unlike Multi-AZ DB Instance (where the standby is NOT readable), the DB Cluster standbys can serve read traffic. This meets the requirement to offload reporting queries to standbys during normal operation.
  - **Failover time: < 35 seconds**: Multi-AZ DB Cluster uses a writer endpoint that automatically redirects to a promoted standby. Typical failover completes in ~35 seconds (compared to 60-120 seconds for Multi-AZ DB Instance). The cluster endpoint DNS update is faster because the standbys are already running and serving reads.
  - **RPO = 0**: Replication uses semi-synchronous replication to at least one standby — the transaction is committed on the primary only after at least one standby acknowledges receipt. Zero data loss on failover.
  - **Supported engines**: MySQL 8.0.28+ and PostgreSQL 13.4+. MySQL 8.0 is supported.

- **Multi-AZ DB Cluster vs. Multi-AZ DB Instance comparison**:

  | Feature | Multi-AZ DB Instance | Multi-AZ DB Cluster |
  |---|---|---|
  | Standby count | 1 | 2 |
  | Standby readable | ❌ No | ✅ Yes |
  | Replication | Synchronous (block-level) | Semi-synchronous (transaction-level) |
  | Failover time | 60-120 seconds | ~35 seconds |
  | RPO | 0 | 0 |
  | Read endpoint | No | Yes (reader endpoint) |
  | Engine support | All RDS engines | MySQL 8.0.28+, PostgreSQL 13.4+ |

**Workload 2 — Redshift Cross-Region Snapshot Copy:**

- **Redshift automated snapshots**:
  - Redshift automatically creates snapshots every 8 hours or after every 5 GB of data changes (whichever comes first). Default retention: 1 day (configurable up to 35 days).
  - Snapshots are stored in S3 (managed by Redshift — you don't see them in your S3 console).

- **Cross-Region snapshot copy**:
  - Enable cross-Region snapshot copy on the Redshift cluster → Redshift automatically copies every automated snapshot to the specified DR Region (eu-west-1).
  - **RPO**: The last automated snapshot was at most 8 hours ago. With additional manual snapshots (e.g., triggered after nightly ETL), RPO can be < 12 hours. Well within the 24-hour RPO requirement.
  - **KMS encryption**: If the source cluster is encrypted, you must configure a snapshot copy grant in the DR Region with a KMS key available in that Region.

- **DR recovery process**:
  1. Restore a Redshift cluster from the latest snapshot in eu-west-1: `restore-from-cluster-snapshot`
  2. Redshift provisions new nodes and restores data from the snapshot. For a 4-node ra3.4xlarge cluster, restore time is typically 30-90 minutes.
  3. Update application connection strings or DNS records to point to the new cluster endpoint.
  4. Total RTO: 1-2 hours — well within the 4-hour requirement.

- **Minimize standby cost**:
  - No Redshift cluster runs in eu-west-1 during normal operations. The only cost is snapshot storage in the DR Region (managed S3 storage at approximately $0.024/GB/month — included in Redshift pricing for automated snapshots, charged for manual snapshots beyond free tier).
  - This is the **backup-and-restore** DR strategy for Redshift — cheapest option because no compute runs in the DR Region until disaster is declared.

### B. Multi-AZ DB Instance + Redshift data sharing — ❌ Incorrect

- **Multi-AZ DB Instance**: The standby is NOT readable. It's a synchronous block-level replica that exists solely for failover. Reporting queries cannot be offloaded to it — they must run on the primary, competing with production OLTP traffic. This violates the "standby must be readable" requirement.
  - Failover time is 60-120 seconds — exceeds the 35-second requirement.
  - RPO = 0 is correct (synchronous replication).

- **Redshift cross-Region data sharing**: Data sharing allows one Redshift cluster to share live data with another cluster. However:
  - Cross-Region data sharing requires a running Redshift cluster in eu-west-1 — this violates the "no running cluster during normal operations" cost minimization requirement.
  - A 4-node ra3.4xlarge cluster costs ~$13,000/month — running this 24/7 as a standby is expensive for a workload that only needs DR with 24-hour RPO.
  - Data sharing is for live data access, not DR. Snapshot copy is the correct DR mechanism for Redshift.

### C. Read replica + manual promotion + AWS Backup for Redshift — ❌ Incorrect

- **Read replica with manual promotion**:
  - Read replicas DO serve read traffic (good for reporting offload), but promotion is MANUAL — not automatic. An operator must: detect the failure, decide to promote, issue the promotion command, and update application connection strings. Total time: 10-30 minutes depending on human response time. This is NOT "automatic failover."
  - Read replicas use asynchronous replication — RPO > 0. During high write load, replication lag can be seconds to minutes. This violates the RPO = 0 requirement.
  - After promotion, the read replica becomes a standalone instance — it's no longer connected to the original primary. If the primary recovers, data must be manually reconciled.

- **AWS Backup for Redshift**: AWS Backup supports Redshift (creates and manages snapshots), but it adds a layer over Redshift's native snapshot capability. For cross-Region copy, Redshift's built-in cross-Region snapshot copy is simpler and purpose-built. AWS Backup provides centralized management (useful for multiple services) but doesn't add functionality for Redshift specifically.
  - AWS Backup can schedule Redshift backups more frequently (every hour) — but the RPO requirement is 24 hours, so Redshift's native automated snapshots (every 8 hours) are sufficient.

### D. Multi-AZ Cluster + running Redshift in eu-west-1 — ❌ Incorrect

- **Multi-AZ DB Cluster**: Correct for Workload 1.
- **Second Redshift cluster in eu-west-1 with data sharing**: Running a 4-node ra3.4xlarge cluster 24/7 in eu-west-1 costs ~$13,000/month. The requirement explicitly states "minimize standby cost — no running Redshift cluster in eu-west-1 during normal operations." Cross-Region snapshot copy achieves the same DR capability at near-zero standby cost (only snapshot storage).
- Data sharing keeps both clusters in sync for real-time queries — this is over-engineered for a workload with 24-hour RPO tolerance. Snapshot-based DR is perfectly adequate and far cheaper.

## Recommendations

- **RDS Multi-AZ DB Cluster** is the recommended HA option when you need: readable standbys, < 35s failover, AND zero data loss. It's available for MySQL 8.0.28+ and PostgreSQL 13.4+.
- **RDS Multi-AZ DB Instance** is the right choice when: (a) you use an engine not supported by DB Cluster (SQL Server, Oracle, MariaDB, older MySQL/PostgreSQL), or (b) you don't need readable standbys.
- **Read replicas** are for read scaling and cross-Region read access — NOT for automatic HA failover within a Region. Use them in combination with Multi-AZ, not as a replacement.
- **Redshift cross-Region snapshot copy** is the standard DR mechanism for Redshift. Enable it on every production Redshift cluster.
- **Redshift Serverless** is an alternative that eliminates cluster sizing concerns but has different DR options (namespace recovery within the same Region).
- **Test Redshift snapshot restoration quarterly** — measure actual restore time and validate data integrity. Document the restore procedure as a runbook.

## Relevant Links

- [RDS Multi-AZ DB Cluster](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html)
- [RDS Multi-AZ DB Instance vs DB Cluster](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html)
- [RDS Read Replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
- [Redshift Cross-Region Snapshot Copy](https://docs.aws.amazon.com/redshift/latest/mgmt/managing-snapshots-console.html#xregioncopy)
- [Redshift Automated Snapshots](https://docs.aws.amazon.com/redshift/latest/mgmt/working-with-snapshots.html)
