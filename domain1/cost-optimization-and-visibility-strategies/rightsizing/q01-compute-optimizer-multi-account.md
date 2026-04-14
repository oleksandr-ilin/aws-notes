# Q01: AWS Compute Optimizer for Multi-Account Rightsizing

## Question

An enterprise operates 500+ EC2 instances across 40 AWS accounts. The cloud operations team suspects many instances are over-provisioned, leading to an estimated $200,000/year in unnecessary spending. They need a solution that analyzes actual CPU, memory, and network utilization patterns across all accounts and provides specific instance type recommendations — including downsizing and changing instance families — with projected cost savings.

Which approach should the solutions architect recommend?

## Options

- **A.** Enable AWS Compute Optimizer at the organization level from the management account. Opt in all member accounts and enable enhanced infrastructure metrics for instances that need longer analysis periods. Review Compute Optimizer's recommendations dashboard for rightsizing opportunities.
- **B.** Deploy Amazon CloudWatch agents on all EC2 instances to collect custom metrics (CPU, memory, disk, network). Build a custom analytics pipeline using Amazon Kinesis Data Firehose, Amazon S3, and Amazon Athena to analyze utilization patterns and generate rightsizing recommendations.
- **C.** Use AWS Trusted Advisor's cost optimization checks to identify underutilized EC2 instances. Export the recommendations to a spreadsheet and manually map each instance to a more appropriate instance type.
- **D.** Enable AWS Cost Explorer's rightsizing recommendations. Review the recommendations for each account individually and implement the suggested changes.

## Answers

### A. AWS Compute Optimizer (org-level) — ✅ Correct

AWS Compute Optimizer uses machine learning to analyze actual utilization metrics (CPU, memory via CloudWatch agent, network, disk) and recommends optimal AWS resources. Key advantages:
- **Organization-level opt-in** from the management account covers all 40 accounts without per-account configuration.
- Recommends specific instance types, including **cross-family** recommendations (e.g., c5.xlarge → m5.large).
- **Enhanced infrastructure metrics** extends the analysis from 14 days to up to 93 days for seasonal workloads.
- Provides projected **cost impact** and **performance risk** for each recommendation.
- Also covers Lambda functions, EBS volumes, ECS on Fargate, and Auto Scaling groups.

### B. Custom CloudWatch analytics pipeline — ❌ Incorrect

While this approach would work, it requires significant engineering effort to build and maintain: deploying CloudWatch agents, building the ingestion pipeline, creating the analysis logic, and mapping utilization data to instance type recommendations. Compute Optimizer provides all of this as a managed service at no additional cost (basic) or minimal cost (enhanced metrics).

### C. Trusted Advisor — ❌ Incorrect

Trusted Advisor identifies "low utilization" instances (CPU < 10% for 14 days) but provides limited recommendation detail — it flags instances as underutilized without recommending specific replacement types. It also doesn't analyze memory utilization or cross-family opportunities. Compute Optimizer provides much richer recommendations.

### D. Cost Explorer rightsizing — ❌ Incorrect

Cost Explorer's rightsizing recommendations are based on basic CloudWatch metrics and only suggest same-family downsizing (e.g., c5.2xlarge → c5.xlarge). They don't recommend cross-family changes or analyze memory utilization. For 500+ instances across 40 accounts, per-account review would be extremely time-consuming. Compute Optimizer provides superior recommendations with organization-wide visibility.

## Recommendations

- Enable **Compute Optimizer at the organization level** for centralized rightsizing across all accounts.
- Deploy the **CloudWatch agent** on instances to provide memory and disk metrics to Compute Optimizer for more accurate recommendations.
- Enable **enhanced infrastructure metrics** (93-day lookback) for workloads with weekly or monthly usage patterns.
- Compute Optimizer is **free** for basic recommendations; enhanced metrics have a small per-instance charge.
- Combine Compute Optimizer with **AWS Systems Manager** to automate instance type changes across the fleet.

## Relevant Links

- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
- [Opting In at the Organization Level](https://docs.aws.amazon.com/compute-optimizer/latest/ug/getting-started.html#account-opt-in)
- [Enhanced Infrastructure Metrics](https://docs.aws.amazon.com/compute-optimizer/latest/ug/enhanced-infrastructure-metrics.html)
- [Supported Resources](https://docs.aws.amazon.com/compute-optimizer/latest/ug/supported-resources.html)
