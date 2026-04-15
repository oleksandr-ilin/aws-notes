# Q02: Managed Services vs Self-Managed Cost Analysis

## Question

A startup is migrating from on-premises to AWS and must decide between managed and self-managed options for three components. The team has 2 DevOps engineers and wants to minimize total cost of ownership (TCO), not just infrastructure cost.

**Component 1 — Container Platform**
- 20 microservices, 80 containers total
- Needs auto-scaling, rolling deployments, and health checks
- Traffic varies 3× between peak and off-peak daily
- Option A: ECS on Fargate
- Option B: ECS on EC2 (self-managed cluster of m5.xlarge instances)

**Component 2 — Relational Database**
- 200 GB PostgreSQL database
- Traffic varies: 4× peak on weekdays, near-zero on weekends
- Needs Multi-AZ for HA
- Option A: Aurora Serverless v2
- Option B: RDS PostgreSQL db.r6g.xlarge Multi-AZ with manual scaling

**Component 3 — Event Processing**
- 50,000 events/second at peak, 2,000 events/second at baseline
- Events must be processed within 30 seconds
- Option A: SQS + Lambda
- Option B: Self-managed Apache Kafka on EC2 (3-node cluster) + consumer application on EC2

For each component, the team must choose the option that minimizes 3-year TCO (infrastructure cost + operational cost).

Which combination is most cost-effective over 3 years?

## Options

- **A.** All managed: ECS Fargate + Aurora Serverless v2 + SQS/Lambda. Higher per-unit infrastructure cost but zero cluster management, auto-scaling to zero during off-peak, and 1 DevOps engineer can manage all three. The second DevOps engineer focuses on application development, accelerating feature delivery.
- **B.** All self-managed: ECS on EC2 + RDS PostgreSQL + Kafka on EC2. Lower per-unit infrastructure cost. Use Reserved Instances for the database and Kafka nodes. DevOps engineers spend 30% of time on infrastructure management (patching, scaling, monitoring).
- **C.** Mixed: ECS on EC2 (steady-state containers benefit from RI pricing) + Aurora Serverless v2 (variable database traffic) + SQS/Lambda (variable event volume). EC2 for predictable workloads, managed/serverless for variable workloads.
- **D.** Mixed: ECS Fargate (simplify container management) + RDS PostgreSQL with Reserved Instance (predictable database cost) + SQS/Lambda (simplify event processing). Managed for operations-heavy components, RI for cost-predictable components.

## Answers

### A. All managed/serverless — ✅ Correct

For a startup with 2 DevOps engineers and variable traffic, managed services minimize TCO:

- **Component 1 — ECS Fargate vs. ECS on EC2**:

  **Fargate cost analysis**:
  - 80 containers × average 0.5 vCPU + 1 GB RAM per container
  - Fargate: $0.04048/vCPU/hour + $0.004445/GB/hour
  - Peak: 80 containers × ($0.02024 + $0.004445) × 12 hours = $23.66/day
  - Off-peak (~27 containers at 1/3 traffic): 27 containers × ($0.02024 + $0.004445) × 12 hours = $7.89/day
  - Monthly: ($23.66 + $7.89) × 30 = ~$946/month
  - Fargate auto-scales to zero — off-peak cost is proportional to actual usage.

  **EC2 cost analysis**:
  - Need enough EC2 instances for peak: 80 containers on m5.xlarge (4 vCPU, 16 GB). ~5 containers per instance → 16 instances at peak.
  - EC2 m5.xlarge: $0.192/hr × 16 × 24hrs × 30 = $2,212/month (must run 24/7 for availability, even if off-peak uses 5 instances)
  - With RI (1-year): ~$1,400/month (but committed to 16 instances even when using 5)
  - **Hidden EC2 costs**: OS patching (2-4 hours/month), ASG management, instance replacement on failures, EBS volume management, security agent deployment.

  **Verdict**: Fargate is **cheaper for variable traffic** and eliminates cluster management. The 3× traffic variation means EC2 instances are 67% idle during off-peak.

- **Component 2 — Aurora Serverless v2 vs. RDS provisioned**:

  **Aurora Serverless v2**:
  - Scales in 0.5 ACU increments (1 ACU ≈ 2 GB RAM + proportional CPU)
  - Weekday peak: 8 ACUs × 12 hours × $0.12/ACU/hour = $11.52/day
  - Weekday off-peak: 2 ACUs × 12 hours × $0.12 = $2.88/day
  - Weekend: 0.5 ACUs × 24 hours × $0.12 = $1.44/day
  - Monthly: (($11.52 + $2.88) × 22 + $1.44 × 8) = ~$328/month (+ storage)

  **RDS db.r6g.xlarge Multi-AZ**:
  - $0.48/hour × 24 × 30 = $345.60/month (Multi-AZ doubles provisioned cost)
  - With 1-year RI: ~$230/month
  - **BUT**: Sized for peak (4× weekend traffic). On weekends, the instance runs at near-zero utilization — paying full price for idle capacity. Cannot scale down without downtime.
  - **Manual scaling**: Changing instance size requires a reboot (30-60 seconds downtime). Not practical for daily traffic variations.

  **Verdict**: Aurora Serverless v2 auto-scales with traffic — near-zero cost on weekends, full capacity on weekday peaks. RDS RI is cheaper at steady-state but wastes 75% of capacity on weekends.

- **Component 3 — SQS/Lambda vs. Kafka on EC2**:

  **SQS + Lambda**:
  - SQS: $0.40/million requests. 50K events/sec × 3,600 = 180M events/hour at peak. Assume 4 hours peak: 720M messages × $0.0000004 = $0.29/day at peak. Off-peak is proportionally less.
  - Lambda: Processing 50K events/sec with 128 MB, 200 ms average duration. Lambda invocations + compute: ~$250/month at average load.
  - Total: ~$300/month at average utilization.
  - Auto-scales from 2K to 50K events/sec with zero configuration.

  **Kafka on EC2 (3 nodes)**:
  - 3 × m5.xlarge: $0.192/hr × 3 × 24 × 30 = $414/month (must run 24/7)
  - 3 × consumer EC2 instances: $414/month
  - EBS storage for Kafka logs: 3 × 500 GB gp3 = $120/month
  - Total infrastructure: ~$948/month
  - **Operational cost**: Kafka requires: ZooKeeper/KRaft management, partition rebalancing, broker patching, consumer lag monitoring, retention management, upgrade planning. This consumes 10-20% of a DevOps engineer's time.

  **Verdict**: SQS/Lambda is 3× cheaper AND eliminates Kafka operational burden.

- **DevOps engineer cost**:
  - Managed services: 1 engineer sufficient for all three components (monitoring, deployment pipelines, application-level issues). Second engineer focuses on application development.
  - Self-managed: Both engineers spend 30-40% of time on infrastructure (patching 16+ EC2 instances, Kafka operations, RDS maintenance). At $150K/year fully loaded, 30% = $45K/year × 2 = $90K/year in operational labor.
  - **3-year operational savings**: ~$270K — this often exceeds the infrastructure cost difference.

### B. All self-managed — ❌ Incorrect

- **Lower per-unit infrastructure cost is a trap**: EC2 instances are cheaper per vCPU-hour than Fargate, but EC2 instances must be provisioned for peak and run 24/7. Variable traffic means paying for idle capacity.
- **RI for Kafka**: 3-year commitment for a 3-node Kafka cluster locks in operational burden for 3 years. If the team later migrates to MSK or SQS/Lambda, the RI investment is wasted.
- **30% DevOps time on infrastructure**: 2 engineers × 30% × $150K × 3 years = $270K in operational cost. This exceeds the infrastructure savings from self-managed components.
- TCO is not just infrastructure cost — it's infrastructure + labor + opportunity cost of delayed features.

### C. EC2 for containers + managed for variable — ❌ Incorrect

- **ECS on EC2 for "steady-state" containers**: The container workload varies 3× daily — this is NOT steady-state. EC2 instances provisioned for peak are 67% idle off-peak. Fargate's per-second billing aligns cost with actual container usage.
- **RI for EC2 container cluster**: Locks in 16 instances for 1-3 years. If the team optimizes containers (consolidation, smaller images, fewer replicas), the RI commitment is wasted.
- Aurora Serverless v2 and SQS/Lambda portions are correct.

### D. Fargate + RDS RI + Lambda — ❌ Incorrect

- **RDS with Reserved Instance**: RDS RI provides the lowest database cost IF traffic is predictable and steady. With 4× weekday/weekend variation and near-zero weekend traffic, the RI pays for capacity that's unused 30% of the time (weekends + off-peak). Aurora Serverless v2 costs less in total because it scales to near-zero on weekends.
- Fargate and SQS/Lambda portions are correct.
- This option is close but the database choice mismatches the traffic pattern.

## Recommendations

- **TCO = Infrastructure + Labor + Opportunity Cost**: Always calculate all three. Managed services trade higher per-unit infrastructure cost for near-zero labor cost.
- **Use managed/serverless for variable workloads**: Fargate, Aurora Serverless, Lambda, SQS — all scale to zero and bill per-use. Provisioned infrastructure (EC2, RDS) is cheaper only for steady, predictable workloads.
- **Managed/serverless for small teams**: If you have < 5 DevOps engineers, every hour spent on infrastructure management is an hour not spent on product development. Managed services buy engineering time.
- **Self-managed is cheaper at scale**: Large enterprises with dedicated infrastructure teams (10+ engineers) can achieve lower TCO with self-managed services — their labor cost is amortized across hundreds of workloads.
- **Evaluate quarterly**: As traffic patterns change, re-assess managed vs. self-managed. A workload that was variable may become steady-state after product maturity — switching to EC2 + RI may then be optimal.

## Relevant Links

- [ECS Fargate Pricing](https://aws.amazon.com/fargate/pricing/)
- [Aurora Serverless v2 Pricing](https://aws.amazon.com/rds/aurora/pricing/)
- [Lambda Pricing](https://aws.amazon.com/lambda/pricing/)
- [SQS Pricing](https://aws.amazon.com/sqs/pricing/)
- [AWS TCO Calculator](https://aws.amazon.com/economics/)
