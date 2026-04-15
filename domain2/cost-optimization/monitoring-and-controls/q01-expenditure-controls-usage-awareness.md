# Q01: Expenditure Controls and Usage Awareness for a Multi-Account Organization

## Question

A company has an AWS Organization with 120 accounts across 5 OUs (Production, Development, Staging, Sandbox, Shared Services). Last quarter, three cost incidents occurred:

1. A developer in the Sandbox OU launched 20× p4d.24xlarge instances ($32.77/hour each) for a personal ML experiment, costing $47,000 before anyone noticed after 3 days
2. The Production OU's S3 costs increased 400% over 2 weeks due to an application bug creating duplicate objects — no one noticed until the monthly bill
3. A Staging account accumulated $8,000 in NAT Gateway charges because developers provisioned NAT Gateways in 6 AZs across 3 accounts when only 1 was needed

The CFO requires: (a) prevent unauthorized expensive resource launches, (b) detect anomalous spending within 24 hours, (c) rightsizing recommendations, and (d) chargeback reporting per team.

Which combination of controls achieves all four requirements?

## Options

- **A.** (1) SCP on Sandbox OU denying `ec2:RunInstances` for instance types matching `p4d.*`, `p3.*`, `p5.*`, `trn1.*`, `inf2.*`, and `g5.*`. (2) AWS Cost Anomaly Detection with individual monitors per OU and SNS alerts to a Slack channel via Chatbot. (3) AWS Compute Optimizer enabled organization-wide with opt-in for enhanced metrics. (4) Cost allocation tags (`Team`, `Project`, `Environment`) enforced via SCP that denies resource creation if required tags are missing, with Cost Explorer configured to group by tag.
- **B.** (1) IAM policies on developers' roles denying GPU instance types. (2) AWS Budgets with threshold alerts at 80%, 100%, and 150% of monthly budget per account. (3) Trusted Advisor cost optimization checks. (4) AWS Cost and Usage Report (CUR) exported to S3, queried with Athena, and visualized in QuickSight for chargeback.
- **C.** (1) AWS Service Catalog portfolio restricting available instance types in Sandbox. (2) CloudWatch billing alarm set at $50,000 for the organization. (3) AWS Config rules checking for oversized instances. (4) Custom tagging solution using Lambda triggered by CloudTrail to auto-tag resources.
- **D.** (1) SCP denying all EC2 actions in Sandbox OU. (2) Monthly manual review of Cost Explorer by the FinOps team. (3) Third-party cost optimization tool (e.g., CloudHealth). (4) Spreadsheet-based chargeback using Cost and Usage Report.

## Answers

### A. SCP (instance type deny) + Cost Anomaly Detection + Compute Optimizer + tag enforcement via SCP — ✅ Correct

Each control directly addresses one of the four requirements:

- **(1) SCP on Sandbox OU denying expensive instance types — prevents unauthorized launches**:
  - Service Control Policies are the strongest guardrail in AWS Organizations — they override ALL IAM permissions in the affected OU. Even an account administrator cannot bypass an SCP.
  - Denying `ec2:RunInstances` with a condition on `ec2:InstanceType` matching `p4d.*`, `p3.*`, etc. prevents GPU/ML instances from being launched. Developers can still use general-purpose instances for experimentation.
  - **Why SCP over IAM policies**: IAM policies are per-role/user and can be modified by account admins. A Sandbox account admin could grant themselves GPU access by modifying their own IAM policy. SCPs are managed at the Organization level — only the management account can modify them.
  - Example SCP condition: `"StringLike": {"ec2:InstanceType": ["p4d.*", "p3.*", "p5.*", "trn1.*", "inf2.*", "g5.*"]}` with `"Effect": "Deny"`.

- **(2) Cost Anomaly Detection with per-OU monitors — detects anomalies within 24 hours**:
  - Cost Anomaly Detection uses ML to establish spending baselines and alerts when spending deviates significantly. It detects anomalies daily (or more frequently with AWS Cost Anomaly Detection's near-real-time evaluation).
  - **Per-OU monitors**: Monitoring at the OU level catches the S3 duplicate object bug (Production OU) and the NAT Gateway overprovisioning (Staging OU) — each OU has its own baseline, so relative spikes are detected even if the absolute amount is small relative to the total org spend.
  - **SNS → Chatbot → Slack**: Immediate notification to the team. The S3 400% increase over 2 weeks would trigger an anomaly alert within 1-2 days (not 2 weeks).

- **(3) Compute Optimizer organization-wide — rightsizing recommendations**:
  - Compute Optimizer analyzes CloudWatch metrics for EC2, EBS, Lambda, and ECS to recommend: downsize, upsize, or change instance family.
  - **Enhanced metrics**: Extends the look-back period from 14 days to 3 months for more accurate recommendations (requires CloudWatch agent for memory metrics).
  - **Organization-wide**: The management account enables Compute Optimizer across all accounts — every account gets recommendations without individual setup.
  - Catches issues like 6 NAT Gateways in Staging when 1 is sufficient (though NAT Gateway rightsizing is more of a VPC architecture review — Compute Optimizer focuses on compute resources).

- **(4) Tag enforcement via SCP + Cost Explorer grouping — chargeback per team**:
  - SCP with a `Deny` effect on resource-creating actions (e.g., `ec2:RunInstances`, `s3:CreateBucket`) when required tags (`Team`, `Project`, `Environment`) are not present. Uses the `aws:RequestTag` condition key.
  - **Enforcement at creation time**: Resources cannot be created without tags — eliminates the problem of untagged resources that can't be attributed.
  - Cost Explorer group-by-tag functionality allows the FinOps team to generate per-team cost reports directly — enabling automated chargeback.
  - **Cost allocation tags** must be activated in the Billing console for the tagged values to appear in cost reports.

### B. IAM deny + Budgets alerts + Trusted Advisor + CUR/Athena/QuickSight — ❌ Incorrect

- **IAM policies for instance type deny**: IAM policies can be modified by the account administrator. In a Sandbox OU, developers may have broad permissions (or even admin access). They could remove the restriction from their own role. SCPs cannot be bypassed at the account level.
- **AWS Budgets threshold alerts**: Budgets alert at predefined dollar thresholds (e.g., 80% of $10,000). This is useful for budget tracking but does NOT detect anomalies. If the budget is set at $50,000/month and a $47,000 GPU experiment happens, the 80% alert fires at $40,000 — after most of the damage is done. Cost Anomaly Detection detects pattern deviations regardless of absolute threshold.
- **Trusted Advisor cost checks**: Provides static recommendations (idle ELBs, unassociated EIPs, underutilized EC2) — not dynamic anomaly detection. Checked periodically, not in real-time. Doesn't detect a 400% S3 spike.
- **CUR + Athena + QuickSight for chargeback**: Technically correct and very powerful for detailed chargeback reporting. However, this requires significant setup (CUR export, Athena table definitions, QuickSight dashboards) and doesn't enforce tagging — if resources are untagged, CUR data can't attribute costs to teams. Without tag enforcement, chargeback data has gaps.

### C. Service Catalog + CloudWatch billing alarm + Config rules + Lambda auto-tagging — ❌ Incorrect

- **Service Catalog restricting instance types**: Service Catalog limits what products can be launched, but only if developers use the Service Catalog to launch resources. They can bypass it entirely by using the EC2 console, CLI, or SDK directly. SCP enforcement cannot be bypassed regardless of launch method.
- **Single CloudWatch billing alarm at $50,000**: One alarm for the entire organization at a high threshold. The $8,000 NAT Gateway issue and $47,000 GPU experiment would both need to push total org spend past $50,000 to trigger — by then the damage is done. Also, CloudWatch billing alarms are only available in us-east-1 and are evaluated ~2× per day. Cost Anomaly Detection is more granular and detects relative anomalies (not just absolute thresholds).
- **AWS Config rules for oversized instances**: Config can check compliance (e.g., "no p4d instances running") but it's detective, not preventive. The instance launches, then Config detects it. By the time remediation occurs (even with auto-remediation), costs have accrued. SCPs prevent the launch in the first place (preventive).
- **Lambda auto-tagging via CloudTrail**: Reactive — resources are created untagged, then a Lambda function adds tags after the fact. There's a window (seconds to minutes) where the resource exists without tags. If the Lambda fails, resources remain untagged. SCP tag enforcement prevents untagged resources from being created at all (proactive).

### D. SCP deny all EC2 + monthly manual review + third-party tool + spreadsheet — ❌ Incorrect

- **SCP denying ALL EC2 actions in Sandbox**: Over-broad. Developers need compute resources for experiments — denying all EC2 actions makes the Sandbox unusable. The goal is to prevent expensive instance types, not all instances. Specific instance-type deny conditions are the correct approach.
- **Monthly manual review**: The S3 bug ran for 2 weeks before anyone noticed. Monthly review would catch it 2-4 weeks later. The requirement is detection within 24 hours. Automated anomaly detection is necessary.
- **Third-party tools**: Tools like CloudHealth (now VMware Aria Cost) provide excellent cost management, but introducing a third-party dependency when native AWS services (Cost Anomaly Detection, Compute Optimizer, Cost Explorer) cover all four requirements adds cost and complexity. Valid for mature FinOps organizations, but not the best first step.
- **Spreadsheet chargeback**: Unscalable for 120 accounts. Manual data extraction, reconciliation, and distribution is error-prone and time-consuming. Cost Explorer tag grouping provides this natively.

## Recommendations

- **SCPs are the strongest preventive control** for cost guardrails — use them to deny expensive resource types, restrict Regions, and enforce tagging. They cannot be overridden by any IAM permission within the account.
- **Cost Anomaly Detection** should be enabled on day 1 of any multi-account organization — it's free and provides ML-based anomaly detection with minimal configuration.
- **Tag enforcement at creation time** (via SCP or AWS Config + auto-remediation) is critical for chargeback and cost attribution. Retroactive tagging always has gaps.
- **Cost allocation tags** must be activated in the Billing console — creating tags on resources is not sufficient; they must also be enabled as cost allocation tags to appear in billing reports.
- **Combine preventive (SCP) + detective (Anomaly Detection) + advisory (Compute Optimizer) controls** for defense in depth against cost overruns.

## Relevant Links

- [Service Control Policies (SCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
- [AWS Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/manage-ad.html)
- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
- [Cost Allocation Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Tag Policies in AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies.html)
