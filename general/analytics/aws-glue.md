# AWS Glue

## Service Overview
AWS Glue is a serverless ETL service providing job orchestration, a metadata catalog, and data transformation capabilities for building data lakes and pipelines.

## Tasks this service can solve
- ETL/ELT jobs to transform and load data into analytics stores
- Building and managing a centralized Glue Data Catalog
- Scheduling, monitoring and retrying data jobs

## Alternatives
| Name | Type | Short description | Pro vs Glue | Cons vs Glue | Price comparison |
|---|---|---:|---|---|---|
| Amazon EMR | AWS | Managed Spark/Hadoop clusters | More control and tuning | More operational overhead | EMR cluster costs; Glue billed by DPUs per second and can be cheaper for smaller or bursty jobs |
| Airflow on EC2 / MWAA | OSS / AWS | Workflow orchestration | Rich orchestration features | Additional infra and management | MWAA + infra costs; Glue includes ETL runtime built-in |
| Self-managed Spark | OSS | Custom Spark on EC2/k8s | Full control and custom libraries | Heavy ops | Higher ops cost; no per-job serverless billing |

## Limitations
- Cold starts for jobs; cost model based on DPUs (resource allocation) and job duration.
- Glue’s auto-generated code may need tuning for complex transformations.

## Price info
- Charged by DPU-second for ETL jobs, plus Glue Data Catalog storage and requests.

## Network & Multi-region considerations
- Glue is regional. For multi-region pipelines, orchestrate jobs per region and centralize catalog replication where needed.
- Integrates with S3, Redshift, Athena, EMR, and Lake Formation for governance.

## Popular use cases
- ETL into data warehouses and data lakes
- Cataloging datasets for query engines like Athena
- Scheduled nightly data transformations

## When Not to Use This Service
- Not ideal for ultra-low-latency streaming transforms or when you need complete control over cluster tuning; consider EMR or self-managed Spark for heavy, long-running workloads.

## DR strategy
- Maintain job definitions and ETL code in source control; replicate Glue Data Catalog metadata across regions or export it. For critical pipelines, run parallel jobs in another region or use cross-region replication of source data (S3) and orchestrate failover.
