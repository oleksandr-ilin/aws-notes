# Q05: Trusted Advisor and Pricing Calculator in Cost Optimization

## Question

A company with 15 AWS accounts under AWS Organizations is conducting a cost optimization review. The CFO has asked the solutions architect to:
1. Identify all idle and underutilized resources across the organization (EC2, EBS, RDS, load balancers)
2. Estimate the cost of migrating a 40-server legacy environment currently running on-premises to AWS, comparing Reserved Instances, Savings Plans, and On-Demand pricing
3. Detect security group configurations that expose databases to the public internet — since a recent breach at a competitor was caused by this
4. Provide an actionable monthly report showing top 5 cost-saving opportunities ranked by potential savings

Which combination of AWS tools should the solutions architect use?

## Options

- **A.** Enable AWS Trusted Advisor at the Organization level with a Business or Enterprise Support plan. Use Trusted Advisor's cost optimization checks (idle LBs, underutilized EC2, unassociated EIPs, idle RDS) and security checks (unrestricted security groups) across all 15 accounts. Use the AWS Pricing Calculator to model the on-premises migration by configuring instance types, storage, and comparing Reserved Instance, Savings Plans, and On-Demand pricing scenarios. Schedule Trusted Advisor organizational report in Amazon S3 for the monthly executive summary.
- **B.** Use AWS Compute Optimizer for all resource rightsizing recommendations. Use AWS Cost Explorer forecasting to estimate the on-premises migration cost. Use Amazon Inspector for security group scanning. Generate monthly reports with Cost Explorer API exports.
- **C.** Use AWS Config rules to detect idle resources and open security groups. Use the Simple Monthly Calculator (deprecated) for migration cost modeling. Aggregate findings manually in a spreadsheet for the CFO.
- **D.** Use Amazon CloudWatch dashboards with custom metrics to track resource utilization. Use third-party pricing tools for migration cost comparison. Use GuardDuty for security group auditing. Build a custom Lambda function to generate monthly cost reports.

## Answers

### A. Trusted Advisor (Org-level) + Pricing Calculator + Org report — ✅ Correct

This directly addresses every requirement:

**Trusted Advisor for idle/underutilized resources**:
- **Organizational view**: With Business or Enterprise Support, Trusted Advisor runs checks across all accounts in the Organization.
- **Cost optimization checks** include: idle load balancers (no active connections), underutilized EC2 instances (< 10% average CPU over 14 days), unassociated Elastic IPs, idle RDS instances, and underutilized EBS volumes.
- **Security checks** include: unrestricted security groups (0.0.0.0/0 on sensitive ports like 3306, 5432, 1433) — directly catching the database-to-internet exposure scenario.
- **Organizational report**: Can be scheduled weekly, aggregated into S3, and visualized — perfect for the monthly CFO summary.

**AWS Pricing Calculator for migration estimation**:
- Configure instance types matching on-premises server specs (CPU, memory, storage).
- Create multiple estimates: On-Demand, 1-year/3-year Reserved Instances, Compute/EC2 Instance Savings Plans.
- Compare scenarios side-by-side with monthly and upfront cost breakdowns.
- Shareable estimates can be reviewed by finance teams.

### B. Compute Optimizer + Cost Explorer + Inspector — ❌ Partially incorrect

- **Compute Optimizer** provides rightsizing recommendations for *existing* EC2/EBS/Lambda/ECS but cannot identify **idle** resources (zero-traffic load balancers, unassociated EIPs). It also can't detect security group issues.
- **Cost Explorer** forecasts spending trends for *existing* AWS usage — it cannot model costs for workloads that don't exist yet on AWS (the on-premises migration).
- **Amazon Inspector** scans for software vulnerabilities (CVEs) and network reachability, but its primary focus is not security group auditing in the same way Trusted Advisor's checks flag unrestricted ports.
- This combination lacks idle resource detection and migration cost modeling.

### C. Config rules + deprecated calculator — ❌ Incorrect

- **AWS Config** can detect open security groups (via managed rules like `restricted-common-ports`), but it requires custom rules or Lambda evaluations for idle resource detection — it doesn't natively check utilization metrics.
- The **Simple Monthly Calculator** is **deprecated** — replaced by the AWS Pricing Calculator. Relying on a deprecated tool is not an acceptable recommendation.
- Manual spreadsheet aggregation doesn't scale for 15 accounts and doesn't provide automated monthly reporting.

### D. CloudWatch + third-party tools + GuardDuty — ❌ Incorrect

- **CloudWatch** provides raw metrics but does not automatically identify idle or underutilized resources — you'd need to build custom dashboards and thresholds for every service, which is exactly what Trusted Advisor provides out of the box.
- **Third-party pricing tools** are unnecessary when AWS Pricing Calculator provides native, always-up-to-date pricing for all instance types and commitment options.
- **GuardDuty** detects threats and anomalous behavior, not insecure security group configurations. It might detect exploitation of an open port but won't proactively flag the misconfiguration.
- Custom Lambda for reporting adds maintenance burden; Trusted Advisor organizational reports provide this natively.

## Recommendations

- **Trusted Advisor** requires **Business or Enterprise Support** for full check access. Basic/Developer plans only get 6 core checks (security-group-related and S3 bucket permissions are included in the free tier).
- **Trusted Advisor Organizational view** requires: (1) Organizations enabled, (2) Business/Enterprise Support on every account you want to check, (3) Trusted Advisor organizational view enabled in the management account.
- **AWS Pricing Calculator** (calculator.aws) replaced the deprecated Simple Monthly Calculator — always reference the current tool.
- Combine **Trusted Advisor** (broad idle/security checks) with **Compute Optimizer** (deep rightsizing ML recommendations) for comprehensive optimization.
- For ongoing cost optimization governance, use **Trusted Advisor + Cost Explorer + Budgets**: Trusted Advisor identifies waste, Cost Explorer tracks trends, and Budgets enforce spending limits with alerts.

## Relevant Links

- [AWS Trusted Advisor](https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor.html)
- [Trusted Advisor Organizational View](https://docs.aws.amazon.com/awssupport/latest/user/organizational-view.html)
- [AWS Pricing Calculator](https://calculator.aws/about)
- [Trusted Advisor Check Reference](https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor-check-reference.html)
- [Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
