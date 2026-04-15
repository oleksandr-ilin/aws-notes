# Q01: Highly Available Multi-AZ Architecture Design

## Question

A company is designing a new customer portal that must achieve 99.99% availability. The application is a stateless Java REST API with a PostgreSQL database and Redis-based session cache. Requirements:
- The application must survive an entire Availability Zone failure without manual intervention
- Database failover must complete in under 30 seconds
- User sessions must be preserved during AZ failure
- Health checks must detect unhealthy instances within 10 seconds
- The architecture must support scaling from 100 to 10,000 concurrent users

Which architecture should the solutions architect design?

## Options

- **A.** Deploy a minimum of 3 EC2 instances across 3 AZs in an Auto Scaling group behind an ALB. Use Aurora PostgreSQL Multi-AZ (1 writer + 2 readers across AZs) with Aurora failover priority. Deploy ElastiCache Redis with Multi-AZ and automatic failover (cluster mode enabled with replicas in each AZ). Configure ALB health checks with 10-second interval and 2-check threshold. Use target tracking scaling on ALB request count per target.
- **B.** Deploy 2 EC2 instances in 2 AZs behind a Network Load Balancer. Use RDS PostgreSQL Multi-AZ (single-standby). Deploy a single ElastiCache Redis node. Configure NLB TCP health checks. Use simple scaling with CloudWatch CPU alarms.
- **C.** Deploy the application on a single large EC2 instance (scale-up approach) in one AZ. Use RDS with automated backups for recovery. Use application-level session storage in local memory. Configure Route 53 health checks for the single instance.
- **D.** Deploy the application on ECS Fargate across 2 AZs. Use Aurora Serverless v1 for the database. Use DynamoDB for session storage. Configure ALB health checks with 60-second interval.

## Answers

### A. 3-AZ ASG + Aurora Multi-AZ + ElastiCache Multi-AZ — ✅ Correct

This achieves 99.99% availability:
- **3 AZs with ASG**: For 99.99% availability, 3 AZs are recommended. If one AZ fails, the remaining 2 AZs have 67% of original capacity already running — ASG launches replacements. With 2 AZs, an AZ failure leaves only 50% capacity, which may be insufficient during scale.
- **Aurora PostgreSQL Multi-AZ**: Aurora's failover typically completes in < 30 seconds (usually under 15s). With 2 readers across AZs, a reader is promoted to writer almost instantly. Aurora replicas share the same storage volume — no data replication delay on failover.
- **ElastiCache Redis Multi-AZ with cluster mode**: Replicas in each AZ serve reads. On primary failure, automatic failover promotes a replica in another AZ. Cluster mode distributes data across shards for better scaling. Sessions survive AZ failure because Redis replicas in other AZs have the data.
- **ALB health checks (10s interval, 2 threshold)**: Detects unhealthy targets within 20 seconds. ALB stops routing to unhealthy targets immediately after detection.
- **Target tracking scaling on ALB request count**: Automatically adjusts capacity based on actual load — handles 100 to 10,000 users smoothly.

### B. 2-AZ NLB + RDS Multi-AZ + single Redis — ❌ Insufficient

- **2 AZs**: An AZ failure halves capacity, potentially causing brownouts while ASG scales. 2-AZ designs typically achieve 99.95%, not 99.99%.
- **RDS Multi-AZ (non-Aurora)**: Failover takes 60-120 seconds (DNS-based), exceeding the 30-second requirement. RDS uses synchronous replication to a standby, but the switchover involves DNS record update and connection re-establishment.
- **Single Redis node**: No failover capability — AZ failure loses all sessions. No Multi-AZ resilience.
- **NLB TCP health checks**: Cannot inspect HTTP response codes — an application returning 500 errors would still appear healthy at TCP level.
- **Simple scaling + CPU alarm**: Reactive and slow — cooldown periods delay subsequent scaling actions.

### C. Single instance — ❌ Incorrect

- A single instance in one AZ is a single point of failure — any AZ failure or instance failure causes complete downtime. 99.99% availability is impossible.
- Local memory sessions are lost on any instance restart or failure.
- RDS automated backups require restore time (hours) — not failover.
- Scale-up has a ceiling: the largest instance type still has finite resources.

### D. Fargate 2-AZ + Aurora Serverless v1 + DynamoDB sessions — ❌ Partially incorrect

- **2 AZs**: Same concern as option B — insufficient for 99.99%.
- **Aurora Serverless v1**: Has scaling pauses during capacity changes (cold starts of 25+ seconds). V1 doesn't support read replicas or global databases. **Aurora Serverless v2** would be acceptable, but v1 has significant limitations.
- **DynamoDB sessions**: Actually a good choice — DynamoDB is Multi-AZ by design and highly available. But combined with the other limitations, this doesn't meet all requirements.
- **60-second health check interval**: Too slow — failing instances serve errors for up to 60 seconds before detection. The requirement is 10-second detection.

## Recommendations

- **3-AZ minimum** for 99.99% targets. Size each AZ to handle N-1 load (if 3 AZs, each should handle 50% of peak).
- **Aurora** over RDS for fast failover: Aurora replicas share storage, so promotion is near-instant. RDS failover involves DNS changes and storage operations.
- **ElastiCache Redis** with cluster mode enabled for production: sharding provides both scalability and AZ-level redundancy.
- **ALB health checks**: Use HTTP health checks (not TCP) with short intervals. Configure a dedicated health check endpoint that verifies downstream dependencies (DB, cache connectivity).
- Use **cross-zone load balancing** (enabled by default on ALB) to distribute traffic evenly across AZs even if instance counts differ.
- Monitor **AZ imbalance**: ASG may have uneven instance distribution during scaling — enable AZ rebalancing.

## Relevant Links

- [Multi-AZ Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [Aurora Failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [ElastiCache Multi-AZ](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/AutoFailover.html)
- [ALB Health Checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)
- [Auto Scaling Target Tracking](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
