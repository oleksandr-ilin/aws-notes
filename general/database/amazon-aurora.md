# Amazon Aurora

## Service Overview
Amazon Aurora is a MySQL- and PostgreSQL-compatible relational database built for the cloud with distributed storage and high performance.

## Tasks this service can solve
- High-throughput OLTP relational workloads
- Read replicas and read-scaling for web apps
- Managed relational features with automated backups and failover

## Alternatives
| Name | Type | Short description | Pro vs Aurora | Cons vs Aurora | Price comparison |
|---|---|---:|---|---|---|
| Amazon RDS | AWS | Managed MySQL/Postgres engines | Simpler, cheaper at small scale | Less performance at scale | RDS often cheaper for small instances |
| Self-hosted DB on EC2 | OSS/Custom | Full control over DB and tuning | Full control & extensions | Ops and HA complexity | Higher operational cost |
| Google Cloud SQL | GCP | Managed relational DB on GCP | Multi-cloud parity | Different ecosystem | Comparable pricing across clouds |

## Limitations
- Regional service features and replication limits; certain advanced extensions may not be supported.

## Price info
- Charged for instance hours, storage, I/O, and backups; Serverless variant has ACU-based pricing.

## Network & Multi-region considerations
- Aurora supports Multi-AZ replicas and Aurora Global Database for cross-region read scaling and fast recovery; plan topology for failover and replication lag.

## When Not to Use This Service
- Not ideal for tiny single-node low-cost workloads where standard RDS or a single EC2-hosted DB is cheaper; for non-relational or massive scale key-value needs use DynamoDB.

## DR strategy
- Use automated backups, snapshots, cross-region read replicas or Aurora Global Database for DR. Regularly test failover and restore procedures.

## Popular use cases
- High-traffic transactional web applications, SaaS control planes, and read-heavy relational workloads.
