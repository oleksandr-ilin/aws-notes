# Amazon Athena

## Service Overview
Amazon Athena is a serverless interactive query service that lets you run standard SQL against data stored in Amazon S3 without provisioning servers.

## Tasks this service can solve
- Ad-hoc SQL queries over S3-based data lakes
- Schema-on-read exploration and quick reporting
- Data discovery and sampling for ETL planning

## Alternatives
| Name | Type | Short description | Pro vs Athena | Cons vs Athena | Price comparison |
|---|---|---:|---|---|---|
| Amazon Redshift Spectrum | AWS | Query S3 data via Redshift cluster | Better for complex analytics and concurrency | Requires Redshift cluster | Redshift costs + spectrum scanning costs; Athena is serverless per-query pricing |
| Presto on EMR | OSS | Presto queries on EMR clusters | More control over tuning | Cluster ops overhead | EMR + EC2 costs can be higher for small/irregular workloads |
| Google BigQuery | GCP | Serverless analytics on GCP | Strong concurrency & pricing model | Different ecosystem | BigQuery uses per-byte pricing similar to Athena per-GB scanned |

## Limitations
- Costs scale with data scanned; requires partitioning and compression to optimize.
- Not intended for low-latency transactional queries.
- Schema-on-read implies consistency depends on data layout and formats.

## Price info
- Charged per TB scanned (with compression and partitions reducing scanned bytes). Additional costs for result storage and data transfer.

## Network & Multi-region considerations
- Athena is regional. For multi-region queries, replicate data or use cross-region S3 replication and run Athena in each region; aggregate results via ETL or federated queries.
- Integrates with AWS Lake Formation and Glue Data Catalog for permissions and schemas.

## Popular use cases
- Interactive analysis of S3 logs and telemetry
- Ad-hoc BI queries over ingested data
- Quick data investigations during migrations or ETL development

## When Not to Use This Service
- Not suitable for low-latency transactional queries or workloads requiring frequent small updates; use OLTP databases or data stores (RDS, DynamoDB) instead.

## DR strategy
- Athena queries operate against S3; ensure S3 replication or cross-region replication for durable data copies. Catalog and Glue metadata should be replicated or exported; consider cross-region data pipelines for analytics continuity.
