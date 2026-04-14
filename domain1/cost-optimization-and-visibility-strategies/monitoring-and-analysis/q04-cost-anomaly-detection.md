# Q04: Cost Anomaly Detection vs. Budgets

## Question

A SaaS company deploys microservices across 30 AWS accounts. Each microservice team independently manages its AWS resources. The platform engineering team wants to be alerted about **unexpected** cost spikes that deviate from normal spending patterns — for example, a sudden 300% increase in Lambda invocations due to a retry storm, or an accidental provisioning of expensive instance types. They do NOT want to set static budget thresholds for each service because spending patterns vary significantly between teams and change frequently.

Which approach should the solutions architect recommend?

## Options

- **A.** Create individual AWS Budgets for each account with monthly threshold alerts. Set the budget amount to 120% of the previous month's actual spend, and use AWS Lambda to automatically update budget amounts each month.
- **B.** Configure AWS Cost Anomaly Detection with cost monitors grouped by AWS service and linked account. Set alert preferences to notify the platform team via Amazon SNS when anomalies exceed a minimum impact threshold of $100.
- **C.** Enable Amazon CloudWatch billing metrics with anomaly detection enabled. Create CloudWatch anomaly detection alarms for each account's estimated charges metric.
- **D.** Deploy a custom cost monitoring solution using AWS Cost Explorer API data ingested into Amazon OpenSearch Service, with built-in anomaly detection to identify cost outliers.

## Answers

### B. AWS Cost Anomaly Detection — ✅ Correct

AWS Cost Anomaly Detection uses machine learning to learn each service's and account's normal spending patterns. It automatically identifies unexpected cost increases without requiring static thresholds. You can create cost monitors by linked account, AWS service, cost category, or cost allocation tag. When an anomaly is detected above the minimum impact threshold, alerts are sent via SNS or email. This is exactly what the platform team needs — dynamic, ML-based detection that adapts to changing patterns.

### A. Auto-updating AWS Budgets — ❌ Incorrect

While creative, this approach still uses static thresholds (120% of last month). It won't catch mid-month anomalies that are within the budget but represent abnormal patterns. It also requires custom Lambda code to update budgets monthly, adding operational overhead. Spending patterns that vary significantly make fixed-percentage thresholds unreliable.

### C. CloudWatch billing anomaly detection — ❌ Incorrect

CloudWatch anomaly detection works well for operational metrics, but billing metrics in CloudWatch are limited to total account estimated charges. They don't provide the service-level or resource-level granularity needed to identify a Lambda retry storm or accidental instance provisioning. CloudWatch billing metrics are also only available in us-east-1.

### D. Custom OpenSearch solution — ❌ Incorrect

Building a custom anomaly detection solution on OpenSearch requires significant development effort, ongoing maintenance, and additional infrastructure cost. AWS Cost Anomaly Detection provides this capability as a managed service at no extra charge — there's no reason to build it yourself.

## Recommendations

- Use **Cost Anomaly Detection** for dynamic, pattern-based alerting that adapts to changing usage.
- Use **AWS Budgets** for fixed, known spending limits (e.g., "this account should never exceed $5,000/month").
- Cost Anomaly Detection monitors can be organized by: individual service, linked account, cost allocation tag, or cost category.
- Set a minimum dollar impact threshold to avoid noisy alerts from small fluctuations.
- Combine both tools: Budgets for hard limits, Anomaly Detection for unexpected patterns.

## Relevant Links

- [AWS Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/manage-ad.html)
- [Creating Cost Monitors](https://docs.aws.amazon.com/cost-management/latest/userguide/manage-ad-monitors.html)
- [AWS Budgets vs. Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
