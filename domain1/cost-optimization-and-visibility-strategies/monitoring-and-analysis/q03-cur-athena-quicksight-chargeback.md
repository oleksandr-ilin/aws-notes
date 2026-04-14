# Q03: Custom Cost Dashboard with Athena and QuickSight

## Question

A financial services company with 120 AWS accounts needs to build a chargeback model that allocates AWS costs to individual business units based on custom cost allocation tags. The finance team requires hourly cost granularity, the ability to create ad-hoc queries, and interactive dashboards that can be shared with business unit leaders. The data must be retained for 3 years for audit purposes. AWS Cost Explorer's built-in reports do not provide sufficient granularity or customization.

Which approach should the solutions architect recommend?

## Options

- **A.** Enable AWS Cost and Usage Reports (CUR) with hourly granularity, deliver them to an Amazon S3 bucket in the management account, use AWS Glue to catalog the data, query the data with Amazon Athena, and build interactive dashboards with Amazon QuickSight connected to Athena.
- **B.** Enable AWS Cost Explorer with hourly granularity, export the data via the Cost Explorer API to Amazon S3, and build dashboards with Amazon QuickSight connected to S3.
- **C.** Use AWS Budgets Reports to generate weekly cost summaries per business unit, store them in Amazon S3, and build dashboards with Amazon QuickSight.
- **D.** Create an AWS Lambda function that calls the AWS Cost Explorer API hourly, stores results in Amazon DynamoDB, and builds dashboards using Amazon Managed Grafana.

## Answers

### A. CUR + S3 + Glue + Athena + QuickSight — ✅ Correct

AWS Cost and Usage Reports (CUR) provide the most detailed cost and usage data available, including hourly granularity, resource-level IDs, and custom cost allocation tags. Storing CUR in S3 provides cost-effective long-term retention (3+ years). AWS Glue creates a data catalog for serverless querying via Athena. QuickSight provides interactive, shareable dashboards with row-level security for business unit leaders. This is the AWS-recommended approach for custom cost analysis beyond what Cost Explorer offers.

### B. Cost Explorer API export — ❌ Incorrect

Cost Explorer provides daily or monthly granularity, not hourly. The API is also subject to rate limits and pagination constraints, making bulk export for 120 accounts complex and unreliable. Cost Explorer data also has limited retention compared to CUR stored in S3.

### C. AWS Budgets Reports — ❌ Incorrect

Budgets Reports generate periodic cost summaries but do not support hourly granularity, ad-hoc queries, or the custom tag-based breakdown needed for chargeback models. They are designed for budget tracking, not detailed cost analytics.

### D. Lambda + Cost Explorer API + DynamoDB + Grafana — ❌ Incorrect

This approach requires significant custom development and maintenance. Hourly API calls across 120 accounts would hit rate limits. DynamoDB is not optimized for analytical queries, and the solution doesn't leverage the purpose-built CUR pipeline. The ongoing Lambda and DynamoDB costs would also be higher than the serverless CUR + Athena approach.

## Recommendations

- **CUR** is the single most detailed source of AWS billing data. Always use CUR when Cost Explorer doesn't meet granularity or customization needs.
- Enable **cost allocation tags** in the management account — both AWS-generated and user-defined tags are available in CUR.
- Use S3 **lifecycle policies** to manage CUR data retention (e.g., transition to Glacier after 1 year for the 3-year audit requirement).
- Consider the **Cloud Intelligence Dashboards** (CUDOS, CID) — open-source QuickSight dashboards built on CUR that AWS provides as reference implementations.

## Relevant Links

- [AWS Cost and Usage Reports](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html)
- [Setting Up Athena for CUR Queries](https://docs.aws.amazon.com/cur/latest/userguide/cur-query-athena.html)
- [Cost Allocation Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
- [Cloud Intelligence Dashboards](https://www.wellarchitectedlabs.com/cost/200_labs/200_cloud_intelligence/)
- [Amazon QuickSight](https://docs.aws.amazon.com/quicksight/latest/user/welcome.html)
