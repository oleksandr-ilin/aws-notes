# Q02: Scaling Strategy Improvement and Service Quota Management

## Question

An e-commerce platform experiences predictable traffic patterns (3× surge during daily sales events, 10× during annual peak events) and steady growth (40% year-over-year). Current architecture:

| Component | Current Configuration | Observed Problem |
|---|---|---|
| EC2 ASG | Target tracking (CPU 70%), min=4, max=20 | **Scale-out too slow**: Traffic spikes in < 2 minutes. ASG takes 5-7 minutes to add capacity (launch + boot + ALB health check). 3-5 minutes of degraded performance during each spike. |
| ALB | Default settings | No issues |
| Aurora MySQL | db.r6g.2xlarge, 2 read replicas | **Read replica overwhelm**: During peak, read replicas max CPU. Adding replicas takes 10-15 minutes. By then, the spike is over. |
| DynamoDB | On-demand capacity mode | **Throttling during sudden spikes**: On-demand adapts quickly, BUT if traffic doubles from previous peak instantly, DynamoDB throttles for 5-30 minutes while scaling (on-demand can instantly handle up to 2× the previous peak, but beyond that, it throttles). |
| Lambda functions | 1,000 concurrent burst limit (account default) | **Throttling**: During peak sales, Lambda concurrent executions hit 1,000 within 2 minutes. New invocations return `TooManyRequestsException`. Image processing and notification functions fail. |
| NAT Gateway | Single NAT GW in one AZ | **Bottleneck**: 45 Gbps bandwidth limit shared by all outbound traffic. During peak, outbound API calls to payment processors time out. |

The architecture team needs to: (a) eliminate scaling delays, (b) prevent service quota throttling, and (c) plan for 40% annual growth.

What changes address all scaling issues?

## Options

- **A.** **EC2 scaling**: Replace target tracking with **scheduled scaling** + **predictive scaling**. Scheduled: pre-scale to min=12 at 09:00 (30 minutes before daily sales events). Predictive: enable predictive scaling (ML-based) to learn the daily pattern and pre-provision capacity. Keep target tracking as a reactive backup (catches unexpected spikes). Use **warm pools** — keep 8 pre-initialized instances in `Stopped` state. ASG pulls from warm pool instead of launching new instances: start time = 30-60 seconds instead of 5-7 minutes. **Aurora scaling**: Add **Aurora Auto Scaling** for read replicas (min=2, max=8, target CPU 60%). Aurora replicas launch from the shared storage volume — no data copy needed — launch time is 5-7 minutes (faster than standard RDS). For sub-minute read scaling, add **ElastiCache** in front of Aurora for the hottest queries. **DynamoDB**: Switch from on-demand to **provisioned capacity with auto scaling** for the base tables. Set provisioned WCU/RCU to the expected peak (handle daily spikes without throttling). Configure auto scaling (target utilization 70%, min/max WCU/RCU). For the annual 10× event: **pre-warm by requesting a capacity increase** via AWS support before the event, or **use reserved capacity** for baseline + on-demand for burst. **Lambda quotas**: Request a **concurrency quota increase** from 1,000 to 10,000 (Service Quotas console). Set **reserved concurrency** on critical functions: payment processing = 3,000, notification = 2,000, image processing = 1,000. Configure **provisioned concurrency** on the payment processing function (500 units) to eliminate cold starts. **NAT Gateway**: Deploy **one NAT Gateway per AZ** (3 AZs = 3 NAT GWs). Each AZ's route table points to its own NAT GW. Bandwidth scales to 3 × 45 Gbps = 135 Gbps aggregate. For payment processor API calls: move to **VPC endpoints** or **PrivateLink** if the payment processor is in AWS (eliminates NAT Gateway entirely for that traffic). **Growth planning**: Implement **AWS Service Quotas dashboard** monitoring. Create CloudWatch alarms at 80% of each quota. Review quotas quarterly and request increases proactively (some increases take 1-3 weeks for approval).
- **B.** Replace EC2 with Lambda for all compute (serverless scales automatically). Replace Aurora with DynamoDB Global Tables for automatic scaling. Use API Gateway with throttling to protect backend services.
- **C.** Over-provision all resources to 10× current peak: EC2 min=40, Aurora 8 read replicas permanently, DynamoDB provisioned at 10× current usage. This eliminates scaling delays by always having capacity.
- **D.** Implement application-level rate limiting and queuing. Use SQS to buffer all requests during spikes. Process requests at a constant rate regardless of incoming traffic. Users see "request queued" messages during peaks.

## Answers

### A. Predictive scaling + warm pools + quota management — ✅ Correct

**EC2 — Eliminating the 5-7 Minute Scale-Out Delay:**

- **The problem**: Standard ASG reactive scaling → detect metric breach (1-2 min) → API call to launch (30s) → instance launch (2 min) → boot + app startup (2 min) → ALB health check pass (30s) = 5-7 minutes total. For a traffic spike that arrives in < 2 minutes, the application is degraded for 3-5 minutes.

- **Solution 1 — Scheduled Scaling (for predictable events)**:
  - Daily sales events at 09:30 → schedule ASG to scale to min=12 at 09:00 (30 min buffer)
  - Annual peak event → schedule min=30 one hour before the event starts
  - `aws autoscaling put-scheduled-update-group-action --scheduled-action-name pre-sale --recurrence "0 9 * * *" --min-size 12`
  - The instances are running and healthy BEFORE traffic arrives — zero scaling delay

- **Solution 2 — Predictive Scaling (ML-based)**:
  - AWS analyzes 14 days of ASG historical data and forecasts future capacity needs
  - Creates scheduled scaling actions automatically based on the predicted pattern
  - Mode: `ForecastAndScale` — both predicts and acts (vs. `ForecastOnly` for monitoring)
  - Works alongside target tracking: predictive handles the expected pattern, target tracking handles unexpected deviations
  - Requires 24 hours of data to begin predictions, 14 days for full accuracy

- **Solution 3 — Warm Pools (for reactive scaling speed)**:
  - Pre-initialized instances kept in `Stopped` state (no compute charges — only EBS storage charges)
  - When ASG needs to scale out: starts a warm pool instance (30-60 seconds) instead of launching a new one (5-7 minutes)
  - Warm pool lifecycle: `Pending → Warmed:Stopped → InService` (vs. normal: `Pending → InService`)
  - Set warm pool size = max - desired (e.g., max=20, desired=4 → warm pool size=16)
  - **Cost**: 16 stopped instances × EBS volume cost only = ~$25/month (vs. running = ~$2,000/month). 99% cheaper than over-provisioning

- **Combined strategy**:

  | Trigger | Mechanism | Response Time |
  |---|---|---|
  | Daily sales event (predictable) | Scheduled scaling | 0 minutes (pre-provisioned) |
  | Daily pattern variations | Predictive scaling | 0-5 minutes (predicted) |
  | Unexpected traffic spike | Target tracking + warm pool | 1-2 minutes (warm start) |
  | Extreme unexpected spike | Target tracking + cold launch | 5-7 minutes (last resort) |

**Aurora — Faster Read Scaling:**

- **Aurora Auto Scaling for read replicas**: Aurora read replicas share the same storage volume as the primary (no data replication needed). This means replica creation is faster than standard RDS (~5-7 min vs. 15-20 min for RDS). Aurora Auto Scaling policies scale replicas based on CPU or connections.

- **ElastiCache for sub-minute read scaling**: Even 5-7 minutes is too slow for a spike that arrives in 2 minutes. For the hottest queries (product catalog, pricing, inventory counts), cache results in ElastiCache Redis. Cache hit ratio > 80% reduces Aurora read replica load by 80% — the replicas that exist can handle the remaining 20%.

- **Aurora Serverless v2 alternative**: If the workload is truly spiky, consider Aurora Serverless v2. It scales in increments of 0.5 ACU (Aurora Capacity Unit) and can scale from 0.5 to 256 ACUs in seconds. No replica management needed — capacity adjusts automatically.

**DynamoDB — Avoiding On-Demand Throttling:**

- **On-demand throttling explained**: DynamoDB on-demand can instantly serve up to **2× the previous peak** traffic. If the previous peak was 5,000 WCU and traffic suddenly jumps to 15,000 WCU (3× previous), DynamoDB throttles until it adapts (5-30 minutes).

- **Provisioned + Auto Scaling** for predictable baselines:
  - Set provisioned capacity to the expected daily peak (e.g., 10,000 WCU)
  - Auto scaling adjusts up/down based on utilization (target: 70%)
  - For the annual 10× event: use `UpdateTable` to pre-increase provisioned capacity 1 hour before the event

- **Pre-warming for large events**:
  - DynamoDB provisioned capacity can scale up instantly up to the table's current capacity
  - For very large increases (10× or more): increase capacity gradually over several days, or contact AWS support to pre-warm the underlying partitions
  - Alternative: Use on-demand for the event period (switch to on-demand 24 hours before — DynamoDB handles the transition with zero downtime)

**Lambda — Quota Management:**

- **Concurrency quota increase**: The default 1,000 concurrent execution limit is a soft limit. Request an increase via the Service Quotas console:
  - Navigate to Service Quotas → Lambda → Concurrent executions → Request quota increase
  - Most increases to 10,000 are approved within 1-3 days
  - For very high limits (50,000+), AWS may require an architecture review

- **Reserved concurrency**: Guarantees a portion of the account's concurrency for a specific function:
  - `payment-processing`: 3,000 reserved → always has 3,000 concurrent executions available, even if other functions use the rest of the account pool
  - **Caution**: Reserved concurrency for one function REDUCES available concurrency for all other functions. If account limit is 10,000 and you reserve 6,000 across functions, only 4,000 remain for unreserved functions

- **Provisioned concurrency**: Pre-initializes function execution environments (eliminates cold starts):
  - Payment processing function with provisioned concurrency = 500: 500 execution environments are always warm, ready to handle requests in < 10 ms
  - Beyond 500 concurrent: Lambda uses standard cold starts for additional invocations
  - **Cost**: ~$0.015/GB-hour for provisioned environments. 500 × 512 MB × 24 hours = ~$90/day. Use Application Auto Scaling to schedule provisioned concurrency (high during business hours, low at night)

**NAT Gateway — Bandwidth Scaling:**

- **One NAT Gateway per AZ** is AWS best practice:
  - Each AZ's private subnets route through the AZ-local NAT GW
  - If one NAT GW fails, only that AZ's outbound traffic is affected (not all AZs)
  - Bandwidth: each NAT GW supports 45 Gbps. 3 AZs × 45 Gbps = 135 Gbps aggregate
  - Also improves AZ independence — cross-AZ NAT GW traffic incurs data transfer charges ($0.01/GB). AZ-local NAT GW eliminates this

- **VPC endpoints for AWS services**: If outbound traffic includes S3, DynamoDB, or other AWS service calls, use **gateway VPC endpoints** (S3, DynamoDB — free) or **interface VPC endpoints** (other services — $0.01/GB) to route traffic directly without NAT GW. This reduces NAT GW bandwidth consumption and cost.

**Service Quota Planning for Growth:**

- **AWS Service Quotas** dashboard: Centralized view of all service quotas and current utilization
- CloudWatch metrics: `AWS/Usage` namespace publishes utilization percentages for many quotas
- Alarm: `ResourceCount / ServiceQuota > 0.8` → alert at 80% of quota
- Quarterly review cadence: Check quotas against projected growth. If 40% YoY growth, quotas hit in ~2 years at current limits. Request increases proactively.

### B. All-serverless migration — ❌ Incorrect

- **Lambda for all compute**: Not all workloads suit Lambda. Long-running processes (> 15-minute timeout), stateful applications, WebSocket connections, and high-throughput data processing may not work with Lambda. Migration effort is massive — the question asks for improvements to the existing architecture, not a rewrite.
- **DynamoDB Global Tables**: Replacing Aurora MySQL with DynamoDB requires a complete data model redesign (relational → key-value/document). This is a multi-month project, not a scaling improvement.
- **API Gateway throttling**: Throttling protects backends but doesn't add capacity. During a sales event, throttling means rejecting real customer requests — directly impacting revenue.

### C. Over-provision to 10× peak — ❌ Incorrect

- **Cost**: 40 EC2 instances running 24/7 = ~$14,000/month (vs. 4 baseline instances = $1,400/month). 10× infrastructure cost for capacity used 0.1% of the time.
- **DynamoDB provisioned at 10× is extremely expensive**: Provisioned WCU/RCU charges are per-hour. 10× capacity 24/7 means paying for 10× peak even during 2 AM when traffic is 10% of peak.
- **8 Aurora read replicas permanently**: Each replica costs the same as an additional Aurora instance (~$700/month for r6g.2xlarge). 8 replicas = $5,600/month, needed only during peaks.
- **Not sustainable**: 40% annual growth means 10× today is barely sufficient in 2.5 years. The approach requires repeated over-provisioning reviews.
- **Warm pools achieve the same effect at 1% cost**: Stopped instances in warm pool provide instant capacity at EBS storage cost only.

### D. Queue everything — ❌ Incorrect

- **Terrible user experience**: Telling customers "your request is queued" during a sales event means lost sales. E-commerce requires real-time responses — customers won't wait for a "your order will be processed in 10 minutes" message.
- **Rate limiting legitimate traffic**: Fixed-rate processing limits throughput by design. The goal is to INCREASE capacity to match demand — not to limit demand to match capacity.
- **SQS adds complexity without solving scaling**: SQS is useful for decoupling and handling bursts in background processing (image processing, notifications). It's not appropriate for user-facing request/response flows where the user expects an immediate result.

## Recommendations

- **Layer scaling strategies**: Scheduled (predictable) + Predictive (ML-learned patterns) + Reactive (target tracking/step scaling) + Pre-provisioned (warm pools/provisioned concurrency). Each layer catches what the previous one missed.
- **Pre-game for known events**: Before any planned high-traffic event: pre-scale EC2 (scheduled), pre-warm DynamoDB (increase provisioned capacity or switch to on-demand), verify Lambda concurrency quota, and test the full load path.
- **Monitor quotas, not just utilization**: A service at 5% CPU but 95% of its concurrency quota will fail on the next spike. Create Service Quotas dashboards alongside performance dashboards.
- **NAT Gateway per AZ is always the right answer**: Single NAT GW is a single point of failure AND a bandwidth bottleneck. One per AZ is an AWS Well-Architected best practice.
- **Growth planning**: With 40% YoY growth, capacity that works today hits limits in 18-24 months. Review service quotas, instance family availability, and Regional capacity quarterly.

## Relevant Links

- [Predictive Scaling for ASG](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-predictive-scaling.html)
- [ASG Warm Pools](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-warm-pools.html)
- [Aurora Auto Scaling](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Integrating.AutoScaling.html)
- [DynamoDB Auto Scaling](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/AutoScaling.html)
- [Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)
- [Service Quotas](https://docs.aws.amazon.com/servicequotas/latest/userguide/intro.html)
- [NAT Gateway Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
