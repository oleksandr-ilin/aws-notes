# Amazon RDS

## Service Overview
Amazon RDS provides managed relational database engines (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server) with automated backups and maintenance.

## Tasks this service can solve
- Lift-and-shift relational applications
- Managed DB maintenance, backups, and patching

## Alternatives
| Name | Type | Short description | Pro vs RDS | Cons vs RDS | Price comparison |
|---|---|---:|---|---|---|
| Aurora | AWS | High-performance distributed relational DB | Higher performance & features | Higher cost at scale | Aurora often more expensive but faster at scale |
| Self-hosted DB on EC2 | OSS/Custom | Full control over DB and tuning | Complete control | Ops overhead | Higher ops cost |

## Limitations
- Vertical scaling limits and engine-specific restrictions; cross-region replicas are limited by engine features.

## Price info
- Charged for instance hours, storage, I/O, backups, and licensing for commercial engines.

## Network & Multi-region considerations
- Use Multi-AZ for HA and read replicas for read scalability; cross-region replicas for DR where supported.

## When Not to Use This Service
- Not ideal for ultra-high scale distributed workloads (consider Aurora or NoSQL solutions for different scale/latency profiles).

## DR strategy
- Use automated Multi-AZ, snapshots, and cross-region read replicas for disaster recovery; test failover and restore regularly.

## Popular use cases
- Traditional relational applications, ERP systems, and legacy app migrations.
