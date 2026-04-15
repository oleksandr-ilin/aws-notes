# Q01: Automated Cross-Region DR with Database Replication

## Question

An insurance company processes claims through a web application running in us-east-1. The application uses Aurora PostgreSQL with a single-writer cluster and ElastiCache Redis for session management. The stated requirements are:
- RPO ≤ 1 minute and RTO ≤ 15 minutes
- If us-east-1 becomes unavailable, the application must serve traffic from eu-west-1
- Session data must survive a regional failover without forcing users to re-authenticate
- The DR environment in eu-west-1 should remain cost-efficient during normal operations
- Failover and failback must be tested monthly without impacting production

Which DR configuration should the solutions architect design?

## Options

- **A.** Create an Aurora Global Database with the primary cluster in us-east-1 and a secondary cluster in eu-west-1. Configure ElastiCache Global Datastore with a primary replication group in us-east-1 and a secondary in eu-west-1. In eu-west-1, deploy a scaled-down Auto Scaling group (min=1) with the same AMI and launch template. Use Route 53 failover routing with health checks against the ALB in us-east-1. Use AWS FIS to simulate Region failure monthly by temporarily blocking ALB health check responses.
- **B.** Take hourly Aurora snapshots and copy them to eu-west-1. Export ElastiCache RDB snapshots to S3 with Cross-Region Replication. In eu-west-1, keep no running infrastructure — deploy from CloudFormation on failover. Use Route 53 failover routing for DNS cutover.
- **C.** Use Aurora Multi-AZ in us-east-1 with 2 read replicas. Use ElastiCache Multi-AZ with automatic failover. Deploy the full application stack identically in eu-west-1 running at production capacity. Use Route 53 active-active (weighted 50/50) routing.
- **D.** Use AWS DMS with continuous replication from Aurora us-east-1 to a standalone RDS PostgreSQL instance in eu-west-1. Use application-level session replication via SQS cross-Region. Deploy application servers in eu-west-1 on Spot instances to save cost.

## Answers

### A. Aurora Global Database + ElastiCache Global Datastore + warm standby — ✅ Correct

This meets all requirements:
- **Aurora Global Database**: Replication lag is typically < 1 second (well within RPO ≤ 1 minute). Managed failover promotes the eu-west-1 secondary cluster to writer in ~1 minute. No snapshot restore delay.
- **ElastiCache Global Datastore**: Replicates Redis data cross-Region with sub-second lag. On failover, the secondary becomes primary — sessions survive without user re-authentication.
- **Warm standby (scaled-down)**: A small ASG (min=1) in eu-west-1 keeps instances running and warm. On failover, the ASG scales up to production capacity — achievable within the 15-minute RTO. Cost-efficient because capacity is minimal during normal operations.
- **Route 53 failover routing**: Health checks detect ALB failure in us-east-1 and route traffic to eu-west-1 ALB. DNS TTL should be ≤ 60s for fast cutover.
- **FIS testing**: AWS Fault Injection Simulator can inject failures (e.g., block health check endpoint) to test the full failover chain without destroying infrastructure. Monthly testing validates the runbook.

### B. Hourly snapshots + deploy from CloudFormation — ❌ Incorrect

- **Hourly snapshots** = up to 60-minute RPO — violates the ≤ 1 minute requirement.
- **Deploy from CloudFormation on failover** = cold start infrastructure (EC2, ALB, restore DB from snapshot) which takes 30-60+ minutes — violates the 15-minute RTO.
- ElastiCache RDB snapshots exported to S3 are periodic, not continuous — session data loss is likely.
- This is a **backup and restore** strategy, suitable for RTO of hours, not minutes.

### C. Multi-AZ + full active-active — ❌ Incorrect

- **Aurora Multi-AZ and ElastiCache Multi-AZ** are single-Region HA — they protect against AZ failures, not Region failures. If us-east-1 becomes unavailable, these provide no protection.
- Running the **full stack at production capacity** in eu-west-1 doubles cost — not cost-efficient during normal operations.
- **Active-active weighted 50/50** means both Regions serve traffic continuously — this is a multi-site strategy, not a warm standby. It requires write conflict resolution and is overkill for the stated RTO/RPO.

### D. DMS + SQS sessions + Spot instances — ❌ Incorrect

- **DMS continuous replication** works but adds latency and complexity vs Aurora Global Database's native replication. DMS requires a replication instance to manage.
- **SQS for session replication** is not a session store — SQS is a message queue. Sessions require key-value store semantics (get/set by session ID), not message consumption.
- **Spot instances** can be interrupted with 2-minute notice — using them for DR readiness means the DR environment may not be available when needed. DR infrastructure should use On-Demand or Reserved.

## Recommendations

- **Aurora Global Database** is the go-to for cross-Region relational DR: managed replication, managed failover, and RPO < 1 second.
- **ElastiCache Global Datastore** ensures session continuity — critical for user experience during failover.
- **Warm standby** balances cost and RTO: keep a minimal footprint running, scale up on failover. For RTO < 5 minutes, consider pre-scaling to nearer production capacity.
- **Test DR monthly** using FIS: inject controlled failures, measure actual RTO/RPO, and iterate on the runbook.
- After failover, plan for **failback**: re-establish Aurora Global Database with us-east-1 as secondary, replicate, then switch back during a maintenance window.

## Relevant Links

- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [ElastiCache Global Datastore](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastore.html)
- [Route 53 Failover Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
- [AWS Fault Injection Simulator](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
