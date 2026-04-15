# Q01: Pricing Model Selection for Mixed Workload Portfolio

## Question

A company runs the following workloads in us-east-1 and wants to optimize compute costs. The CTO has approved a 1-year commitment budget of $500,000.

| Workload | Instance Type | Count | Usage Pattern | Interruption Tolerance |
|----------|--------------|-------|---------------|----------------------|
| Production API | m6i.2xlarge | 10 | 24/7, steady | None — SLA-bound |
| Data Pipeline | c6i.4xlarge | 8 | 24/7, steady | None — job ordering dependencies |
| ML Training | p3.8xlarge | 4 | 10 hrs/day, weekdays only | High — checkpoints every 30 min |
| Dev/Test | m6i.xlarge | 15 | 8 hrs/day, weekdays only | Full — can be terminated anytime |
| Seasonal Analytics | r6i.2xlarge | 6 | 3 months/year (Q4), 24/7 during Q4 | None during Q4 |

Current state: All workloads run on On-Demand. Monthly bill is approximately $85,000. The team wants to reduce costs by at least 40%.

Which pricing strategy achieves the target savings?

## Options

- **A.** Purchase a Compute Savings Plan at $28/hour commitment (covers Production API + Data Pipeline 24/7 baseline). Use Spot Instances for ML Training with Spot Fleet diversification across p3.8xlarge, p3.16xlarge, and p3dn.24xlarge. Use Spot Instances with Auto Scaling for Dev/Test. Use On-Demand for Seasonal Analytics only during Q4 (no commitment for 3-month usage).
- **B.** Purchase EC2 Instance Savings Plans: m6i plan for Production API + Dev/Test, c6i plan for Data Pipeline, r6i plan for Seasonal Analytics. Use Reserved Instances (1-year, All Upfront) for ML Training p3.8xlarge.
- **C.** Purchase Standard Reserved Instances (1-year, All Upfront) for all workloads at their current instance types and full counts. Use Scheduled Reserved Instances for Dev/Test and ML Training.
- **D.** Use Spot Instances for everything with a Spot Fleet mixed-instances policy. Add On-Demand capacity reservations as backup for Production API.

## Answers

### A. Compute SP (steady-state) + Spot (training + dev/test) + On-Demand (seasonal) — ✅ Correct

This strategy matches each pricing model to the workload's characteristics:

- **Compute Savings Plan at $28/hour**:
  - Covers the 24/7 steady-state workloads (Production API + Data Pipeline) that are commitment-worthy. These run continuously and cannot be interrupted — ideal for commitment-based pricing.
  - **Why Compute SP over EC2 Instance SP**: Compute Savings Plans apply to ANY instance family, Region, OS, tenancy, and even Fargate/Lambda. If the team migrates from m6i to m7g (Graviton) or changes Regions, the commitment still applies. EC2 Instance SP is locked to a specific family + Region — less flexible.
  - $28/hour × 8,760 hours/year = $245,280 annual commitment. The underlying On-Demand cost for 10× m6i.2xlarge + 8× c6i.4xlarge running 24/7 ≈ $42/hour. The $28/hour commitment represents ~33% savings — aligning with Compute SP's typical 1-year discount.

- **Spot Instances for ML Training**:
  - 10 hours/day × weekdays × 4 instances = ~200 GPU-hours/day. At p3.8xlarge On-Demand ($12.24/hour), this is ~$2,448/day. Spot pricing is typically 60-70% off for GPU instances.
  - **Fleet diversification** across p3.8xlarge, p3.16xlarge, p3dn.24xlarge: The Spot Fleet selects from multiple pools — if one pool's capacity is constrained, another provides instances. This reduces Spot interruption frequency.
  - **High interruption tolerance**: ML training with 30-minute checkpoints means at most 30 minutes of work is lost on interruption. Spot is the correct pricing model.

- **Spot Instances for Dev/Test**:
  - Full interruption tolerance — can be terminated anytime. Dev/test environments are the textbook Spot use case.
  - 8 hours/day × weekdays × 15 instances. Spot saves ~70% vs. On-Demand.

- **On-Demand for Seasonal Analytics**:
  - Only 3 months/year (Q4). A 1-year commitment would pay for 12 months to use 3 — a 75% utilization rate that erases the discount benefit. On-Demand for only Q4 is cheaper than committing.
  - If the seasonal pattern is predictable year-over-year, the team could explore Convertible Reserved Instances and sell them on the RI Marketplace after Q4 — but this adds operational complexity.

- **Combined savings estimate**: ~33% on steady-state (SP) + ~65% on ML Training (Spot) + ~70% on Dev/Test (Spot) + 0% on Seasonal Analytics = blended ~45% savings — meeting the 40% target.

### B. EC2 Instance SPs for each family + RI for ML Training — ❌ Incorrect

- **EC2 Instance SP for Dev/Test (m6i plan)**: Dev/Test runs 8 hours/day, weekdays — that's ~24% utilization. A Savings Plan commits to $/hour for all 8,760 hours/year. You'd pay for 100% utilization but use only 24%. The commitment cost exceeds On-Demand cost for that usage pattern. This is wasting money, not saving it.
- **EC2 Instance SP for r6i (Seasonal Analytics)**: Same problem — 3 months usage = 25% utilization. Paying for 12 months of commitment is worse than On-Demand.
- **Reserved Instance for ML Training**: Standard RI is locked to exact instance type, AZ, and platform. p3.8xlarge availability varies — if the team needs flexibility to switch to p3.16xlarge or p3dn.24xlarge, the RI is wasted. Plus, 10 hours/day weekdays = ~30% utilization — RI is overcommitted.
- This approach over-commits on low-utilization workloads.

### C. Standard RI for all + Scheduled RI — ❌ Incorrect

- **Standard RI for ALL workloads**: Standard RIs cannot be exchanged for different instance types. If the team right-sizes, migrates to Graviton, or changes workloads, these RIs are stranded. Savings Plans have replaced RIs as the recommended commitment mechanism for most scenarios.
- **Scheduled Reserved Instances**: AWS has discontinued the ability to purchase new Scheduled Reserved Instances. This option is no longer available. Even when available, they required a 1-year commitment for specific time windows — inflexible for workloads whose schedules may shift.
- This approach is outdated and inflexible.

### D. Spot for everything + On-Demand capacity reservations — ❌ Incorrect

- **Spot for Production API**: The Production API has an SLA and zero interruption tolerance. Spot Instances can be terminated with 2 minutes notice. This violates the SLA — production services with availability commitments should never run entirely on Spot.
- **Spot for Data Pipeline with job ordering dependencies**: If a pipeline step is interrupted, downstream steps fail or stall. While individual tasks may be retried, the pipeline ordering dependency means interruptions cascade — this is not "high interruption tolerance."
- **On-Demand capacity reservations as "backup"**: Capacity Reservations guarantee that On-Demand capacity is available in a specific AZ when needed — but you pay the On-Demand price whether you use the reservation or not. If Spot is interrupted and the workload falls back to On-Demand capacity reservations, you're paying full price during the fallback period — negating Spot savings.
- Spot is a tool for fault-tolerant workloads, not a universal pricing strategy.

## Recommendations

- **Commitment sizing rule**: Only commit (SP or RI) for the steady-state minimum — the capacity that runs 24/7, 365 days. Variable and part-time workloads should use Spot or On-Demand.
- **Compute Savings Plans** are almost always preferred over EC2 Instance SPs — they provide similar discounts with more flexibility (any family, Region, OS).
- **Spot for batch/training/dev**: Any workload that can checkpoint, retry, or be terminated is a Spot candidate. Diversify across instance types and AZs.
- **Utilization threshold for commitments**: If a workload runs less than ~70% of the time, the commitment cost may exceed On-Demand cost. Calculate: `SP hourly rate × 8,760 hours` vs. `On-Demand rate × actual annual hours`.
- **Review commitments quarterly**: Use Cost Explorer's SP/RI utilization reports to verify commitments are being used. Underutilized commitments should be right-sized at renewal.

## Relevant Links

- [Savings Plans Overview](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
- [Compute Savings Plans vs. EC2 Instance Savings Plans](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html#plan-types)
- [Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html)
- [Cost Explorer RI/SP Coverage](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-ris.html)
- [On-Demand Capacity Reservations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)
