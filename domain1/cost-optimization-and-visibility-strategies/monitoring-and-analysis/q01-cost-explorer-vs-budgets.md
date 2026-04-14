# Q01: Cost Explorer vs. AWS Budgets for Multi-Account Tracking

## Question

A large enterprise has 80 AWS accounts organized under AWS Organizations. The finance team needs to identify unexpected cost increases at the account level on a daily basis and automatically notify account owners when spending exceeds 80% of their monthly allocation. The operations team also needs to visualize cost trends across all accounts over the past 12 months, broken down by service and linked account.

Which combination of AWS services should the solutions architect recommend? **(Select TWO.)**

## Options

- **A.** Use AWS Cost Explorer to visualize historical cost trends by service and linked account over the past 12 months, and create custom reports for the operations team.
- **B.** Configure AWS Budgets with individual budgets per account, set an 80% threshold alert, and use Amazon SNS to notify account owners when the threshold is breached.
- **C.** Enable AWS Trusted Advisor cost optimization checks across all accounts and configure email notifications for cost anomalies.
- **D.** Use Amazon CloudWatch billing alarms on the management account to monitor total organizational spend and send SNS notifications.
- **E.** Deploy AWS Cost and Usage Reports (CUR) to Amazon S3, then use Amazon Athena and Amazon QuickSight to build a custom cost dashboard for the operations team.

## Answers

### A. AWS Cost Explorer — ✅ Correct

AWS Cost Explorer provides built-in visualization of cost and usage data at daily or monthly granularity. It supports filtering and grouping by linked account and service, with up to 12 months of historical data. This directly meets the operations team's requirement to visualize cost trends across all accounts.

### B. AWS Budgets with per-account thresholds — ✅ Correct

AWS Budgets allows you to set custom cost thresholds per account and trigger alerts when spending reaches a specified percentage. Combined with Amazon SNS, it provides automated notifications to account owners when the 80% threshold is breached — exactly matching the finance team's requirement.

### C. AWS Trusted Advisor — ❌ Incorrect

Trusted Advisor provides cost optimization recommendations (e.g., idle resources, underutilized instances), but it does not provide daily spend tracking, threshold-based alerts, or historical cost trend visualization. It's useful for rightsizing but not for the budget monitoring described here.

### D. CloudWatch billing alarms — ❌ Incorrect

CloudWatch billing alarms can monitor total account charges, but they work at the management account level for overall spending. They do not provide per-linked-account granularity or per-account budget thresholds. They also lack the rich trend visualization the operations team needs.

### E. CUR + Athena + QuickSight — ❌ Incorrect

While CUR + Athena + QuickSight can provide deeper custom analysis, it requires significant setup and maintenance. The question asks for visualization of cost trends — Cost Explorer provides this out of the box without the overhead of building a custom pipeline. This combination is more appropriate for highly customized or advanced reporting needs beyond what Cost Explorer offers.

## Recommendations

- Use **AWS Cost Explorer** for quick, built-in cost visualization with up to 12 months of history.
- Use **AWS Budgets** for proactive alerting based on cost or usage thresholds; supports per-account budgets in Organizations.
- Reserve **CUR + Athena + QuickSight** for advanced, custom analysis (e.g., cost allocation by custom tags, cross-account chargeback models).
- CloudWatch billing alarms are simpler but lack per-account granularity in multi-account setups.

## Relevant Links

- [AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Managing AWS Budgets in AWS Organizations](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-create.html)
- [AWS Cost and Usage Reports](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html)
