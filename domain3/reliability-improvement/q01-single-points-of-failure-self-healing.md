# Q01: Remediating Single Points of Failure and Implementing Self-Healing

## Question

A financial services company runs its loan origination system with the following architecture, deployed in a single AZ:

| Component | Configuration | Issue |
|---|---|---|
| Application tier | 2 × EC2 m5.2xlarge in us-east-1a, behind ALB | Single AZ — AZ failure takes down both instances |
| Database | RDS PostgreSQL db.r5.2xlarge, Single-AZ | No standby — AZ failure = total database outage |
| File storage | EBS volumes (gp3) attached to EC2 | EBS is AZ-scoped — data lost if AZ fails |
| Cache | Single ElastiCache Redis node, no cluster mode | Node failure = cold cache + data loss |
| Message queue | SQS standard queue | ✅ Multi-AZ by default |
| External API calls | Direct HTTP calls to 3rd-party APIs | No retry logic, no circuit breaker |

**Recent incidents:**
1. **AZ outage** (us-east-1a, 45 minutes): Complete system outage. RTO target is 15 minutes — missed by 3×.
2. **ElastiCache node failure**: Cache rebuilt from empty — 30 minutes of cache-miss storm hammered the database, causing cascading slowdown.
3. **3rd-party API timeout**: External credit scoring API timed out for 10 minutes. Application threads blocked waiting for responses. New loan applications couldn't be submitted even though the rest of the system was healthy.

The CTO requires: eliminate single points of failure, achieve RTO < 15 minutes for AZ failure, and implement self-healing to reduce operator intervention.

Which approach addresses all three incident types?

## Options

- **A.** **AZ resilience**: Spread EC2 ASG across 3 AZs (us-east-1a, 1b, 1c) with minimum 3 instances. Enable RDS Multi-AZ (synchronous standby in different AZ — automatic failover < 120 seconds). Replace EBS file storage with EFS (Multi-AZ by default). **Cache resilience**: Enable ElastiCache Multi-AZ with automatic failover (Redis replication group: 1 primary + 2 replicas across AZs). Enable `append-only file (AOF)` persistence so the replica rebuilds from the AOF log on promotion (warm cache, not cold). **Self-healing**: Configure ASG health checks using ALB target group health (not just EC2 status). Set health check grace period to 120 seconds. ASG automatically replaces unhealthy instances. For database: RDS automatic failover handles AZ failure. For cache: ElastiCache automatic failover promotes a replica within 30 seconds. **External API resilience**: Implement circuit breaker pattern (AWS Step Functions or application-level with resilience library). Set: 5-second timeout, 3 retries with exponential backoff, circuit opens after 5 consecutive failures. When circuit is open: return cached credit scores (from ElastiCache) or queue the request in SQS for async processing. Add CloudWatch alarm on circuit breaker state — notify operations when a 3rd-party dependency is degraded.
- **B.** Deploy the entire application in a second Region (us-west-2) with Route 53 failover routing. If us-east-1 fails, Route 53 routes traffic to us-west-2. RDS cross-region read replica is promoted to primary.
- **C.** Keep Single-AZ deployment but increase instance sizes (m5.4xlarge) for more capacity. Add CloudWatch alarms to page operators when issues occur. Create runbooks for manual failover procedures.
- **D.** Use AWS Elastic Disaster Recovery (DRS) to replicate all instances to us-east-1b. When an AZ failure occurs, launch the recovery instances in the secondary AZ. Set up automated launch with EventBridge.

## Answers

### A. Multi-AZ + ElastiCache replication + circuit breaker — ✅ Correct

**Eliminating Single Points of Failure — AZ Level:**

- **EC2 ASG across 3 AZs**: With `min=3, desired=3` across 3 AZs, each AZ has at least 1 instance. If an AZ fails:
  - ALB detects unhealthy targets in the failed AZ (health check fails within 30 seconds)
  - ALB stops routing traffic to the failed AZ
  - ASG detects instance(s) are unhealthy → launches replacement(s) in healthy AZs
  - Remaining 2 instances handle traffic during the launch (capacity drops by 33%, not 100%)
  - Set `desired=4` or `desired=6` if 33% capacity loss is unacceptable during failover

- **RDS Multi-AZ deployment**:
  - RDS creates a synchronous standby in a different AZ (data is written to both primary and standby before commit is acknowledged)
  - During AZ failure: RDS detects primary is unreachable → promotes standby to primary → DNS endpoint resolves to new primary
  - **Failover time**: 60-120 seconds. Application retries database connections automatically (connection pool refreshes after DNS TTL)
  - **No data loss**: Synchronous replication ensures standby has all committed transactions
  - For RDS PostgreSQL: Multi-AZ deployment also protects against storage failure, instance failure, and OS patching (failover during maintenance window)

- **EBS → EFS migration**:
  - EBS volumes are AZ-scoped. If us-east-1a fails, data on EBS volumes in that AZ is inaccessible until the AZ recovers
  - EFS is a regional service: data is stored across multiple AZs automatically. All EC2 instances (regardless of AZ) mount the same EFS filesystem
  - EFS supports POSIX permissions, NFS locking, and thousands of concurrent connections
  - **Performance**: Use EFS General Purpose mode for most workloads. For high-throughput document processing, use EFS Max I/O mode or Elastic Throughput
  - **Cost optimization**: Enable EFS Intelligent-Tiering to automatically move infrequently accessed files to EFS IA (90% cheaper per GB)

**Cache Resilience — ElastiCache Multi-AZ:**

- **Replication group** (1 primary + 2 replicas across AZs):
  - Primary handles reads and writes; replicas handle reads and serve as failover targets
  - If primary node fails: ElastiCache promotes a replica to primary within 15-30 seconds (with Multi-AZ enabled)
  - Replicas have a copy of the data — no cold cache on failover. The promoted replica serves requests immediately with warm data

- **AOF persistence**:
  - Append-Only File logs every write operation. On node restart or replica promotion, data is reconstructed from the AOF
  - Without AOF: a node restart means cold cache (all data lost). With AOF: data is recovered (minus any writes during the failure window)
  - Set `appendfsync everysec` — AOF flushes to disk every second (max 1 second of data loss on crash, with minimal performance impact)

- **Cache-aside with fallback**: In the application, implement cache-aside pattern with fallback:
  ```
  get(key):
    if cache.get(key) → return cached value
    if cache is unavailable → query database directly (with rate limiting to prevent overwhelming RDS)
    set cache with TTL
  ```
  This prevents a complete outage if ElastiCache is temporarily unavailable.

**Self-Healing — Reducing Operator Intervention:**

- **ALB health checks** (not just EC2 status checks):
  - EC2 status checks only detect hardware/hypervisor failures. An instance with a crashed application process passes EC2 status checks.
  - ALB health checks hit the application's `/health` endpoint. If the application doesn't respond (crash, OOM, deadlock), ALB marks it unhealthy → ASG replaces it.
  - **Health check grace period**: Set to 120 seconds (time for the instance to boot and pass health checks before ASG marks it unhealthy on first launch).

- **RDS auto-failover**: No operator action needed. RDS handles failover automatically when the primary becomes unreachable.

- **ElastiCache auto-failover**: Enabled with `--automatic-failover-enabled` on the replication group. Failover completes in 15-30 seconds without operator action.

**External API Resilience — Circuit Breaker Pattern:**

- **Problem**: Without a circuit breaker, application threads block on 3rd-party API timeouts. With a 60-second default timeout and 100 concurrent requests, 100 threads are blocked → thread pool exhaustion → application can't serve ANY requests (including those that don't need the external API).

- **Circuit breaker states**:
  1. **Closed** (normal): Requests flow to the external API. Track failure rate.
  2. **Open** (degraded): After 5 consecutive failures, stop sending requests to the external API. Return a fallback response (cached credit score, or queue for async processing).
  3. **Half-open** (testing): After 30 seconds, allow one test request. If it succeeds, close the circuit. If it fails, re-open.

- **Implementation options**:
  - Application-level: Use a resilience library (Resilience4j for Java, Polly for .NET, timeout + retry middleware for Node.js)
  - AWS Step Functions: Use Step Functions for the API call with built-in retry, timeout, and catch/fallback states
  - API Gateway: Place API Gateway in front of the 3rd-party API with caching, throttling, and timeout settings

- **Fallback strategies**:
  - Return the most recent cached credit score from ElastiCache (acceptable for initial assessment — final underwriting can use the live score later)
  - Queue the loan application in SQS. Process it asynchronously when the external API recovers. Notify the user: "Your application is being processed" (better UX than an error page)

### B. Multi-Region deployment — ❌ Incorrect

- **Over-engineered for AZ-level resilience**: Multi-Region protects against an entire Region failure (extremely rare). The incidents described are AZ-level and component-level failures — Multi-AZ addresses these at lower cost and complexity.
- **RDS cross-region read replica promotion**: Cross-region read replicas use asynchronous replication (potential data loss). Promotion is manual (requires operator intervention) and takes 5-15 minutes. This doesn't meet the "self-healing" or "< 15 minute RTO" requirements without additional automation.
- **Cost**: Running full stacks in 2 Regions doubles infrastructure cost. Multi-AZ adds ~30% cost (standby instances + Multi-AZ RDS surcharge).

### C. Bigger instances + manual runbooks — ❌ Incorrect

- **Bigger instances don't add resilience**: A larger instance in a single AZ is still a single point of failure. When the AZ fails, instance size is irrelevant — it's down.
- **CloudWatch alarms + manual runbooks**: Alarms notify operators (5-minute detection + 5-minute manual response + 5-minute execution = 15 minutes minimum). This barely meets RTO and requires 24/7 operator availability.
- **Self-healing requires automation**: The requirement is to "reduce operator intervention." Manual runbooks are the opposite of self-healing. RDS Multi-AZ auto-failover and ASG auto-replacement are self-healing. Manual runbooks are disaster recovery.

### D. Elastic Disaster Recovery (DRS) for AZ failover — ❌ Incorrect

- **DRS is for Region-level disaster recovery**, not AZ-level. DRS continuously replicates block storage from source servers to a staging area in another Region. It's designed for on-premises-to-AWS migration and cross-Region DR.
- **AZ-level resilience should use Multi-AZ deployments**: RDS Multi-AZ, ElastiCache Multi-AZ, ASG across AZs. These are AWS-native, automatic, and faster than DRS launch.
- **DRS launch time**: DRS launches recovery instances on demand (not pre-provisioned). Launch + boot + configuration takes 5-15 minutes. RDS Multi-AZ failover takes 60-120 seconds. ASG launches replacements in 2-3 minutes.
- **DRS doesn't handle ElastiCache or application state**: DRS replicates EC2 block storage. It doesn't replicate ElastiCache data, SQS messages, or ALB configuration. You'd still need Multi-AZ for those components.

## Recommendations

- **Start with Multi-AZ, then Multi-Region**: Multi-AZ handles 95% of failure scenarios (AZ outage, instance failure, storage failure). Add Multi-Region only when business requirements demand Region-level resilience (regulatory, RPO < 1 second globally).
- **Health check hierarchy**: Use ALB application health checks (not EC2 status checks) for ASG health evaluation. Test the health check endpoint includes dependency checks (database connectivity, cache availability).
- **Circuit breaker for ALL external dependencies**: Apply the circuit breaker pattern to every external API call (3rd-party APIs, cross-service calls, database connections). Identify fallback behavior for each dependency ahead of time.
- **Chaos Engineering**: Use AWS Fault Injection Service (FIS) to simulate AZ failures, instance termination, and network disruption. Validate that self-healing mechanisms work as expected BEFORE a real incident.
- **EBS → EFS evaluation**: Not all workloads benefit from EFS migration. If files are instance-specific (logs, temp files), keep EBS. If files are shared across instances (document uploads, configuration), use EFS or S3.

## Relevant Links

- [RDS Multi-AZ Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [ElastiCache Multi-AZ with Auto-Failover](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/AutoFailover.html)
- [ASG Health Checks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-health-checks.html)
- [EFS Overview](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)
- [Circuit Breaker Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html)
- [Fault Injection Service](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
