# Q01: Instance Family Selection and Right-Sizing for Mixed Workloads

## Question

A media company runs three workloads on AWS that were originally deployed on m5.2xlarge instances (8 vCPU, 32 GiB RAM) as a one-size-fits-all approach. After 3 months of production, CloudWatch and Compute Optimizer data shows:

**Workload A — Video Transcoding Service (batch, non-time-critical)**
- Average CPU utilization: 95%, Memory: 18%
- Can tolerate interruptions (jobs resume from checkpoints stored in S3)
- Runs 12 hours/day during off-peak, processing next-day content

**Workload B — In-Memory Analytics Engine**
- Average CPU utilization: 22%, Memory: 92%
- Runs 24/7, performs real-time aggregation on a 500 GB dataset held in memory
- Cannot tolerate interruptions — financial dashboards depend on it

**Workload C — Web Application Servers (Auto Scaling group)**
- Average CPU: 35%, Memory: 40%
- Traffic follows a daily pattern: 2× peak from 9 AM–6 PM, baseline overnight
- P99 latency target: 100 ms, currently at 85 ms

The team wants to optimize cost without degrading performance. Which combination of instance changes yields the most cost savings?

## Options

- **A.** Workload A: Switch to c6g.2xlarge (Graviton, compute-optimized) Spot Instances with Spot Fleet capacity-optimized allocation. Workload B: Switch to r6g.8xlarge (Graviton, memory-optimized) with a 1-year Compute Savings Plan. Workload C: Switch to m6g.xlarge (Graviton, half-sized) with a mix of On-Demand baseline + Spot for scaling.
- **B.** Workload A: Switch to c5.4xlarge (compute-optimized) On-Demand. Workload B: Switch to r5.4xlarge Reserved Instance (3-year, All Upfront). Workload C: Keep m5.2xlarge, add scheduled scaling.
- **C.** Workload A: Switch to p4d.24xlarge (GPU) Spot Instances. Workload B: Switch to x2idn.16xlarge (memory-optimized, 1 TB RAM). Workload C: Switch to t3.xlarge burstable instances.
- **D.** All three workloads: Switch to Graviton m7g.xlarge instances. Apply a 3-year EC2 Instance Savings Plan for m7g in the current Region. Use Auto Scaling for all three.

## Answers

### A. Graviton C6g Spot (A) + Graviton R6g Savings Plan (B) + Graviton M6g half-size mixed (C) — ✅ Correct

Each workload is right-sized based on its actual resource utilization and availability requirements:

- **Workload A → c6g.2xlarge Spot Instances**:
  - **Compute-optimized (C-family)**: CPU is the bottleneck (95%), memory is wasted (18% of 32 GiB = ~6 GiB used). C6g.2xlarge provides 8 vCPU + 16 GiB — sufficient CPU with right-sized memory.
  - **Graviton ("g" suffix)**: Up to 40% better price-performance than x86 C5. Video transcoding libraries (FFmpeg, MediaConvert) support ARM/Graviton.
  - **Spot Instances**: Workload tolerates interruptions (checkpoint/resume). Spot provides up to 90% discount over On-Demand. Capacity-optimized allocation strategy selects pools with the most available capacity, reducing interruption frequency.
  - **Combined savings**: ~40% (Graviton) + ~70% (Spot) ≈ ~80% cost reduction vs. m5.2xlarge On-Demand.

- **Workload B → r6g.8xlarge with Compute Savings Plan**:
  - **Memory-optimized (R-family)**: Memory is the bottleneck (92%), CPU is underutilized. R6g.8xlarge provides 32 vCPU + 256 GiB — for 500 GB dataset, 2× r6g.8xlarge or 1× r6g.16xlarge (512 GiB). Right-sizes to memory needs.
  - **Graviton**: Lower cost per GiB of RAM. The analytics engine must be compatible with ARM — if it is, Graviton provides ~20% cost savings.
  - **Compute Savings Plan (1-year)**: This workload runs 24/7 and cannot be interrupted — steady-state commitment pricing is ideal. Compute Savings Plan provides up to 66% discount AND maintains flexibility (any Region, instance family, OS, tenancy) — better than an EC2 Instance SP if the team might change families.
  - **Not Spot**: Cannot tolerate interruptions — financial dashboards require continuous availability.

- **Workload C → m6g.xlarge (half-sized) with mixed instances**:
  - **General purpose (M-family)**: Balanced CPU (35%) and memory (40%) utilization — still needs both, so M-family is correct.
  - **Half-sized (xlarge vs. 2xlarge)**: Current utilization is 35% CPU / 40% memory at peak. An xlarge (4 vCPU, 16 GiB) at 2× traffic means ~70% CPU / ~80% memory — well within safe operating range while P99 latency (85 ms) has 15 ms headroom.
  - **On-Demand baseline + Spot scaling**: Use On-Demand for the minimum ASG capacity (overnight baseline), Spot Instances for scaling during 9 AM–6 PM peak. Web servers are stateless — Spot interruptions are handled by the ASG launching replacements.

### B. C5 On-Demand + R5 RI + keep M5 — ❌ Incorrect

- **c5.4xlarge On-Demand for transcoding**: On-Demand is the most expensive pricing model for a workload that can tolerate Spot interruptions. Also, c5.4xlarge (16 vCPU) is double the CPU of the current instance — over-provisioned. No Graviton savings.
- **r5.4xlarge 3-year RI**: 3-year commitment provides deeper discount but less flexibility than Savings Plans. r5.4xlarge has 128 GiB — insufficient for a 500 GB in-memory dataset (would need 4+ instances, adding complexity). No Graviton.
- **Keep m5.2xlarge for web**: Doesn't right-size. CPU is at 35% — half the instance is wasted at peak.
- This approach misses Graviton, Spot, and right-sizing optimizations.

### C. GPU Spot + oversized memory + burstable — ❌ Incorrect

- **p4d.24xlarge GPU Spot for transcoding**: Extreme over-provisioning. p4d.24xlarge has 8× A100 GPUs, 96 vCPU, 1,152 GiB RAM — costs ~$32/hour On-Demand. Video transcoding typically uses CPU or specialized ASIC (MediaConvert), not multi-GPU HPC instances. Even on Spot, this is far more expensive than necessary.
- **x2idn.16xlarge (1 TB RAM)**: The dataset is 500 GB — 1 TB provides 100% headroom which is wasteful. This is a storage-optimized variant with NVMe SSDs — unnecessary for an in-memory dataset. Over-provisioned.
- **t3.xlarge burstable for web**: Burstable instances are cost-effective only if average CPU is below baseline. At 2× daily peak, t3.xlarge uses burst credits — if credits deplete, performance degrades and P99 latency exceeds the 100 ms target. Risky for production web applications with strict latency SLAs.

### D. Same family for all + 3-year EC2 Instance SP — ❌ Incorrect

- **m7g.xlarge for ALL workloads**: General purpose is wrong for compute-bound (Workload A, needs C-family) and memory-bound (Workload B, needs R-family) workloads. For Workload A, M-family wastes memory budget on unused RAM. For Workload B, m7g.xlarge has only 16 GiB — catastrophically undersized for a 500 GB dataset.
- **3-year EC2 Instance SP**: Locks to a specific instance family (m7g) and Region. If workload requirements change (different family, different Region), the commitment is wasted. Compute Savings Plan provides similar savings with more flexibility.
- **Auto Scaling for all**: Workload A is batch (should be scheduled, not auto-scaled). Workload B is a monolithic in-memory engine (can't just add/remove instances — the 500 GB dataset isn't partitioned across an ASG).
- This "one-size-fits-all" approach repeats the original mistake.

## Recommendations

- **Match instance family to resource bottleneck**: CPU-bound → C-family, Memory-bound → R-family, Balanced → M-family, GPU → P/G-family, Storage I/O → I/D-family.
- **Graviton provides 20-40% better price-performance** for compatible workloads. Most open-source languages/runtimes (Java, Python, Node.js, .NET 6+, Go) support ARM. Check application compatibility before migrating.
- **Use Compute Optimizer** — it analyzes 14 days of CloudWatch metrics and recommends right-sized instance types. Enable Enhanced Infrastructure Metrics for 3-month look-back.
- **Pricing model selection**: Spot for interruptible batch, Savings Plans for steady-state 24/7, On-Demand for spiky/unknown. Mixed strategies (On-Demand baseline + Spot scaling) optimize cost for predictable patterns.
- **Burstable instances (T-family)** are only cost-effective when average CPU stays below the baseline credit rate. For sustained workloads, fixed-performance instances (M/C/R) are more predictable and often cheaper.

## Relevant Links

- [Amazon EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [AWS Graviton Processor](https://aws.amazon.com/ec2/graviton/)
- [Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
- [Savings Plans](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
- [Spot Instances Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)
- [EC2 Auto Scaling Mixed Instances Policy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-mixed-instances-groups.html)
