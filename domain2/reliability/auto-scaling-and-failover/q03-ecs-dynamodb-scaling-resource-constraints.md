# Q03: ECS Service Auto Scaling, DynamoDB Throughput Scaling, and Resource Constraints

## Question

A company runs a food delivery platform with three workloads that must independently scale during peak hours (11 AM – 1 PM lunch, 6 PM – 9 PM dinner):

**Workload 1 — Order API (ECS Fargate)**
- 20 ECS tasks during off-peak, needs to scale to 200 tasks during dinner peak
- Each task handles 50 concurrent requests
- Current problem: during the dinner ramp-up, new tasks take 45 seconds to start. By the time tasks are ready, the backlog of requests has already caused timeouts
- Requirement: scale based on incoming request rate (requests per task), not CPU

**Workload 2 — Order Processing (DynamoDB)**
- Table stores order records with partition key `orderId`
- Off-peak: 500 WCU / 2,000 RCU
- Dinner peak: 5,000 WCU / 20,000 RCU (10x spike in 15 minutes)
- Current problem: DynamoDB on-demand mode throttles during sudden spikes because the table's previous peak was only 3,000 WCU. On-demand doubles the previous peak — but the table hasn't seen 5,000 WCU before
- Requirement: handle 10x spikes reliably without throttling

**Workload 3 — Real-time Dashboards (ElastiCache Redis)**
- 3-node Redis cluster (r6g.xlarge) serves driver location data
- Current: max connections per node = 65,000. During peak, 180 ECS tasks × 200 connections each = 36,000 connections per node
- After scaling to 200 ECS tasks: 200 × 200 = 40,000 connections per node — still under limit
- BUT: the team plans to add a new analytics service that will add 30,000 connections per node
- Requirement: plan for the connection limit constraint before it causes failures

Which scaling strategy handles all three workloads correctly?

## Options

- **A.** Workload 1: ECS Service Auto Scaling with target tracking on a **custom CloudWatch metric** — `RequestCountPerTask` (ALB `RequestCount` ÷ ECS `RunningTaskCount`). Target value: 40 requests/task (80% of capacity). Configure a **step scaling policy** as a secondary policy with aggressive scaling steps: add 50 tasks when `RequestCountPerTask > 45`, add 100 tasks when `> 48`. Set ECS minimum task count to 50 during peak hours with a scheduled scaling action. Workload 2: DynamoDB **provisioned capacity** with Application Auto Scaling — target tracking on `ConsumedWriteCapacityUnits` at 70% utilization. Set the **maximum capacity** to 10,000 WCU. Before peak hours, use a **scheduled scaling action** to pre-warm the table to 3,000 WCU (so auto scaling starts from a higher base). Workload 3: Enable Redis **cluster mode** with 3 shards × 2 replicas. Add a CloudWatch alarm on `CurrConnections` at 80% of `maxclients` (52,000). Before launching the analytics service, scale to r6g.2xlarge (higher `maxclients` limit) or add shards to distribute connections.
- **B.** Workload 1: ECS scaling on CPU utilization at 70%. Workload 2: DynamoDB on-demand mode (no configuration needed). Workload 3: No action — 65,000 connections is sufficient.
- **C.** Workload 1: ECS scaling on ALB `ActiveConnectionCount` target tracking. Add 10 tasks at a time. Workload 2: DynamoDB provisioned with fixed 5,000 WCU 24/7. Workload 3: Add a second Redis cluster for the analytics service.
- **D.** Workload 1: Lambda functions behind API Gateway instead of ECS (serverless auto scaling). Workload 2: DynamoDB Global Tables for write distribution. Workload 3: Redis with read replicas and ElastiCache Auto Discovery.

## Answers

### A. Custom metric ECS scaling + DynamoDB provisioned with pre-warming + Redis cluster scaling — ✅ Correct

**Workload 1 — ECS Service Auto Scaling:**

- **ECS Service Auto Scaling** adjusts the `desiredCount` of tasks in an ECS service. It supports three scaling policy types:
  - **Target tracking**: Maintain a metric at a target value (e.g., keep CPU at 70%). ECS adds/removes tasks to maintain the target.
  - **Step scaling**: Add/remove specific numbers of tasks based on alarm thresholds (e.g., add 50 tasks when metric > threshold A, add 100 when > threshold B).
  - **Scheduled scaling**: Set minimum/maximum task counts at specific times (e.g., minimum 50 tasks at 5:30 PM before dinner rush).

- **Custom metric — `RequestCountPerTask`**:
  - The pre-built `ALBRequestCountPerTarget` metric is available for target tracking, but it measures requests per target registered in the ALB — which may differ from tasks if health checks are failing or tasks are draining.
  - A custom CloudWatch metric (`RequestCount ÷ RunningTaskCount`) provides a more accurate per-task load measure.
  - Target value of 40 (out of 50 capacity) gives 20% headroom — tasks aren't saturated before scaling kicks in.

- **Step scaling as secondary policy**:
  - Target tracking handles gradual increases smoothly. But during the dinner rush (traffic doubles in 5 minutes), target tracking may be too slow — it evaluates every 60 seconds and is conservative by design.
  - A step scaling policy adds aggressive scaling for sudden spikes: if `RequestCountPerTask > 45`, immediately add 50 tasks; if `> 48`, add 100 tasks. This ensures large-scale response to rapid traffic surges.
  - **Multiple scaling policies work together**: ECS evaluates all scaling policies and takes the action that results in the LARGEST change. Target tracking handles steady-state; step scaling handles spikes.

- **Scheduled scaling for pre-warming**:
  - At 5:30 PM (before the 6 PM dinner rush), a scheduled action sets `minCapacity = 50`. This ensures 50 tasks are already running when traffic ramps — eliminating the 45-second cold-start delay for the first batch of tasks.
  - After 9 PM, another scheduled action resets `minCapacity = 20`.

- **ECS Fargate task start time**: Fargate tasks take 30-60 seconds to start (image pull + container initialization). This latency means reactive-only scaling will always have a gap. Pre-warming with scheduled scaling + aggressive step scaling minimizes the impact.

**Workload 2 — DynamoDB Provisioned with Pre-warming:**

- **DynamoDB on-demand vs. provisioned scaling**:
  - **On-demand**: Automatically scales, but has a limit: it can handle up to **2x the previous peak** traffic within 30 minutes. If the table's previous peak was 3,000 WCU, on-demand can handle 6,000 WCU — but NOT instantly. It needs time to allocate partitions. A sudden jump from 500 to 5,000 WCU may cause throttling.
  - **Provisioned with auto scaling**: You set a target utilization (e.g., 70%) and min/max capacity. DynamoDB adjusts provisioned capacity based on actual consumption. The advantage: you control the maximum capacity and can pre-warm using scheduled scaling.

- **Pre-warming strategy**:
  - At 5 PM, a scheduled Application Auto Scaling action sets `MinCapacity = 3,000 WCU`. DynamoDB immediately provisions 3,000 WCU worth of partitions.
  - When traffic rises from 3,000 to 5,000 WCU, auto scaling increases capacity gradually — the table already has enough partitions to handle the ramp-up.
  - Without pre-warming: starting from 500 WCU, auto scaling needs 4-5 scaling actions (each taking 2-4 minutes) to reach 5,000 WCU. During this period, requests are throttled.

- **DynamoDB partition behavior**: DynamoDB distributes capacity across partitions (3,000 RCU or 1,000 WCU per partition). A table at 500 WCU has 1 partition. Scaling to 5,000 WCU requires 5 partitions — DynamoDB must split and redistribute data. Pre-warming gives DynamoDB time to allocate partitions before peak traffic arrives.

- **Maximum capacity guard**: Setting `MaxCapacity = 10,000 WCU` prevents runaway scaling (a bug or DDoS attack causing unlimited writes). This is a cost safety net.

**Workload 3 — Redis Connection Limit Planning:**

- **Redis `maxclients` limit**: Each Redis node type has a maximum client connection limit:
  - r6g.xlarge: 65,000 connections
  - r6g.2xlarge: 65,000 connections (same — `maxclients` is generally 65,000 for all node types, but available memory for connection buffers differs)
  - Actual usable connections depend on node memory (each connection consumes ~10 KB of memory for buffers)

- **Connection math**:
  - Current: 180 tasks × 200 connections = 36,000 per node (55% of limit) ✅
  - After ECS scaling: 200 tasks × 200 = 40,000 per node (62%) ✅
  - After analytics service: 40,000 + 30,000 = 70,000 per node — **exceeds 65,000 limit** ❌

- **Solution — horizontal scaling**:
  - Enable Redis cluster mode with 3 shards × 2 replicas per shard. Connections are distributed across shards. Each shard handles ~23,000 connections (70,000 ÷ 3 shards).
  - Alternative: use connection pooling in the analytics service (reduce 30,000 to 5,000 connections via a connection pool).
  - **CloudWatch alarm on `CurrConnections`**: Alert at 80% of `maxclients` (52,000) — gives the team warning before hitting the hard limit. This is a resource constraint that must be planned proactively, not discovered during an outage.

- **Key exam concept — resource constraints**: When evaluating scaling solutions, consider not just compute and database scaling but also: network connections, storage IOPS, DNS resolution limits, VPC IP addresses, ENI limits per subnet, and NAT Gateway bandwidth caps. The "correct" scaling answer must account for all resource constraints, not just the primary metric.

### B. CPU-based ECS scaling + on-demand DynamoDB + no Redis action — ❌ Incorrect

- **ECS CPU scaling**: CPU utilization is a lagging indicator — by the time CPU hits 70%, requests are already queuing and users experience latency. Request-based scaling (custom metric or `ALBRequestCountPerTarget`) is more responsive because it reacts to incoming demand, not resource saturation.
  - For Fargate tasks with mixed CPU/IO workloads, CPU may stay low (30%) while the application is overloaded with IO-bound requests. CPU-based scaling wouldn't trigger at all.

- **DynamoDB on-demand with no configuration**: On-demand mode works for many workloads, but it has a documented limitation: the 2x previous peak ceiling. For a table that's never seen 5,000 WCU, on-demand may throttle during the first spike. The question specifically states this is the current problem.

- **No Redis action**: Ignoring the connection limit is exactly how outages happen. When `CurrConnections` hits `maxclients`, all new connections are rejected — the analytics service AND the existing order API lose Redis connectivity simultaneously. Resource constraint planning is a core reliability skill.

### C. ALB connection-based ECS scaling + fixed DynamoDB capacity + separate Redis cluster — ❌ Incorrect

- **ALB `ActiveConnectionCount` scaling**: This metric counts active connections to the ALB, not connections per target. If you scale based on total connections, ECS adds tasks but each task may have very few connections (traffic isn't redistributed instantly). `RequestCountPerTarget` (or the custom per-task metric) is more accurate for scaling individual tasks.
  - Adding only 10 tasks at a time is too conservative for a 10x spike — it would take 18 scaling actions to go from 20 to 200 tasks.

- **Fixed 5,000 WCU 24/7**: Provisioning fixed capacity wastes money. At 500 WCU off-peak × 15 hours + 5,000 WCU peak × 9 hours: auto scaling costs ~60% less by reducing capacity during off-peak. Fixed capacity is a cost anti-pattern.

- **Separate Redis cluster for analytics**: Running two Redis clusters adds operational complexity — separate monitoring, failover handling, and data inconsistency (analytics reads stale data from a separate cache). Connection pooling or cluster mode scaling solves the connection problem without duplicating infrastructure.

### D. Lambda replacement + Global Tables + read replicas — ❌ Incorrect

- **Replacing ECS with Lambda**: Lambda auto-scales naturally, but this is a re-architecture decision, not a scaling solution. Migrating a running ECS application to Lambda requires: code changes (handler format), cold start management, 15-minute execution limits, 10 GB memory limits, and 6 MB synchronous payload limits. The question asks for scaling the existing ECS workload, not re-platforming.

- **DynamoDB Global Tables**: Global Tables replicate data across Regions for multi-Region availability — they don't solve single-Region throughput spikes. A table with 5,000 WCU in us-east-1 still needs 5,000 WCU in us-east-1 regardless of Global Tables. Global Tables are for DR and low-latency multi-Region reads, not for handling within-Region scaling.

- **ElastiCache Auto Discovery**: Auto Discovery helps clients find cluster nodes — it doesn't add capacity or solve connection limits. It's a service discovery feature, not a scaling feature.

## Recommendations

- **Combine multiple scaling policies**: Scheduled (pre-warm) + target tracking (steady-state) + step scaling (spikes). They work together — the action producing the largest change wins.
- **DynamoDB pre-warming**: For predictable spikes, use scheduled Application Auto Scaling actions to increase `MinCapacity` 30 minutes before peak. This ensures partitions are allocated before traffic arrives.
- **Monitor resource constraints proactively**: Create CloudWatch alarms for: Redis `CurrConnections`, EC2 network bandwidth, EBS IOPS utilization, NAT Gateway bytes processed, and VPC available IPs.
- **ECS scaling best practices**: Use `ALBRequestCountPerTarget` or custom request metrics. Avoid CPU-only scaling for IO-bound workloads. Set ECS deployment `minimumHealthyPercent` to 100 and `maximumPercent` to 200 for zero-downtime scaling.
- **DynamoDB on-demand considerations**: On-demand is excellent for unpredictable workloads. For predictable, spiky workloads where the spike exceeds 2x the recent peak, provisioned mode with auto scaling and pre-warming is more reliable.

## Relevant Links

- [ECS Service Auto Scaling](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html)
- [Application Auto Scaling Scheduled Actions](https://docs.aws.amazon.com/autoscaling/application/userguide/application-auto-scaling-scheduled-scaling.html)
- [DynamoDB Auto Scaling](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/AutoScaling.html)
- [DynamoDB On-Demand Scaling Behavior](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html#HowItWorks.OnDemand)
- [ElastiCache Redis Connection Limits](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/redis-metrics-dimensions.html)
