# Q01: Identifying Cost Optimization Opportunities from Usage Reports and Tagging

## Question

A company's AWS monthly spend has grown from $120K to $340K over 18 months without proportional business growth. The CFO demands a 30% cost reduction. The cloud team has gathered the following data from Cost Explorer and Trusted Advisor:

**Cost Explorer findings:**

| Service | Monthly Cost | % of Total | Trend |
|---|---|---|---|
| EC2 (On-Demand) | $142,000 | 42% | ↑ 15% MoM |
| RDS | $68,000 | 20% | Flat |
| S3 | $45,000 | 13% | ↑ 8% MoM |
| Data Transfer | $38,000 | 11% | ↑ 20% MoM |
| Other | $47,000 | 14% | Mixed |

**Trusted Advisor findings:**
- 38 EC2 instances with average CPU < 5% for 14 days
- 12 unattached EBS volumes (total 8 TB, $800/month)
- 5 idle RDS instances (no connections for 30+ days, $3,200/month combined)
- 4 underutilized RDS instances (CPU < 10%)
- 22 unassociated Elastic IP addresses ($110/month)

**Tagging audit:**
- Only 45% of resources have a `CostCenter` tag
- No `Environment` tag (can't distinguish dev/staging/prod costs)
- No `Owner` tag (can't assign costs to teams)

**Team observations:**
- "We don't know which team owns which resources"
- "Developers create instances and forget to delete them"
- "We have no idea what's dev vs. production"
- "S3 costs keep growing but nobody manages lifecycles"

What is the most effective cost optimization strategy?

## Options

- **A.** **Phase 1 — Quick wins (Week 1-2, ~$8K/month savings):** Delete 12 unattached EBS volumes ($800/month). Terminate 5 idle RDS instances ($3,200/month). Release 22 unassociated EIPs ($110/month). Stop 38 low-CPU EC2 instances after confirming with owners via CloudTrail `RunInstances` event creator ($3,800/month estimate). **Phase 2 — Tagging enforcement (Week 2-4):** Implement mandatory tagging via **AWS Organizations SCP** — deny `ec2:RunInstances`, `rds:CreateDBInstance`, `s3:CreateBucket` unless tags `CostCenter`, `Environment`, `Owner` are present. Retroactively tag existing resources using **Tag Editor** (bulk-tag by account/resource type). Activate **Cost Allocation Tags** in the Billing console for `CostCenter`, `Environment`, `Owner` — enables grouping in Cost Explorer. **Phase 3 — Right-sizing and pricing models (Week 4-8):** Use **AWS Compute Optimizer** to generate right-sizing recommendations for EC2 and RDS. Convert stable workloads (always-on production) to **Savings Plans** (Compute Savings Plans for flexibility, EC2 Instance Savings Plans for deeper discount). Implement **S3 Intelligent-Tiering** on buckets with unknown access patterns. Add **S3 Lifecycle Policies**: transition to S3 IA after 30 days, Glacier after 90 days, Glacier Deep Archive after 365 days, delete after 7 years (if compliance allows). **Phase 4 — Data Transfer optimization (Week 8-12):** Audit data transfer paths using **VPC Flow Logs**. Replace cross-AZ traffic where possible (move communicating services to the same AZ or use VPC endpoints). Use **S3 Gateway Endpoint** (free) and **DynamoDB Gateway Endpoint** (free) to eliminate NAT Gateway charges for S3/DynamoDB traffic. Evaluate **CloudFront** for frequently accessed S3 content (CloudFront data transfer is cheaper than S3 direct transfer in most cases). **Phase 5 — Governance (ongoing):** Create **AWS Budgets** with alerts at 80% and 100% of monthly target. Use **Cost Anomaly Detection** to detect unexpected cost spikes (alerts within hours, not at month-end). Schedule **non-production environments** to run only during business hours (Instance Scheduler or ASG scheduled actions) — saves 70% on dev/staging compute. Report costs by `CostCenter` and `Owner` — make teams accountable for their spend.
- **B.** Move all workloads to Spot Instances for maximum savings. Spot pricing is 60-90% cheaper than On-Demand.
- **C.** Commit to a 3-year All Upfront Reserved Instance plan for all EC2 and RDS instances. This provides the maximum discount (up to 72% off On-Demand).
- **D.** Migrate all applications to Lambda (serverless) and S3 for storage. Serverless automatically optimizes cost by charging only for actual usage.

## Answers

### A. Phased approach — quick wins, tagging, right-sizing, pricing, governance — ✅ Correct

**Phase 1 — Quick Wins ($8K/month, minimal risk):**

- **Unattached EBS volumes ($800/month):** These are abandoned volumes from terminated instances. Use AWS CLI:
  ```
  aws ec2 describe-volumes --filters "Name=status,Values=available"
  ```
  "Available" status = not attached to any instance. Snapshot before deleting (safety net), then delete. Cost: $0.08/GB/month × 8,000 GB = $640/month for gp3; $800/month confirms a mix of volume types.

- **Idle RDS instances ($3,200/month):** 30+ days with zero connections. Steps:
  1. Check CloudTrail for `CreateDBInstance` — identify who created each one
  2. Notify owners with a 7-day warning
  3. Take a final snapshot (`create-db-snapshot`)
  4. Delete instances after confirmation
  - For "just in case" instances: Use RDS snapshots. Keep the snapshot ($0.02/GB/month) instead of a running instance ($640/month for r6g.xlarge). Restore from snapshot in 15-30 minutes if needed.

- **Unassociated EIPs ($110/month):** AWS charges for EIPs NOT associated with running instances ($0.005/hour = $3.60/month each). 22 × $3.60 = $79.20/month. Release all unassociated EIPs. If they're "reserved for future use" — don't reserve EIPs; allocate new ones when needed (IPv4 addresses cost $0.005/hour charged or not since February 2024).

- **Low-CPU EC2 instances ($3,800/month):** 38 instances with < 5% CPU for 14 days. These are candidates for:
  1. **Termination** — if nobody claims them after notification
  2. **Right-sizing** — if claimed but oversized (m5.4xlarge at 5% CPU → t3.small)
  3. **Scheduling** — if they're dev/test that runs 24/7 but is used only during business hours
  - Use CloudTrail `RunInstances` event to identify the creator. Tag with `Owner` and hold creators accountable.

**Phase 2 — Tagging Enforcement (foundation for all future optimization):**

- **Without tags, cost optimization is guesswork.** You can't reduce costs if you can't answer: "Which team owns this?" and "Is this production or development?"

- **SCP for mandatory tagging:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "RequireTags",
      "Effect": "Deny",
      "Action": ["ec2:RunInstances", "rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/CostCenter": "true",
          "aws:RequestTag/Environment": "true",
          "aws:RequestTag/Owner": "true"
        }
      }
    }]
  }
  ```
  This prevents creating new resources without required tags. Apply to all accounts in the Org (except sandbox/exempt accounts).

- **Tag Editor for retroactive tagging:**
  - AWS Tag Editor → select Region → select resource types → filter by "no CostCenter tag" → bulk-apply tags
  - For 55% untagged resources: use a combination of CloudTrail analysis (who created it?) and naming conventions (resource names often indicate team/environment)

- **Cost Allocation Tags:**
  - In the Billing console → Cost Allocation Tags → activate `CostCenter`, `Environment`, `Owner`
  - After activation (24-48 hours), Cost Explorer can group/filter by these tags
  - Create Cost Explorer reports: "Monthly cost by CostCenter" and "Monthly cost by Environment"

**Phase 3 — Right-Sizing and Pricing Models ($40-60K/month savings potential):**

- **Compute Optimizer recommendations:**
  - Analyzes 14 days of CloudWatch metrics (CPU, memory, network, disk) for EC2 and RDS
  - Recommends: downsize (oversized), upsize (constrained), or change family (wrong workload type)
  - Example: m5.4xlarge at 15% CPU → m6g.xlarge (Graviton, smaller, cheaper) = 75% savings on that instance

- **Savings Plans:**

  | Plan Type | Discount | Flexibility | Best For |
  |---|---|---|---|
  | Compute Savings Plans | 20-40% | Any EC2, Fargate, Lambda | Mixed workloads, changing instance types |
  | EC2 Instance Savings Plans | 30-50% | Specific instance family + Region | Stable production workloads |
  | 1-year No Upfront | 20-30% | Lower commitment | Uncertain growth |
  | 3-year All Upfront | 50-72% | Highest commitment | Very stable workloads |

  - **Start with 1-year Compute Savings Plans** covering 60-70% of baseline usage. This gives flexibility (change instance types, use Fargate later) with meaningful savings.
  - Use the **Savings Plans recommendations** in Cost Explorer (analyzes 30 days of usage, suggests optimal commitment level).

- **S3 Intelligent-Tiering:**
  - Automatically moves objects between access tiers based on access patterns:
    - Frequent Access tier (standard pricing)
    - Infrequent Access tier (40% cheaper, after 30 days no access)
    - Archive Instant Access tier (68% cheaper, after 90 days)
    - Archive Access tier (optional, after configurable days)
    - Deep Archive Access tier (optional, after configurable days)
  - No retrieval fees. Small monitoring fee ($0.0025/1,000 objects/month).
  - Ideal when you DON'T know access patterns — let AWS optimize automatically.

- **S3 Lifecycle Policies:**
  - For known patterns: set explicit transitions
  - Logs: delete after 90 days (or transition to Glacier after 30 days if retention required)
  - Backups: Glacier after 30 days, Deep Archive after 365 days
  - Impact: S3 Standard = $0.023/GB/month → Glacier = $0.004/GB/month = 83% savings for cold data

**Phase 4 — Data Transfer ($38K/month — 11% of total):**

- **Cross-AZ traffic**: EC2 instance in AZ-a calls RDS in AZ-b = $0.01/GB each way ($0.02/GB round trip). If they exchange 5 TB/month = $100/month per service pair. Solution: co-locate communicating services in the same AZ where possible, or use caching to reduce cross-AZ calls.

- **NAT Gateway transfer charges**: NAT Gateway charges $0.045/GB processed. If 10 TB/month passes through NAT GW = $450/month in processing fees alone. S3 and DynamoDB traffic routed through NAT GW can be redirected to free VPC gateway endpoints.

- **S3 → CloudFront**: For publicly accessed S3 content, CloudFront data transfer is $0.085/GB (first 10 TB) vs. S3 direct transfer $0.09/GB. CloudFront also caches at the edge, reducing total transfer volume.

**Phase 5 — Governance (sustaining savings):**

- **AWS Budgets**: Set $240K/month budget (30% reduction from $340K). Alert at $192K (80%), $240K (100%), $264K (110% — forecasted overage alert).
- **Cost Anomaly Detection**: ML-based service that detects unusual spending patterns. Alerts on: service-level anomalies, account-level anomalies, and cost-category anomalies. Catches "someone launched 50 GPU instances" within hours.
- **Non-production scheduling**: Instance Scheduler solution → start dev/staging at 08:00, stop at 20:00 weekdays. 12 hours/day × 5 days/week = 60 hours/week vs. 168 hours/week = 64% savings on non-prod compute.

### B. All Spot Instances — ❌ Incorrect

- **Spot can be interrupted with 2-minute warning**: Not suitable for stateful applications (RDS, ElastiCache), user-facing web servers, or any workload requiring consistent availability.
- **Spot is excellent for specific workloads**: batch processing, CI/CD builds, data analytics, test environments. Use Spot for 10-20% of the fleet (fault-tolerant workloads), not 100%.
- **Ignores the root causes**: No tagging, no lifecycle policies, no right-sizing. Even at 90% cheaper, running 38 idle instances on Spot still wastes money.

### C. 3-year All Upfront RI for everything — ❌ Incorrect

- **Commits to current inefficiency**: Buying 3-year RIs for 38 low-CPU instances and 5 idle RDS instances locks in waste for 3 years. Right-size FIRST, then commit.
- **No flexibility**: 3-year commitments assume instance families and regions don't change. Over 3 years, the company will likely: adopt Graviton (different family), try Fargate/Lambda (different service), or change regions.
- **Includes dev/staging**: Dev instances should be scheduled (run 40 hours/week, not 168). Buying RIs for dev instances paying for 168 hours/week when they're needed 40 = paying for 128 hours of idle time at a "discount."
- **Correct approach**: Right-size first → Savings Plans (more flexible than RIs) → 1-year commitment first → extend to 3-year only for proven stable workloads.

### D. Migrate to serverless — ❌ Incorrect

- **Massive migration effort**: Rewriting all applications to Lambda is a multi-quarter engineering project. The CFO wants cost reduction now, not in 6-12 months.
- **Lambda isn't always cheaper**: At high sustained throughput, Lambda can be MORE expensive than EC2. A function executing 100M times/month at 1 second each = $200,000/month in Lambda costs vs. a few thousand in EC2.
- **Not all workloads suit Lambda**: 15-minute execution timeout, 10 GB memory limit, no persistent connections, cold starts. Production databases can't run on Lambda.
- **S3 isn't a replacement for all storage**: S3 is object storage. It can't replace EBS (block storage for databases) or EFS (shared file system with POSIX semantics).

## Recommendations

- **Optimize in order: Eliminate → Right-size → Commit → monitor**: First remove waste (idle resources, orphaned volumes). Then right-size (match resource size to actual usage). Then commit (Savings Plans for predictable baseline). Then monitor (Budgets, Cost Anomaly Detection, monthly reviews).
- **Tags are non-negotiable**: Without `CostCenter`, `Environment`, and `Owner` tags, cost optimization is guessing. Implement tagging enforcement (SCP) before any other cost initiative.
- **Cost Explorer + Compute Optimizer + Trusted Advisor**: Use all three together. Cost Explorer shows WHAT you're spending. Compute Optimizer shows WHERE to right-size. Trusted Advisor shows WHAT's idle or orphaned.
- **Non-production scheduling**: The single highest-impact action for most organizations. Dev environments running 24/7 waste 70% of their compute cost. Schedule them to business hours only.
- **Review costs monthly**: Create a monthly cost review cadence. Review Budget vs. actual, top 5 cost drivers, anomaly alerts, and right-sizing recommendations. Assign cost reduction goals per team based on `CostCenter` reports.

## Relevant Links

- [AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
- [Savings Plans](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
- [S3 Intelligent-Tiering](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intelligent-tiering.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)
- [Tag Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies.html)
- [Instance Scheduler](https://docs.aws.amazon.com/solutions/latest/instance-scheduler-on-aws/solution-overview.html)
