# Q02: Spot Instance Strategies for Fault-Tolerant Workloads

## Question

A data analytics company runs three batch workloads that are candidates for Spot Instances:

**Workload A — Nightly ETL Pipeline**
- 50 m5.2xlarge instances processing 10 TB of data, running 6 hours (midnight–6 AM)
- Jobs are split into 500 independent tasks, each processing ~20 GB
- Tasks can be retried on failure — processing is idempotent
- If interrupted, a task restarts from the beginning (no checkpointing) — each task takes ~35 minutes
- Current On-Demand cost: $1,150/night

**Workload B — Continuous ML Model Training**
- 10 p3.2xlarge GPU instances running 24/7
- Training uses distributed data parallelism (all 10 instances work on the same model)
- Losing even 1 instance requires restarting the current epoch (15 minutes of work lost)
- Models checkpoint every 30 minutes to S3
- Current On-Demand cost: $73,000/month

**Workload C — Web Scraping Fleet**
- 200 t3.medium instances running 24/7
- Each instance runs an independent scraping job
- Losing instances only reduces throughput — no data loss
- Current On-Demand cost: $12,000/month

Which Spot Instance strategy is optimal for each workload?

## Options

- **A.** Workload A: Spot Fleet with capacity-optimized allocation strategy, diversified across m5.2xlarge, m5d.2xlarge, m5a.2xlarge, m5n.2xlarge, and m4.2xlarge. Use SQS queue to distribute tasks — instances pull tasks from the queue; interrupted instances' tasks return to the queue for retry. Workload B: Mixed strategy — 2 On-Demand p3.2xlarge (minimum for training continuity) + 8 Spot p3.2xlarge with capacity-optimized allocation. Checkpoint every 30 minutes. If Spot interruptions exceed 2 instances, scale down to On-Demand only and resume with smaller data parallelism. Workload C: Spot Fleet with lowest-price allocation across t3.medium, t3a.medium, t2.medium. Use EC2 Auto Scaling to maintain desired capacity — interrupted instances are automatically replaced.
- **B.** All three: Use Spot Fleet with lowest-price allocation strategy and a single instance type per workload. No On-Demand fallback.
- **C.** Workload A: On-Demand (ETL is time-sensitive — can't afford restarts). Workload B: All Spot (GPU savings are too significant to pass up). Workload C: Reserved Instances (24/7 workload → commitment makes sense).
- **D.** All three: Use EC2 Auto Scaling with mixed instances policy — 50% On-Demand base + 50% Spot. Same instance types for all. Use Spot instance interruption handler to drain containers on 2-minute notice.

## Answers

### A. Diversified Spot Fleet + SQS (A) + On-Demand base + Spot (B) + Spot with auto-replacement (C) — ✅ Correct

Each workload gets a Spot strategy tailored to its fault tolerance and instance requirements:

- **Workload A — Diversified Spot Fleet + SQS Queue**:
  - **Capacity-optimized allocation**: Spot Fleet selects instance pools with the most available capacity at the time of launch — reducing interruption frequency. This is preferred over lowest-price, which always launches in the cheapest pool (which may have the most competition and highest interruption rate).
  - **Instance type diversification**: By including m5, m5d, m5a, m5n, and m4 families (all 2xlarge — similar CPU/memory), the Spot Fleet has 5 pools to choose from. If one pool's price spikes or capacity drops, others absorb the demand.
  - **SQS-based task distribution**: Instances poll a queue for tasks. If an instance is interrupted mid-task, the message returns to the queue (visibility timeout expires) and another instance picks it up. No coordination logic needed — SQS handles retry natively.
  - **35-minute tasks without checkpointing**: Worst case, an interruption wastes 35 minutes of one task. With 500 independent tasks across 50 instances, a few retries add < 30 minutes to the total 6-hour pipeline. Acceptable.
  - **Cost savings**: Spot m5.2xlarge ≈ $0.12/hr vs. On-Demand $0.384/hr (~70% savings). 50 instances × 6 hours × $0.12 = $36/night vs. $1,150 On-Demand. **Annual savings: ~$400K.**

- **Workload B — On-Demand base + Spot scaling for GPU training**:
  - **Why not 100% Spot**: Distributed data parallelism requires all instances to synchronize. If a Spot interruption takes out 1 of 10 workers, the remaining 9 must wait or restart the epoch. Frequent interruptions reduce effective training throughput to below what 2 On-Demand instances could achieve alone.
  - **2 On-Demand minimum**: Guarantees training continues even during Spot capacity shortages. The model can train on 2 instances (slower, smaller batch) without interruption.
  - **8 Spot instances**: When available, 10 total instances provide maximum training speed. p3.2xlarge Spot ≈ $1.00/hr vs. $3.06/hr On-Demand (67% savings).
  - **Cost estimate**: 2 On-Demand ($6.12/hr) + 8 Spot ($8.00/hr) = $14.12/hr vs. $30.60/hr all On-Demand. **Monthly savings: ~$11,800 (38% reduction).**
  - **30-minute checkpoints**: On Spot interruption, at most 30 minutes of training is lost. Resume from the latest checkpoint on replacement instances.
  - **Capacity-optimized allocation for GPU**: GPU instances have constrained supply. Capacity-optimized strategy selects the pool most likely to have available p3 capacity.

- **Workload C — Spot Fleet with lowest-price + auto-replacement**:
  - **Independent scrapers**: Each instance works independently — losing instances only reduces throughput, not correctness. This is the ideal Spot use case.
  - **Lowest-price allocation**: For 200 small instances (t3.medium at $0.042/hr On-Demand), lowest-price selects the cheapest available pool. t3.medium, t3a.medium (AMD), and t2.medium provide 3 pools. The price difference between pools is small, so interruption risk is acceptable given the workload's high fault tolerance.
  - **Auto Scaling replacement**: EC2 Auto Scaling maintains the desired count. When instances are interrupted, ASG launches replacements within seconds. The fleet self-heals.
  - **Cost savings**: Spot t3.medium ≈ $0.013/hr (70% savings). 200 × $0.013 × 730 hours = $1,898/month vs. $12,000 On-Demand. **Monthly savings: ~$10,000.**

- **Total annual savings across all three**: ~$400K (A) + ~$142K (B) + ~$120K (C) = **~$662K/year.**

### B. Single instance type + lowest-price for all — ❌ Incorrect

- **Single instance type**: No diversification means the entire fleet competes for one Spot pool. If that pool's price spikes or capacity drops, ALL instances are interrupted simultaneously.
- **Lowest-price allocation for GPU**: The lowest-price p3.2xlarge pool has the most demand (everyone wants the cheapest GPUs). This pool has the highest interruption rate. Capacity-optimized selects the pool with the most available capacity — better for constrained instance types.
- **No On-Demand fallback for ML training**: 100% Spot for distributed training means any interruption pauses the entire training run. During Spot capacity shortages (common for GPU instances), training stops entirely — potentially for hours. On-Demand base ensures minimum training continuity.

### C. On-Demand (A) + All Spot (B) + RI (C) — ❌ Incorrect

- **On-Demand for ETL**: The ETL pipeline is the BEST Spot candidate — 500 independent, idempotent tasks with natural retry via SQS. Staying on On-Demand wastes $400K/year.
- **All Spot for ML training**: As explained, distributed data parallelism requires instance coordination. Frequent Spot interruptions cause epoch restarts, reducing effective throughput. A mixed strategy (On-Demand base + Spot) provides the best cost-performance tradeoff.
- **Reserved Instances for web scrapers**: 200 t3.medium RI (1-year) saves ~30-40% vs. On-Demand. Spot saves 70%. RI provides $4,800/month; Spot provides $10,000/month in savings. RI also locks in 200 instances for 1 year — if scraping needs decrease, the RI is wasted.

### D. 50/50 On-Demand/Spot for all — ❌ Incorrect

- **50/50 split for ETL**: ETL tasks are independently retryable — 50% On-Demand is unnecessarily conservative. 100% Spot with SQS retry is safe and maximizes savings.
- **50/50 for scrapers**: Same reasoning — scrapers are perfectly fault-tolerant. 100% Spot is appropriate.
- **Same instance types for all**: GPU workloads (p3) and CPU workloads (m5, t3) cannot share instance types. This generic approach doesn't account for workload-specific requirements.
- **Container draining on 2-minute notice**: Relevant for containerized workloads (ECS, EKS) but ETL tasks on SQS and independent scrapers don't need graceful draining — they simply retry.

## Recommendations

- **Capacity-optimized allocation** is recommended for most Spot workloads — it reduces interruption frequency by selecting the most available pools. Use lowest-price only for highly fault-tolerant workloads where you want maximum savings and accept higher interruption rates.
- **Diversify across at least 4-6 instance types** — same vCPU/memory, different families (m5, m5a, m5n, m4, m5d). This creates multiple Spot pools for the fleet to draw from.
- **SQS as a work queue** is the standard pattern for Spot batch processing — tasks self-retry on interruption with zero custom logic.
- **On-Demand base + Spot scaling** for coordinated workloads (distributed ML, stateful applications) — ensures minimum functionality during Spot capacity shortages.
- **Monitor Spot interruption frequency** using CloudWatch metrics and EC2 Spot Instance Advisor to choose instance types with < 10% interruption frequency.
- **Use EC2 Instance Metadata Service** to detect the 2-minute interruption notice and perform graceful shutdown (flush buffers, save state, deregister from target group).

## Relevant Links

- [Spot Fleet Allocation Strategies](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet-allocation-strategy.html)
- [Spot Instance Advisor](https://aws.amazon.com/ec2/spot/instance-advisor/)
- [EC2 Auto Scaling Mixed Instances Policy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-mixed-instances-groups.html)
- [Spot Instance Interruption Handling](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-instance-termination-notices.html)
- [Spot Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)
