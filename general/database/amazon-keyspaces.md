# Amazon Keyspaces

## Service Overview
Amazon Keyspaces is a scalable, serverless Apache Cassandra-compatible wide-column database for high-write workloads.

## Tasks this service can solve
- Large-scale write-heavy workloads with wide-column access patterns
- Cassandra-compatible apps migrated to AWS

## Alternatives
| Name | Type | Short description | Pro vs Keyspaces | Cons vs Keyspaces | Price comparison |
|---|---|---:|---|---|---|
| Self-managed Cassandra | OSS | Run Cassandra on EC2/k8s | Full control & custom tuning | Ops burden | Higher ops cost; Keyspaces reduces management overhead |
| DynamoDB | AWS | Serverless key-value/document DB | Simpler serverless model | Different data model | DynamoDB costs depend on access patterns; Keyspaces billed by capacity or on-demand |

## Limitations
- Query model constrained by Cassandra data modeling; limited to supported CQL features.

## Price info
- Charged by read/write capacity or on-demand requests plus storage.

## Network & Multi-region considerations
- Regional service; implement application-level replication or multi-region strategies for cross-region resilience.

## When Not to Use This Service
- Not ideal when you need rich secondary indexes or SQL-like ad-hoc queries—consider other DBs or integrate analytics pipelines.

## DR strategy
- Use periodic backups and export tools; maintain replication at application level or use cross-region export to S3.

## Popular use cases
- Time-series ingestion, write-heavy telemetry stores, and Cassandra migration targets.
