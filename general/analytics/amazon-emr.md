# Amazon EMR

## Service Overview
Amazon EMR is a managed big data platform for running Apache Hadoop, Spark, Presto, and other distributed frameworks on AWS-managed clusters.

## Tasks this service can solve
- Large-scale batch processing and ETL
- Interactive analytics with Presto/Trino
- Custom big-data frameworks requiring cluster control

## Alternatives
| Name | Type | Short description | Pro vs EMR | Cons vs EMR | Price comparison |
|---|---|---:|---|---|---|
| AWS Glue | AWS | Serverless ETL with managed Spark | Serverless, less ops overhead | Less control for custom tuning | EMR can be cheaper for sustained large clusters; Glue cheaper for small/bursty jobs |
| Spark on EKS | OSS/AWS | Run Spark on Kubernetes | Kubernetes-native, portability | More complex setup | Depends on node provisioning; EMR provides optimized AMIs and integrations |
| Databricks | Commercial | Managed Spark platform | Rich UX and optimizations | Higher cost | Databricks licensing + infra; often higher total cost for heavy workloads |

## Limitations
- Cluster startup time and tuning required for performance.
- Managing spot instances and cluster scaling adds orchestration complexity.

## Price info
- Billed for EC2 instances used by the cluster, EMR software add-ons, EBS volumes, and any data transfer.

## Network & Multi-region considerations
- EMR clusters are deployed in a VPC and AZs; for multi-region analytics, run clusters per region and consolidate results via S3 or data pipelines.
- Integrates with S3, Glue, and IAM for data access and security.

## Popular use cases
- Large-scale ETL and data lake processing
- Batch analytics and machine learning model training at scale
- Interactive SQL via Presto/Trino for large datasets

## When Not to Use This Service
- Not appropriate for small, ad-hoc query needs or teams without cluster ops expertise; consider Athena or Glue for serverless alternatives.

## DR strategy
- Persist intermediate and output data to S3 with lifecycle and replication. Automate EMR cluster bootstrapping via templates (CloudFormation/Terraform) and maintain backed-up configuration and scripts to recreate clusters in another region.
