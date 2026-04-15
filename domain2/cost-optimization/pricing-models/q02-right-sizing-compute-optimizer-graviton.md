# Q02: Right-Sizing with Compute Optimizer and Graviton Migration

## Question

A company's AWS monthly bill is $280,000 — 65% is EC2 compute. The FinOps team enabled AWS Compute Optimizer and received recommendations for 150 instances. Key findings:

| Instance Group | Current Type | Count | Avg CPU | Avg Memory | Compute Optimizer Recommendation | Monthly Cost (Current) |
|---|---|---|---|---|---|---|
| Web Servers | m5.2xlarge (8 vCPU, 32 GB) | 40 | 28% | 35% | m6g.xlarge (4 vCPU, 16 GB) Graviton | $36,800 |
| API Servers | c5.4xlarge (16 vCPU, 32 GB) | 30 | 72% | 45% | c6g.2xlarge (8 vCPU, 16 GB) Graviton | $41,400 |
| Java App Servers | r5.2xlarge (8 vCPU, 64 GB) | 50 | 40% | 78% | r6g.2xlarge (8 vCPU, 64 GB) Graviton | $63,800 |
| ML Inference | g4dn.xlarge (4 vCPU, 16 GB, 1 GPU) | 30 | GPU 85% | 60% | No recommendation (GPU optimized) | $40,000 |

Constraints:
- Web servers run nginx (stateless, easily portable)
- API servers run Node.js services
- Java app servers run Spring Boot on Amazon Corretto 17
- ML inference runs TensorFlow Serving
- All workloads run in an Auto Scaling group behind ALB
- The team has a 1-year Compute Savings Plan at $120/hour that covers current usage

Which migration approach maximizes cost savings while minimizing risk?

## Options

- **A.** Phase 1: Migrate Web Servers to m6g.xlarge (Graviton) — stateless nginx is trivially Graviton-compatible. Use ALB weighted target groups: 10% → Graviton → 50% → 100% over 2 weeks. Phase 2: Migrate API Servers to c6g.2xlarge — Node.js runs natively on ARM/Graviton with no code changes. Same gradual traffic shift. Phase 3: Migrate Java App Servers to r6g.2xlarge — Corretto 17 supports ARM natively. Test with staging first, then gradual shift. Phase 4: Keep ML Inference on g4dn.xlarge — GPU workloads require ARM-compatible model compilation (TensorFlow on Graviton GPU is not available). Evaluate Inf2 (Inferentia) for future cost reduction. Compute Savings Plan applies to ALL Graviton instances automatically (Compute SP is instance-family-agnostic).
- **B.** Migrate all 150 instances to Graviton simultaneously in a single maintenance window. Update the Auto Scaling group launch template to use Graviton instance types. Apply a new EC2 Instance Savings Plan for Graviton types.
- **C.** Don't migrate to Graviton — instead, right-size within the same family. Web: m5.xlarge, API: c5.2xlarge, Java: keep r5.2xlarge, ML: keep g4dn.xlarge. No Graviton risk.
- **D.** Migrate Web and API to Graviton. Re-architect Java App Servers to Lambda (eliminate EC2 entirely). Migrate ML Inference to SageMaker Serverless Inference. Cancel the Savings Plan.

## Answers

### A. Phased Graviton migration (Web → API → Java) + keep GPU + existing Savings Plan — ✅ Correct

This approach maximizes savings with controlled risk:

- **Phase 1 — Web Servers (m5.2xlarge → m6g.xlarge)**:
  - **Savings**: m5.2xlarge ($0.384/hr) × 40 = $15.36/hr → m6g.xlarge ($0.154/hr) × 40 = $6.16/hr. **60% reduction** ($9.20/hr saved = $6,624/month).
  - **Why this is safe**: nginx is compiled for ARM/Graviton — Amazon Linux 2 and AL2023 include ARM-optimized nginx packages. Stateless web servers are the lowest-risk Graviton migration.
  - **Right-sizing included**: m5.2xlarge (8 vCPU, 32 GB) at 28% CPU → m6g.xlarge (4 vCPU, 16 GB) is half the size. Graviton provides ~20% better performance per vCPU, so 4 Graviton vCPUs ≈ 5 x86 vCPUs — sufficient for 28% utilization workload.
  - **Gradual traffic shift**: ALB weighted target groups (10% → 50% → 100%) allow side-by-side validation. If Graviton instances show errors or latency regression, shift traffic back to x86 immediately.

- **Phase 2 — API Servers (c5.4xlarge → c6g.2xlarge)**:
  - **Savings**: c5.4xlarge ($0.68/hr) × 30 = $20.40/hr → c6g.2xlarge ($0.272/hr) × 30 = $8.16/hr. **60% reduction** ($12.24/hr saved = $8,813/month).
  - **Node.js on Graviton**: Node.js (V8 engine) runs natively on ARM with no code changes. Performance benchmarks show 10-20% better throughput on Graviton for Node.js workloads.
  - **Right-sizing**: 16 vCPU at 72% CPU → 8 vCPU Graviton with ~20% better performance ≈ 9.6 effective vCPUs. 72% × 16 = 11.5 vCPUs utilized → 9.6 effective Graviton vCPUs at ~120% utilization — too tight. Consider c6g.4xlarge (16 vCPU) for headroom, or c6g.2xlarge with Auto Scaling to add instances at 80% CPU.

- **Phase 3 — Java App Servers (r5.2xlarge → r6g.2xlarge)**:
  - **Savings**: r5.2xlarge ($0.504/hr) × 50 = $25.20/hr → r6g.2xlarge ($0.403/hr) × 50 = $20.15/hr. **20% reduction** ($5.05/hr saved = $3,636/month).
  - **Amazon Corretto 17 on ARM**: Corretto is Amazon's distribution of OpenJDK. Corretto 17 has excellent ARM/Graviton support with optimized JIT compilation. Spring Boot on Corretto 17 runs on Graviton with zero code changes — only the AMI/container base image changes.
  - **Same size (r6g.2xlarge)**: Memory is the bottleneck (78% utilization). Keeping 64 GB ensures headroom. Graviton's ~20% better price-performance means the same specs cost 20% less.
  - **Staging test first**: Java applications with native libraries (JNI) may need Graviton-compatible JARs. Test in staging to verify all dependencies are ARM-compatible.

- **Phase 4 — ML Inference stays on g4dn.xlarge**:
  - GPU instances (g4dn, p3, p4d) use NVIDIA GPUs — they are x86 (Intel/AMD) instances with GPU accelerators. There are no Graviton GPU instances. TensorFlow Serving runs on NVIDIA GPUs and requires CUDA — incompatible with Graviton.
  - **Future optimization**: Evaluate AWS Inferentia 2 (Inf2) instances — purpose-built for ML inference at up to 50% lower cost than GPU instances. Requires model compilation with Neuron SDK, which takes development effort.

- **Compute Savings Plan continuity**:
  - Compute Savings Plans apply to ANY instance family, Region, OS, and tenancy — including Graviton. The existing $120/hr commitment continues to provide discounts after migration. No plan changes needed.
  - If the team had EC2 Instance Savings Plans (locked to m5/c5/r5), Graviton migration would lose the SP discount. Compute SP's flexibility is critical for right-sizing and Graviton migrations.

- **Total estimated savings**: $6,624 + $8,813 + $3,636 = **$19,073/month** (~$229K/year) — a 10.5% reduction on the total $182K compute bill.

### B. Simultaneous migration of all 150 instances — ❌ Incorrect

- **Big-bang migration**: Updating 150 instances simultaneously risks a complete outage if any workload has Graviton compatibility issues (e.g., Java native libraries, GPU dependencies). The blast radius is the entire production fleet.
- **No validation period**: No gradual traffic shift means the first users experience any issues — not a canary group. Problems are detected by production incidents, not controlled testing.
- **New EC2 Instance Savings Plan**: EC2 Instance SPs are locked to a specific instance family. If the team discovers issues with one Graviton family and needs to roll back, the SP commitment is wasted. The existing Compute SP already covers Graviton — no new plan needed.

### C. Right-size within x86 only — ❌ Incorrect

- **m5.xlarge**: Right-sizing from m5.2xlarge to m5.xlarge saves ~50% on web servers ($0.384 → $0.192/hr). But m6g.xlarge costs $0.154/hr — Graviton provides an additional 20% savings on top of right-sizing. Leaving money on the table.
- **c5.2xlarge**: Same logic — c5.2xlarge ($0.34/hr) vs. c6g.2xlarge ($0.272/hr). The 20% Graviton discount compounds with right-sizing.
- **"No Graviton risk"**: For nginx, Node.js, and Corretto 17, Graviton migration risk is minimal — these runtimes have been ARM-compatible for years. The risk is manageable with gradual traffic shifting. Avoiding Graviton to avoid risk means accepting 20% higher costs indefinitely.
- Right-sizing without Graviton saves ~$10K/month. With Graviton: ~$19K/month. The incremental $9K/month is worth the migration effort.

### D. Re-architect to Lambda/serverless — ❌ Incorrect

- **Java Spring Boot to Lambda**: Spring Boot applications have 10-30 second cold starts on Lambda. Spring's framework initialization (dependency injection, bean instantiation) is designed for long-running processes, not serverless cold starts. Re-architecting to Lambda requires rewriting endpoints as individual Lambda functions — a months-long effort for 50 servers' worth of code.
- **SageMaker Serverless Inference for ML**: Serverless Inference has cold starts (30-60 seconds) and is unsuitable for real-time inference workloads that need consistent < 100 ms latency. At 30 instances × 85% GPU utilization, this is a high-throughput workload better served by provisioned infrastructure.
- **Cancel Savings Plan**: Compute Savings Plans are non-cancellable commitments. You pay the committed $/hour for the full term regardless of usage. The plan provides discounts on Lambda and Fargate too, so it's still valuable even if some workloads move to serverless.
- This approach has high migration risk, long delivery timeline, and doesn't leverage the existing Savings Plan effectively.

## Recommendations

- **Graviton migration priority**: Start with stateless, interpreted/JIT workloads (nginx, Node.js, Python, Java) — they're trivially portable. Save compiled/native workloads for later.
- **Use Compute Optimizer Enhanced Infrastructure Metrics** (3-month look-back with CloudWatch agent memory metrics) for right-sizing accuracy. Default is 14 days — too short for seasonal workloads.
- **Gradual traffic shift via ALB weighted target groups**: Create a second target group with Graviton instances in the same ASG (mixed instance types). Shift weights: 10% → 25% → 50% → 100%.
- **Compute Savings Plans + Graviton = compounding savings**: SP provides 20-30% vs. on-demand, Graviton provides additional 20% — total ~40-45% savings vs. x86 on-demand.
- **CloudWatch dashboards for migration monitoring**: Track P99 latency, error rate, and CPU utilization side-by-side for x86 and Graviton instances during the migration window.

## Relevant Links

- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
- [AWS Graviton Processor](https://aws.amazon.com/ec2/graviton/)
- [Graviton Getting Started Guide](https://github.com/aws/aws-graviton-getting-started)
- [ALB Weighted Target Groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)
- [Savings Plans](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
