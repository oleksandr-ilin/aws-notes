# Q05: Warm Standby with Automated Scaling

## Question

A SaaS company operates a multi-tier web application in **ap-southeast-1**. The application serves customers across Asia-Pacific and processes approximately 10,000 requests per second at peak. The architecture includes an Auto Scaling group of EC2 instances (min 20, max 50), an Aurora PostgreSQL cluster, and Amazon ElastiCache Redis. Management requires a DR strategy with an RTO of **15 minutes** and RPO of **1 minute**, and insists that DR infrastructure must be **continuously validated** through regular testing without impacting production.

Which strategy best meets these requirements?

## Options

- **A.** Implement a pilot light strategy with Aurora Global Database and pre-configured launch templates. Run monthly manual DR drills by launching EC2 instances from templates.
- **B.** Implement a warm standby in ap-northeast-1 with a scaled-down Auto Scaling group (min 2, max 50), Aurora Global Database secondary cluster, ElastiCache Global Datastore, and Route 53 failover routing. Use AWS Fault Injection Simulator (FIS) to run automated DR validation experiments.
- **C.** Deploy a full-scale active/active environment in ap-northeast-1 with identical capacity. Use Route 53 weighted routing to send 50% of traffic to each Region.
- **D.** Use AWS Elastic Disaster Recovery to replicate all EC2 instances. Schedule weekly recovery drills using DRS's built-in drill feature. Use RDS automated backups with cross-Region copy.

## Answers

### B. Warm standby with FIS — ✅ Correct

A warm standby runs a **scaled-down but fully functional** copy of the production environment. With a min of 2 instances always running, the DR environment can immediately start serving traffic while Auto Scaling grows to full capacity within the 15-minute RTO window. Aurora Global Database provides sub-minute RPO. ElastiCache Global Datastore preserves cache data. AWS Fault Injection Simulator enables **automated, repeatable DR testing** without manual effort — directly addressing the "continuously validated" requirement.

### A. Pilot light with manual drills — ❌ Incorrect

Pilot light can potentially meet the RPO with Aurora Global Database, but launching 20+ EC2 instances from templates, waiting for health checks, and configuring load balancers typically takes **20–40 minutes** — exceeding the 15-minute RTO. Monthly manual drills don't satisfy "continuously validated" — they're infrequent and require human effort.

### C. Full-scale active/active — ❌ Incorrect

A full-scale active/active deployment would meet the RTO/RPO and be continuously validated (since both Regions handle real traffic). However, running 20+ instances in a second Region doubles the compute cost unnecessarily. The requirements don't demand near-zero RTO — 15 minutes is achievable with warm standby at a fraction of the cost.

### D. Elastic Disaster Recovery with RDS backups — ❌ Incorrect

DRS is designed for EC2 instance replication and works well, but **RDS automated backups with cross-Region copy** introduces significant delay — backup copy can take minutes to hours depending on size, and restore time adds more. This doesn't reliably meet the 1-minute RPO for the database. Aurora Global Database is the correct choice for sub-minute RPO. Additionally, DRS launches fresh instances during failover, adding time that may push beyond 15 minutes for 20+ instances.

## Recommendations

- **Warm standby** is the sweet spot when RTO is 10–30 minutes and RPO is minutes or less.
- Keep the warm standby at **minimal capacity** (2–5 instances) and rely on **Auto Scaling** to rapidly scale up during failover.
- **AWS Fault Injection Simulator (FIS)** is the key service for automated, continuous DR validation — remember it for any question mentioning DR testing/validation.
- **ElastiCache Global Datastore** is often overlooked — include it when the architecture uses caching to avoid cold-cache performance issues after failover.
- The scaling sequence during warm-standby failover: Route 53 shifts traffic → warm standby instances absorb initial load → Auto Scaling adds capacity → full capacity reached.

## Relevant Links

- [Warm Standby DR Strategy](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/warm-standby.html)
- [AWS Fault Injection Simulator](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
- [ElastiCache Global Datastore](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastore.html)
- [Aurora Global Database Failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
