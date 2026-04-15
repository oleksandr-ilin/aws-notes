# Q02: Elastic Architecture with Auto Scaling and Storage Performance

## Question

A healthcare SaaS platform processes patient records and medical imaging. The architecture uses:
- ALB → ECS Fargate tasks (API tier) → Aurora PostgreSQL (transactional data)
- S3 → Lambda (image preprocessing) → ECS Fargate (ML inference) → results to DynamoDB
- The imaging pipeline processes MRI/CT scans uploaded by hospitals between 6 AM–10 AM daily (batch upload window)

Current problems:
1. During the 6–10 AM batch window, image processing latency increases from 30 seconds to 8 minutes per scan due to cold-start delays and insufficient concurrency
2. The API tier ECS tasks scale too slowly — CloudWatch CPU alarms trigger at 70% but new tasks take 3 minutes to start, causing request queuing
3. Aurora writer instance (db.r6g.4xlarge) hits 90% CPU during batch window; read replicas are underutilized at 15% CPU

The team needs to reduce imaging latency to under 60 seconds and API response time to under 500 ms during peak, while keeping costs optimized during the 18-hour off-peak period.

## Options

- **A.** (1) Use Lambda provisioned concurrency (scheduled via Application Auto Scaling) for the preprocessing function, ramping to 500 concurrent executions at 5:45 AM and scaling down at 10:15 AM. (2) Switch API tier to ECS Service Auto Scaling with target tracking on ALBRequestCountPerTarget (target: 1,000 req/target) and set a minimum task count of 4 during peak (scheduled scaling). (3) Move write-heavy batch queries to an Aurora parallel query cluster, and route read traffic to Aurora reader endpoint using the application's connection string.
- **B.** (1) Replace Lambda with a dedicated ECS Fargate task pool for preprocessing with step scaling. (2) Switch to EC2 launch type with custom AMI for API tier (pre-baked containers, faster boot). (3) Scale Aurora vertically to db.r6g.16xlarge to handle batch CPU load.
- **C.** (1) Use Lambda provisioned concurrency set to fixed 1,000 concurrent executions 24/7. (2) Switch API tier to EKS on EC2 with Karpenter for node provisioning. (3) Replace Aurora with DynamoDB for the transactional database.
- **D.** (1) Use S3 batch operations to throttle image uploads to a steady rate instead of bursts. (2) Keep current API tier scaling but reduce CloudWatch alarm period from 5 minutes to 1 minute. (3) Add an Aurora Serverless v2 reader for batch query offloading.

## Answers

### A. Lambda provisioned concurrency (scheduled) + ECS target tracking + Aurora read/write splitting — ✅ Correct

This addresses each problem with the right scaling mechanism:

- **Lambda provisioned concurrency (scheduled)**:
  - **Problem solved**: Cold starts cause Lambda containers to initialize (download code, initialize runtime, create connections) — adding 5-15 seconds per cold invocation. At burst scale, hundreds of simultaneous cold starts compound into minutes of queuing.
  - **Solution**: Provisioned concurrency pre-initializes Lambda execution environments. Application Auto Scaling schedules ramp-up to 500 at 5:45 AM (15 min before batch window) — all environments are warm when uploads begin. Scale-down at 10:15 AM releases resources after the window.
  - **Cost optimization**: Provisioned concurrency only during the 4.5-hour window (5:45 AM–10:15 AM) instead of 24/7 — pays for ~19% of the day. The off-peak 18 hours use standard on-demand Lambda (with proportionally fewer cold starts at low volume).

- **ECS Service Auto Scaling with target tracking on ALBRequestCountPerTarget**:
  - **Problem solved**: CPU-based alarms trigger too late (CPU must reach 70%) and scaling is reactive. By the time new tasks start (3 min), requests are already queuing.
  - **Solution**: Target tracking on request count per target is proactive — it scales based on incoming load, not lagging CPU metrics. If the target is 1,000 requests per target and traffic is rising, ECS adds tasks BEFORE CPU spikes. Target tracking also uses predictive pre-scaling.
  - **Scheduled minimum**: Setting minimum task count to 4 during 6–10 AM ensures a baseline is always running — no scale-from-zero delay. Off-peak minimum can be 1-2 tasks.

- **Aurora read/write endpoint separation**:
  - **Problem solved**: Writer at 90% CPU while readers at 15% — batch queries are hitting the writer unnecessarily.
  - **Solution**: Route read queries (SELECT) to the reader endpoint. Batch analytics queries (which are likely read-heavy aggregations on patient records) offload from the writer, immediately reducing writer CPU. Aurora parallel query further accelerates analytical queries by pushing processing to the storage layer.

### B. ECS preprocessing + EC2 API tier + Aurora vertical scaling — ❌ Incorrect

- **ECS Fargate for preprocessing instead of Lambda**: ECS Fargate tasks take 30-60 seconds to start (image pull + container initialization). Lambda cold starts are 5-15 seconds, and provisioned concurrency eliminates them entirely. Moving to Fargate makes the cold-start problem worse, not better.
- **EC2 launch type with custom AMI**: Pre-baked AMIs do boot faster (~30-60 seconds), but introduce infrastructure management burden (patch AMIs, manage ASGs, handle instance failures). Fargate is serverless — the scaling problem is about the scaling policy, not the container platform.
- **Vertical scaling to db.r6g.16xlarge**: Costs 4× more per hour than db.r6g.4xlarge. The read replicas are at 15% CPU — this capacity is wasted. The correct fix is to use the existing replicas by routing reads to the reader endpoint, not buying a bigger writer.

### C. 24/7 provisioned concurrency + EKS migration + DynamoDB — ❌ Incorrect

- **1,000 provisioned concurrency 24/7**: Provisioned concurrency costs ~$0.015/GB-hour. At 1,000 × 1 GB × 24 hours × 30 days = ~$10,800/month — for a workload that needs this capacity only 4 hours/day. Scheduled provisioned concurrency at 500 for 4.5 hours ≈ ~$1,012/month. Over 10× cost difference.
- **EKS on EC2 with Karpenter**: Massive migration (ECS → EKS) to solve a scaling policy problem. Karpenter is excellent for node provisioning but doesn't help with application-level scaling delays — the pod still needs to start, and Karpenter's node provisioning adds time. This is over-engineering.
- **Replace Aurora with DynamoDB**: The transactional database uses complex joins on patient records — relational access patterns. DynamoDB doesn't support joins. This is a fundamental architecture mismatch.

### D. Throttle uploads + faster alarms + Aurora Serverless reader — ❌ Incorrect

- **Throttle uploads to steady rate**: This degrades the user experience — hospitals upload in batches during a limited window. Spreading uploads artificially delays processing and may miss clinical deadlines. The platform should scale to meet demand, not throttle demand to match the platform.
- **Reduce CloudWatch alarm period**: Changing from 5 minutes to 1 minute detects CPU spikes faster but the fundamental problem remains — tasks take 3 minutes to start. Faster detection + same startup time = marginal improvement. Target tracking (request-based, proactive) is fundamentally better than threshold alarms (CPU-based, reactive).
- **Aurora Serverless v2 reader**: This is partially valid for offloading reads, but Serverless v2 cold-starts (ACU ramp-up) introduce latency during the first minutes of the batch window. A provisioned reader at known capacity is more predictable for a known daily pattern.

## Recommendations

- **Lambda provisioned concurrency** eliminates cold starts completely — use it for latency-sensitive functions with predictable traffic patterns. Schedule it with Application Auto Scaling for cost optimization.
- **ECS target tracking on ALBRequestCountPerTarget** is the recommended scaling metric for web services — it's proactive and directly correlates with user-visible load (unlike CPU, which lags).
- **Aurora reader endpoints** distribute reads across replicas with connection-level load balancing. The application must use separate connection strings for reads vs. writes.
- **Scheduled scaling** should "lead" the traffic pattern by 10-15 minutes — pre-warm capacity before demand arrives.
- **Never throttle user traffic** to avoid scaling challenges — scale the infrastructure to meet demand.

## Relevant Links

- [Lambda Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)
- [Application Auto Scaling Scheduled Actions](https://docs.aws.amazon.com/autoscaling/application/userguide/application-auto-scaling-scheduled-scaling.html)
- [ECS Service Auto Scaling](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html)
- [Aurora Read Scaling with Reader Endpoints](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html)
- [Aurora Parallel Query](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-mysql-parallel-query.html)
