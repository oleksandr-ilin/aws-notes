# Q02: Multi-Site Active/Active DR

## Question

A financial services company operates a critical trading platform that must have **near-zero downtime** and **no data loss** during a regional failure. The application runs on Amazon ECS with Fargate, uses Amazon Aurora Global Database for PostgreSQL, and Amazon ElastiCache for Redis. The company's budget for DR is not a primary concern — **availability is the top priority**. The RTO must be under **1 minute** and RPO must be **0 seconds**.

Which disaster recovery approach should the solutions architect implement?

## Options

- **A.** Deploy a warm standby in a secondary Region with scaled-down ECS services and an Aurora cross-Region read replica. Use Route 53 health checks with failover routing. During a failure, scale up ECS and promote the Aurora replica.
- **B.** Deploy a full multi-site active/active architecture across two Regions. Use Aurora Global Database with write forwarding enabled, Amazon ElastiCache Global Datastore, and Route 53 latency-based routing. Both Regions handle production traffic simultaneously.
- **C.** Use AWS Elastic Disaster Recovery to continuously replicate ECS task definitions and Aurora snapshots. Perform automated failover using Route 53 failover routing and AWS Lambda to orchestrate the recovery.
- **D.** Deploy the application in a single Region across three Availability Zones using Aurora Multi-AZ and ECS spread across AZs. Use an Application Load Balancer to distribute traffic.

## Answers

### B. Multi-site active/active — ✅ Correct

Multi-site active/active is the only strategy that achieves near-zero RTO and zero RPO. Aurora Global Database provides synchronous-like replication with typically under 1-second lag and write forwarding. ElastiCache Global Datastore replicates session data across Regions. Both Regions serve live traffic, so failover is simply a matter of Route 53 removing the unhealthy Region — no infrastructure needs to start up.

### A. Warm standby — ❌ Incorrect

Warm standby requires scaling up ECS services and promoting the Aurora replica during failover. Even with automation, this takes **minutes to tens of minutes**, violating the under-1-minute RTO requirement. Aurora replica promotion alone can take 1–2 minutes.

### C. AWS Elastic Disaster Recovery — ❌ Incorrect

Elastic Disaster Recovery is designed for **server-based workloads** (EC2 instances with block storage), not containerized ECS/Fargate workloads. Its recovery process involves launching instances from replicated data, which takes minutes — not suitable for a sub-1-minute RTO. Also, it doesn't continuously replicate Aurora databases.

### D. Single Region multi-AZ — ❌ Incorrect

Multi-AZ provides high availability within a single Region but offers **no protection against a regional failure**. If us-east-1 experiences a Region-wide outage, the entire application is unavailable. The question specifically requires surviving a regional failure.

## Recommendations

- **Multi-site active/active** is the most expensive but provides the highest availability. Reserve it for truly mission-critical workloads where cost is secondary.
- **Aurora Global Database** is the key enabler for cross-Region database availability with near-zero RPO.
- Use **Route 53 health checks** with latency-based or weighted routing to automatically shift traffic away from a failing Region.
- **ElastiCache Global Datastore** ensures session and cache data survives Region failover.
- Understand that multi-AZ ≠ multi-Region: multi-AZ protects against AZ failures, not Region failures.

## Relevant Links

- [Multi-site Active/Active DR](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/multi-site-active-active.html)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [ElastiCache Global Datastore](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastore.html)
- [Route 53 Routing Policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
