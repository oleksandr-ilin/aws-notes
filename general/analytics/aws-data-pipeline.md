Overview
AWS Data Pipeline is a legacy ETL/orchestration service for moving and transforming data on schedule.

Tasks
- Define data nodes, activities, and schedules; provision resources to run activities.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| AWS Glue | Serverless | Different model & migration effort

Limitations
- Older service with less modern integrations compared to Glue.

Price info
- Pay for scheduled activities and underlying compute resources.

Network & multi-region
- Configurable per-region; plan access to data stores and S3 endpoints.

Popular use cases
- Legacy scheduled ETL workflows and periodic batch jobs.

When Not to Use
- New projects; prefer Glue or managed Spark on EMR/EKS.

DR strategy
- Store pipeline definitions in source control and maintain runbooks.
