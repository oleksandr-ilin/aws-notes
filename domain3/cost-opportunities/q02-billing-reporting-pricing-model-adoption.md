# Q02: Cost Reporting, Billing Alarms, and Pricing Model Adoption

## Question

A company operating in AWS has reached $500K/month in spend across 15 accounts in an AWS Organization. The finance team receives a single consolidated bill and distributes costs manually via spreadsheets. Current pain points:

1. **No cost visibility until month-end**: The finance team receives the bill on the 4th of the following month. By then, a $40K cost spike in one account (from an accidental p4d.24xlarge GPU cluster left running for 3 weeks) has already happened.

2. **Inaccurate cost allocation**: The company has 4 business units (BU). Shared services (networking, security, logging) cost $80K/month but are split equally ($20K each). BU-Alpha uses 60% of shared services; BU-Delta uses 5%. BU-Delta subsidizes BU-Alpha.

3. **No pricing model optimization**: 100% of EC2 spend is On-Demand. The compute profile:
   - 120 production instances (always-on, stable) — $180K/month
   - 45 dev/staging instances (business hours only, but run 24/7) — $67K/month
   - 30 batch processing instances (flexible timing, interruptible) — $45K/month
   - 25 spiky instances (ASG, variable) — $38K/month

4. **S3 cost growing unchecked**: 800 TB across 200 buckets. No lifecycle policies. Finance can't determine which buckets belong to which team — only 30% have tags.

The CFO wants: (a) real-time cost visibility, (b) accurate BU-level cost allocation, (c) pricing model optimization to reduce the $330K compute spend, and (d) S3 cost control.

Which approach addresses all four requirements?

## Options

- **A.** **Real-time visibility**: Enable **Cost Explorer** hourly granularity. Create **AWS Budgets** for each account and each BU with forecasted budget alerts (alert when FORECASTED end-of-month spend exceeds budget — warns mid-month, not month-end). Deploy **Cost Anomaly Detection** with monitors for each account and each service — detects unexpected spend within hours (would have caught the $40K GPU cluster within 24 hours). Create a custom **CUR (Cost and Usage Report)** with hourly granularity, Athena integration — query any cost dimension at any time. **Accurate cost allocation**: Create **Cost Categories** in the Billing console to map accounts and tagged resources to BUs. For shared services: use **split charge rules** — allocate shared account costs proportionally based on usage metrics (e.g., VPC Flow Logs data volume per BU, CloudWatch API call count per BU). Configure `CostCenter` tag as a **cost allocation tag** → enables grouping in Cost Explorer and CUR by BU. Result: BU-Alpha sees $48K of shared costs (60%), BU-Delta sees $4K (5%). **Pricing optimization**: (1) Schedule dev/staging instances — business hours only (10 hrs/day × 5 days/week = 50 hrs vs 168 hrs = 70% savings → $67K × 0.70 = $47K/month savings). (2) Purchase **Compute Savings Plan** (1-year, No Upfront) covering 80% of production baseline → ~30% discount on $144K = $43K/month savings. (3) Convert batch processing to **Spot Instances** with Spot Fleet diversification → 60-70% savings → $45K × 0.65 = $29K/month savings. (4) Keep spiky instances On-Demand (unpredictable, can't commit). Total compute savings: $47K + $43K + $29K = **$119K/month** (36% of compute). **S3 cost control**: Tag enforcement (SCP), Intelligent-Tiering for unknown access patterns, lifecycle policies for known patterns (logs → Glacier after 30 days, backups → Deep Archive after 365 days), S3 Storage Lens for bucket-level analytics (identifies largest, fastest-growing, and least-accessed buckets).
- **B.** Implement a third-party FinOps platform (CloudHealth, Spot.io, or similar). The platform automatically optimizes costs, provides dashboards, and handles chargebacks.
- **C.** Create a custom cost dashboard using CloudWatch metrics and Lambda functions that scrape billing data daily. Store in DynamoDB. Build a React frontend for the finance team.
- **D.** Negotiate an Enterprise Discount Program (EDP) with AWS for a bulk spend commitment. EDPs provide 5-15% discount on all AWS services in exchange for a spend commitment.

## Answers

### A. CUR + Budgets + Cost Anomaly Detection + Cost Categories + Savings Plans + Spot — ✅ Correct

**Real-Time Cost Visibility:**

- **AWS Budgets with forecasted alerts**:
  - Create a budget per account: Monthly budget = expected spend. Alert at 80% actual AND 100% forecasted.
  - **Forecasted alert**: AWS Budgets uses your historical spending pattern to forecast end-of-month spend by mid-month. If the forecast exceeds the budget, you're alerted 2 weeks before the bill arrives.
  - Example: Account-Alpha budget = $50K. On day 10, actual spend = $20K, but forecasted end-of-month = $65K (because someone launched GPU instances). Alert fires immediately — 20 days before the bill.
  - **Budget actions**: Automatically apply an SCP that denies `ec2:RunInstances` if the budget is exceeded. Or apply an IAM policy that restricts specific instance types. Prevents runaway costs automatically.

- **Cost Anomaly Detection**:
  - ML-based service that learns your spending patterns and alerts on deviations
  - Alert granularity: can detect a $500 daily spike in a specific service within a specific account
  - The $40K GPU cluster would have triggered an alert within 24 hours (first day's spend would be ~$2K, well above the account's normal daily compute spend)
  - Configure: separate monitors for each AWS account, each major service (EC2, RDS, S3), and each cost category

- **Cost and Usage Report (CUR)**:
  - The most detailed cost data AWS provides: line-item billing for every resource, every hour
  - Delivered to S3, queryable with Athena (SQL queries over your entire billing history)
  - Use cases: "How much did account-Beta spend on EC2 in eu-west-1 on March 15?" — query CUR with Athena, answer in seconds
  - Integration: CUR → S3 → Athena → QuickSight dashboards for the finance team
  - Enable **resource-level** reporting: each line item includes the resource ID (instance ID, bucket name, etc.) — enables per-resource cost tracking

**Accurate Cost Allocation — Cost Categories and Split Charges:**

- **Cost Categories**:
  - Define rules that map costs to business categories (BUs, products, teams)
  - Rule types:
    - Account-based: "Account 123456789012 → BU-Alpha"
    - Tag-based: "CostCenter=Alpha → BU-Alpha"
    - Service-based: "All RDS costs → Database team"
    - Charge type-based: "All RI/SP amortized costs → Finance"
  - Cost categories appear as dimensions in Cost Explorer and CUR — the finance team filters/groups by BU

- **Split charge rules for shared services**:
  - The $80K shared services cost should be allocated based on USAGE, not equally
  - Split charge methods:
    1. **Proportional**: Allocate based on each BU's share of a metric (e.g., data processed through shared VPC). If BU-Alpha processes 60% of VPC data → BU-Alpha gets 60% of networking costs
    2. **Fixed**: Allocate fixed percentages (agreed by BUs). E.g., 60% Alpha, 20% Beta, 15% Gamma, 5% Delta
    3. **Even**: Equal split (current approach — unfair, but simplest)
  - **Usage-based allocation (most accurate)**:
    - Networking: Use VPC Flow Logs aggregated by source account → calculate % of traffic per BU
    - Logging: CloudWatch Logs data volume per BU account → calculate % of logging costs
    - Security: Security Hub findings per account → calculate % of security service utilization

- **Cost Allocation Tags**:
  - Activate `CostCenter`, `Environment`, `Owner`, `Project` as cost allocation tags in the Billing console
  - After activation: these tags appear as filter/group dimensions in Cost Explorer
  - Finance team can generate reports: "Monthly cost where CostCenter = Alpha, grouped by service"
  - **Without activated cost allocation tags, tags exist on resources but are invisible to billing tools**

**Pricing Model Optimization ($119K/month savings):**

- **Dev/staging scheduling ($47K/month savings)**:
  - Current: 45 instances × 24/7 = 168 hours/week each
  - Optimized: Schedule to run 10 hours/day × 5 days/week = 50 hours/week
  - Savings: 1 - (50/168) = 70% reduction → $67K × 0.70 = $47K/month
  - Implementation: **AWS Instance Scheduler** solution (CloudFormation template):
    - Creates Lambda + DynamoDB + CloudWatch Events
    - Tag instances with `Schedule=business-hours`
    - Lambda starts instances at 08:00, stops at 18:00, skips weekends
  - Alternative: ASG scheduled actions to set desired capacity = 0 after hours

- **Production Savings Plans ($43K/month savings)**:
  - 120 production instances, always-on, stable workloads → ideal for Savings Plans
  - **Compute Savings Plans**: 20-30% discount (1-year No Upfront), applies to any EC2, Fargate, Lambda
  - Commit to 80% of production baseline (not 100% — leave room for fluctuation):
    - Production hourly compute cost: $180K / 730 hours = $246.58/hour
    - Commit: $246.58 × 0.80 = $197.26/hour
    - Savings: ~30% on committed amount = $43K/month
  - **Why Compute (not EC2 Instance) Savings Plans**: The company may adopt Graviton instances, Fargate, or Lambda in the future. Compute Savings Plans apply to all of these; EC2 Instance Savings Plans lock you to a specific instance family.

- **Batch processing Spot Instances ($29K/month savings)**:
  - 30 batch instances, flexible timing, interruptible → perfect Spot candidate
  - **Spot Fleet** with diversified instance types and AZs:
    - Request: 30 instances across m5.xlarge, m5a.xlarge, m6i.xlarge, m6g.xlarge (4 families) × 3 AZs = 12 capacity pools
    - If one pool is reclaimed, Fleet shifts to another pool with available capacity
    - Spot pricing: 60-90% off On-Demand. Conservative estimate: 65% savings
    - $45K × 0.65 = $29K/month savings
  - **Spot interruption handling**: Use 2-minute interruption notice to checkpoint batch progress (save state to S3) and resume on a new Spot instance

- **Spiky instances — On-Demand ($0 savings, correct decision)**:
  - Unpredictable ASG workloads can't commit to Savings Plans (risk of over-commitment)
  - These instances scale 0-25 based on demand; any Savings Plan commitment would be wasted during low periods
  - Consider: after 6 months of data, if a baseline is consistent (e.g., min=5 always running), commit that baseline to a Savings Plan

- **Savings summary**:

  | Category | Current | Optimized | Savings | Method |
  |---|---|---|---|---|
  | Production | $180K | $137K | $43K | Compute Savings Plans (1yr) |
  | Dev/Staging | $67K | $20K | $47K | Business hours scheduling |
  | Batch | $45K | $16K | $29K | Spot Instances |
  | Spiky | $38K | $38K | $0 | On-Demand (correct) |
  | **Total compute** | **$330K** | **$211K** | **$119K** | **36% reduction** |

**S3 Cost Control:**

- **S3 Storage Lens**: Organization-wide storage analytics:
  - Dashboard shows: total size per bucket, growth trend, access frequency, % in each storage class
  - Identifies: largest buckets (top 10 by size), fastest growing (% MoM growth), coldest (no access in 90+ days)
  - Advanced metrics: activity-by-prefix (which "folders" in a bucket are hot/cold)
  - Use Storage Lens to **prioritize** which buckets to apply lifecycle policies to first (largest cold buckets = most savings)

- **Intelligent-Tiering**: For buckets where access patterns are unknown:
  - No management overhead — AWS moves objects between tiers automatically
  - 800 TB at Standard = $18,400/month. If 60% transitions to IA (40% savings): saves $4,400/month
  - Zero retrieval fees (unlike S3 IA lifecycle transitions that charge retrieval fees)

- **Lifecycle Policies**: For known patterns:
  - Application logs: Standard (7 days) → IA (30 days) → Glacier Instant Retrieval (90 days) → delete (365 days)
  - Backups: Standard (1 day) → Glacier (30 days) → Deep Archive (365 days) → retain for 7 years
  - Deep Archive = $0.00099/GB/month. 400 TB of cold data at Deep Archive = $400/month vs $9,200/month Standard = $8,800/month savings

### B. Third-party FinOps platform — ❌ Incorrect

- **Cost**: Most FinOps platforms charge 1-3% of AWS spend as their fee. At $500K/month, that's $5K-$15K/month for the platform itself. This reduces the net savings from any optimization.
- **Still requires the same work**: The platform provides dashboards and recommendations, but someone still needs to implement the changes (tagging, scheduling, Savings Plans purchase, lifecycle policies). The tool doesn't optimize automatically.
- **AWS-native tools are sufficient**: CUR + Athena + QuickSight provides the same dashboards. Cost Explorer provides recommendations. Cost Anomaly Detection provides anomaly alerts. These are free or very low cost.
- **When a FinOps platform IS appropriate**: Organizations with $10M+ monthly spend, multi-cloud (AWS + Azure + GCP), or teams that lack the expertise to build AWS-native reporting. At $500K/month on AWS-only, native tools are sufficient.

### C. Custom cost dashboard — ❌ Incorrect

- **Reinventing the wheel**: Cost Explorer, CUR + Athena + QuickSight, and AWS Budgets already provide exactly what a custom dashboard would. Building custom software for cost reporting is itself a cost center.
- **Ongoing maintenance**: A custom React + Lambda + DynamoDB solution requires development, hosting, and maintenance. When AWS changes billing dimensions or adds new services, the custom solution needs updates.
- **Missing critical features**: Cost Anomaly Detection (ML-based) and forecasted budget alerts can't be replicated by a simple Lambda scraper. AWS uses proprietary ML models trained on billing patterns across millions of accounts.
- **When custom dashboards ARE appropriate**: When integrating AWS costs with non-AWS data (internal business metrics, revenue per customer segment, cost per transaction) — overlay AWS cost data with business KPIs.

### D. Enterprise Discount Program (EDP) — ❌ Incorrect

- **EDP provides 5-15% discount** but requires a multi-year spend commitment ($1M+/year typically). At $500K/month ($6M/year), the company might qualify.
- **But EDP doesn't fix the root problems**: 
  - No visibility (still need Budgets + CUR + anomaly detection)
  - No allocation (still need cost categories + tags)
  - No right-sizing (still need Compute Optimizer + scheduling)
  - No S3 management (still need lifecycle policies)
- **EDP stacks with other optimizations**: First optimize (remove waste, right-size, Savings Plans), THEN negotiate an EDP for additional 5-15% on the optimized spend. EDP should be the LAST optimization, not the first.
- **5-15% of $500K = $25-75K/month** sounds good, but optimizing first (Phase 1-4 in option A) saves $119K/month from compute alone + $13K from S3 + quick wins = ~$140K/month. The phased approach saves 2-5× more than an EDP.

## Recommendations

- **Implement CUR + Athena immediately**: CUR is the foundation for all cost analysis. Enable it on day 1. It takes 24 hours for the first report. Athena queries are ready within 48 hours. QuickSight dashboards can be built on top within a week.
- **Cost Anomaly Detection is free (for basic alerting)**: Enable it immediately. It will start learning your patterns and alert on anomalies within 24-48 hours. This alone would have prevented the $40K GPU incident.
- **Right-size before committing**: Never buy Savings Plans for oversized workloads. Right-size first (Compute Optimizer, 2-4 weeks to analyze), then commit the optimized baseline to Savings Plans.
- **Savings Plans → start with 1-year No Upfront**: Lowest risk commitment. After 6-12 months, if workloads are stable, extend to 3-year for deeper discounts. Never start with 3-year All Upfront.
- **S3 Storage Lens → prioritize by size × temperature**: Focus lifecycle policies on the largest, coldest buckets first (max savings per effort). A 50 TB cold bucket transitioning from Standard to Deep Archive saves $1,100/month. Optimizing a 100 MB hot bucket saves $0.

## Relevant Links

- [Cost and Usage Report](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html)
- [Cost Categories](https://docs.aws.amazon.com/cost-management/latest/userguide/manage-cost-categories.html)
- [Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)
- [Savings Plans Recommendations](https://docs.aws.amazon.com/savingsplans/latest/userguide/sp-recommendations.html)
- [S3 Storage Lens](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens.html)
- [Instance Scheduler](https://docs.aws.amazon.com/solutions/latest/instance-scheduler-on-aws/solution-overview.html)
- [Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html)
- [AWS Budgets Actions](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html)
