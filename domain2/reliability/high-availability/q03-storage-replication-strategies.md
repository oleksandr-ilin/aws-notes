# Q03: Storage Replication Strategies for Multi-Tier Data Durability

## Question

A media streaming platform stores three categories of data with different durability and availability requirements:

**Category 1 — User-Generated Content (UGC) videos**
- 200 TB, growing 5 TB/month
- Must survive complete Region failure — legal obligation to retain all uploaded content for 5 years
- Current latency for cross-Region access is acceptable (seconds)
- Objects range 100 MB–10 GB

**Category 2 — Real-time session state and playback position**
- 50 GB total, 100,000 writes/second at peak
- Must be available in < 5 ms (same-Region)
- Can be rebuilt from event logs if lost — eventual consistency across Regions acceptable
- Must survive AZ failure but Region failure tolerance is optional

**Category 3 — Analytics database (viewing history, recommendations)**
- 2 TB Aurora PostgreSQL
- Must survive AZ failure with automatic failover < 30 seconds
- Weekly read-heavy analytics queries should not impact production workload
- Cross-Region read access needed for the EU analytics team (eu-west-1) but writes only in us-east-1

Which combination of replication strategies is most appropriate?

## Options

- **A.** Category 1: S3 with Cross-Region Replication (CRR) to eu-west-1 + S3 Lifecycle policy (Standard for 90 days → Standard-IA → Glacier after 1 year). Category 2: ElastiCache for Redis with Multi-AZ (automatic failover replica in different AZ) — no cross-Region replication needed. Category 3: Aurora PostgreSQL Multi-AZ cluster (2 AZ replicas, < 30s failover) + Aurora cross-Region read replica in eu-west-1 for the EU analytics team + use the reader endpoint for analytics queries to isolate from production writes.
- **B.** Category 1: S3 with Same-Region Replication (SRR) to a second bucket in us-east-1. Category 2: DynamoDB with global tables across 2 Regions. Category 3: RDS PostgreSQL Multi-AZ with a read replica in eu-west-1.
- **C.** Category 1: EFS with cross-Region replication enabled. Category 2: ElastiCache Memcached with multiple nodes across AZs. Category 3: Aurora Global Database with write forwarding in eu-west-1.
- **D.** Category 1: S3 in us-east-1 only (11 9's durability is sufficient). Category 2: Redis on EC2 with EBS snapshots every 5 minutes. Category 3: Aurora Serverless v2 with Global Database.

## Answers

### A. S3 CRR + ElastiCache Redis Multi-AZ + Aurora Multi-AZ + cross-Region read replica — ✅ Correct

Each category gets the replication strategy matched to its durability and performance requirements:

- **Category 1 → S3 CRR + Lifecycle**:
  - **S3 Cross-Region Replication**: Asynchronously replicates objects from us-east-1 to eu-west-1. If us-east-1 experiences a complete Region failure, the eu-west-1 replica has all data — meeting the "survive complete Region failure" requirement.
  - S3 already provides 99.999999999% (11 9's) durability within a single Region across 3+ AZs. CRR adds Region-level protection — belt and suspenders for legal retention obligations.
  - **Lifecycle policy**: Video accessed frequently when first uploaded (Standard) → accessed less after 90 days (Standard-IA, 46% cheaper) → rarely accessed after 1 year (Glacier Flexible Retrieval, 90% cheaper). 5-year retention with tiered storage optimizes costs for 200+ TB.
  - Objects 100 MB–10 GB: CRR handles multi-part uploaded objects natively. S3 Replication Time Control (S3 RTC) can be added if a replication SLA is needed.

- **Category 2 → ElastiCache Redis Multi-AZ**:
  - **< 5 ms latency**: ElastiCache Redis provides sub-millisecond latency for reads and single-digit ms for writes — well within the 5 ms requirement.
  - **100,000 writes/second**: Redis cluster mode enabled can handle millions of operations/second distributed across shards. Each shard handles a partition of the keyspace.
  - **Multi-AZ automatic failover**: Redis replication group with a primary + replica in different AZs. If the primary AZ fails, ElastiCache promotes the replica to primary within 15-30 seconds. Data is replicated synchronously to the replica — no data loss on AZ failure.
  - **No cross-Region replication needed**: Session state can be rebuilt from event logs. Region-level failure tolerance is "optional" — so Multi-AZ within the Region is sufficient. This avoids the cost and complexity of ElastiCache Global Datastore.

- **Category 3 → Aurora Multi-AZ + cross-Region read replica**:
  - **Aurora Multi-AZ cluster**: Aurora automatically creates 6 copies of data across 3 AZs. For failover, Aurora uses a cluster with replicas — failover completes in < 30 seconds (typically 15 seconds) by promoting a replica. The cluster endpoint automatically points to the new primary.
  - **Reader endpoint for analytics isolation**: The Aurora reader endpoint load-balances read queries across replicas. Weekly analytics queries (which may scan millions of rows) are sent to the reader endpoint — they consume reader replica resources, not the writer's CPU/memory. This prevents analytics from degrading real-time application performance.
  - **Cross-Region read replica in eu-west-1**: Aurora supports creating a cross-Region read replica. The EU analytics team connects to this replica for low-latency reads. Replication lag is typically < 100 ms. Writes remain in us-east-1 — the cross-Region replica is read-only.
  - **Why not Aurora Global Database**: Global Database is designed for < 1 second RPO and < 1 minute RTO for Region-level DR with read scaling. For this use case, we only need a read replica for the EU team — a simpler cross-Region read replica is sufficient and cheaper than a full Global Database setup.

### B. S3 SRR + DynamoDB Global Tables + RDS Multi-AZ — ❌ Incorrect

- **S3 Same-Region Replication (SRR)**: Replicates within the same Region (to a different bucket, potentially a different account). SRR does NOT protect against Region failure — both source and replica are in the same Region. If us-east-1 fails, both copies are lost. CRR (Cross-Region) is required for Region-level durability.
- **DynamoDB Global Tables for session state**: Global Tables provide cross-Region replication, but the requirement states cross-Region is optional for session state. Global Tables add cost (replicated write capacity in each Region) that isn't needed. Also, DynamoDB's typical latency is 5-10 ms for single-digit-KB items — right at the 5 ms boundary. ElastiCache Redis is consistently < 1 ms.
- **RDS PostgreSQL Multi-AZ**: RDS Multi-AZ (non-Aurora) failover takes 60-120 seconds — exceeding the < 30 second requirement. Aurora Multi-AZ failover is 15-30 seconds. RDS cross-Region read replicas use asynchronous logical replication — slower and more fragile than Aurora's physical replication.

### C. EFS cross-Region + Memcached + Aurora Global Database — ❌ Incorrect

- **EFS for video storage**: EFS is a file system optimized for shared concurrent access (POSIX). Video files (100 MB–10 GB) are typically accessed by key (filename/ID) — S3's object storage model is more appropriate and much cheaper. EFS costs ~$0.30/GB/month vs. S3 Standard at ~$0.023/GB/month — 13× more expensive. At 200 TB, that's $60,000/month (EFS) vs. $4,600/month (S3).
- **Memcached Multi-AZ**: Memcached does NOT support replication or persistence. If a Memcached node fails, all data on that node is lost. There's no automatic failover — the client must handle node failures by routing to remaining nodes. ElastiCache Redis with Multi-AZ provides automatic failover with data preservation.
- **Aurora Global Database with write forwarding for EU analytics**: Write forwarding sends writes from the secondary Region to the primary — adding cross-Region latency. The requirement is "writes only in us-east-1" — the EU team only needs read access. A simple cross-Region read replica is sufficient without the overhead of a full Global Database deployment. Global Database is over-engineered for read-only cross-Region access.

### D. S3 single-Region + Redis on EC2 + Aurora Serverless Global — ❌ Incorrect

- **S3 in us-east-1 only**: S3's 11 9's durability protects against individual object corruption and AZ failures. However, it does NOT protect against catastrophic Region-level events (Region-wide service disruption, government seizure, regulatory action). For a legal obligation to retain content for 5 years, cross-Region replication provides the additional insurance. "11 9's is sufficient" is a durability argument — the requirement is availability and survivability across Region failure.
- **Redis on EC2 with EBS snapshots**: Self-managed Redis on EC2 requires: manual cluster management, failover scripting, monitoring, patching, and backup management. EBS snapshots every 5 minutes provide a 5-minute RPO but recovery requires launching a new instance and restoring the snapshot — RTO of 10-30 minutes. ElastiCache managed service handles all of this with < 30 second automatic failover.
- **Aurora Serverless v2 with Global Database**: Aurora Serverless v2 can participate in Global Database, but Serverless ACU scaling adds latency during cold starts (ACU ramp-up). For a production analytics database with predictable 24/7 usage, provisioned Aurora instances provide more consistent performance. Serverless v2 is better for variable/unpredictable workloads.

## Recommendations

- **S3 CRR** for any data that must survive Region failure — especially when there's a legal retention obligation. Enable S3 RTC for SLA-backed replication timing.
- **ElastiCache Redis Multi-AZ** is the default for sub-millisecond caching with AZ-level durability. Use Global Datastore only when cross-Region read access or Region-level DR is needed.
- **Aurora Multi-AZ vs. Global Database**: Use Multi-AZ for same-Region HA (< 30s failover). Use Global Database for cross-Region DR with < 1 min RTO. Use cross-Region read replicas for read-only cross-Region access without full Global Database overhead.
- **Reader endpoints for workload isolation**: Always send analytics/reporting queries to Aurora reader endpoints — never let batch analytics compete with production OLTP on the writer.
- **Storage selection**: S3 for objects accessed by key. EFS for shared POSIX file systems. EBS for block storage attached to single instances. FSx for high-performance workloads (Lustre for HPC, ONTAP for NAS).

## Relevant Links

- [S3 Cross-Region Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [ElastiCache Redis Multi-AZ](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/AutoFailover.html)
- [Aurora Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [Aurora Cross-Region Read Replicas](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.CrossRegion.html)
- [Aurora Reader Endpoint](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html#Aurora.Endpoints.Reader)
