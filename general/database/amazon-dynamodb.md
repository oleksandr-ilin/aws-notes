# Amazon DynamoDB

## Service Overview
Amazon DynamoDB is a fully managed, serverless key-value and document database offering single-digit millisecond performance at any scale.

## Tasks this service can solve
- High-scale key-value workloads, session stores, leaderboards
- Serverless backends requiring predictable low-latency reads/writes

## Alternatives
| Name | Type | Short description | Pro vs DynamoDB | Cons vs DynamoDB | Price comparison |
|---|---|---:|---|---|---|
| Amazon RDS | AWS | Relational DBs | Rich queryability and joins | Scaling complexity | RDS better for relational needs; DynamoDB cheaper for massive scale key-value patterns |
| Cassandra / Keyspaces | OSS/AWS | Wide-column store | Tunable consistency & wide-column model | Different query model | Cost varies; DynamoDB managed serverless model reduces ops costs |

## Limitations
- Item size limits (400 KB), query expressiveness is limited compared to SQL, and cost can grow with poorly designed access patterns.

## Price info
- On-demand or provisioned capacity with read/write units, plus storage and optional DAX/Streams charges.

## Network & Multi-region considerations
- Supports global tables for multi-region replication; design for eventual consistency and consider cross-region latency for single-writer patterns.

## When Not to Use This Service
- Avoid for complex relational queries, ad-hoc analytics, or large item sizes beyond limits — use RDS/Redshift or S3+Athena for analytics.

## DR strategy
- Use DynamoDB global tables for multi-region DR, enable continuous backups, and export periodic snapshots to S3 for archival and recovery.

## Popular use cases
- Mobile backends, gaming leaderboards, IoT device registries, and serverless web apps.
