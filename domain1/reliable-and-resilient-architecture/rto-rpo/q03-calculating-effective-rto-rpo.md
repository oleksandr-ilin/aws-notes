# Q03: Calculating Effective RTO and RPO

## Question

A solutions architect has designed the following DR architecture for a web application:

- **Database:** Aurora Global Database with a secondary cluster in the DR Region (replication lag: ~200ms)
- **Compute:** AMIs copied to DR Region nightly. Launch template configured for Auto Scaling group (launch time: ~5 minutes for 10 instances)
- **Load Balancer:** ALB created via CloudFormation during failover (~2 minutes)
- **DNS:** Route 53 health check with failover routing (TTL: 60 seconds; detection time: ~90 seconds)
- **Aurora promotion:** ~1 minute
- **Application warm-up and health check:** ~3 minutes

The operations team performs recovery steps **sequentially**. What are the approximate **effective RTO** and **effective RPO** of this architecture?

## Options

- **A.** RTO: ~12 minutes, RPO: ~200 milliseconds
- **B.** RTO: ~12 minutes, RPO: ~24 hours
- **C.** RTO: ~5 minutes, RPO: ~200 milliseconds
- **D.** RTO: ~24 hours, RPO: ~60 seconds

## Answers

### A. RTO: ~12 min, RPO: ~200ms — ✅ Correct

**RTO calculation (sequential recovery):**
1. Route 53 detects failure and updates DNS: ~90 seconds + 60s TTL = ~2.5 minutes
2. (Parallel with DNS) Aurora promotion: ~1 minute
3. CloudFormation deploys ALB: ~2 minutes
4. Auto Scaling launches instances from AMI: ~5 minutes
5. Application warm-up + health checks: ~3 minutes
6. Total: ~2.5 + 2 + 5 + 3 = **~12.5 minutes** (some steps overlap)

**RPO calculation:**
Aurora Global Database replication lag is ~200ms. This means at most 200ms of committed transactions could be lost during failover. The RPO is ~200 milliseconds.

### B. RTO: ~12 min, RPO: ~24 hours — ❌ Incorrect

The RTO is approximately correct, but the RPO is wrong. The RPO of 24 hours would apply if the database relied on daily snapshots. Aurora Global Database provides continuous replication with ~200ms lag, not 24-hour lag. The nightly AMI copy affects **compute recovery** (AMI freshness), not data RPO.

### C. RTO: ~5 min, RPO: ~200ms — ❌ Incorrect

The RPO is correct, but the RTO underestimates recovery time. 5 minutes only accounts for instance launch time — it ignores DNS propagation (~2.5 min), ALB creation (~2 min), and application warm-up (~3 min). All sequential steps must be summed for the true RTO.

### D. RTO: ~24 hours, RPO: ~60 seconds — ❌ Incorrect

Both values are wrong. The 24-hour figure might come from confusing AMI copy frequency with RTO, but AMIs are already in the DR Region — they just might be up to 24 hours stale (which affects instance config, not recovery time). The 60-second RPO would apply if using RDS automated backups, not Aurora Global Database replication.

## Recommendations

- **RTO = sum of all sequential recovery steps.** Draw a timeline and add them up — this is a common exam question pattern.
- **RPO = replication lag of the data store**, not the backup frequency. With continuous replication, RPO equals the replication lag.
- Don't confuse **AMI freshness** (how current the instance image is) with **RPO** (how much data could be lost). AMI staleness affects configuration drift, not transaction data.
- **Route 53 TTL** is part of RTO — cached DNS records in client resolvers mean clients may continue hitting the old Region for up to 1 x TTL after failover.
- To reduce RTO: lower Route 53 TTL, pre-deploy ALB, use warm standby (instances already running).
- To reduce RPO: use Aurora Global Database or DynamoDB Global Tables instead of snapshot-based replication.

## Relevant Links

- [Aurora Global Database Replication](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database-monitoring)
- [Route 53 Health Checks and DNS Failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html)
- [Auto Scaling Group Launch Time](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-instance-launch.html)
- [Calculating RTO/RPO — AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/plan-for-disaster-recovery-dr.html)
