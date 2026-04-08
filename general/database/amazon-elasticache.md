# Amazon ElastiCache

## Service Overview
Amazon ElastiCache is a managed in-memory data store service supporting Redis and Memcached for caching and fast data access.

## Tasks this service can solve
- Caching frequently accessed data to reduce DB load
- Session stores and leaderboards requiring low latency

## Alternatives
| Name | Type | Short description | Pro vs ElastiCache | Cons vs ElastiCache | Price comparison |
|---|---|---:|---|---|---|
| Self-managed Redis on EC2 | OSS | Run Redis on VMs or k8s | Full control | Ops overhead | Higher ops cost; ElastiCache managed fees apply
| DynamoDB + DAX | AWS | Managed cache for DynamoDB | Integrated with DynamoDB | Different access patterns | DAX adds cost; ElastiCache flexible for many backends |

## Limitations
- In-memory stores need replication/backups; node failover and eviction policies require careful tuning.

## Price info
- Charged per node-hour and memory; clustering and replicas increase costs.

## Network & Multi-region considerations
- ElastiCache is regional and AZ-scoped; for multi-region, replicate data at application layer or use cross-region replication solutions where applicable.

## When Not to Use This Service
- Not suitable for large persistent datasets; prefer persistent stores (RDS, S3, DynamoDB) for durable data.

## DR strategy
- Use Multi-AZ replication, automated snapshots for Redis, and have warm standby clusters in other regions if critical.

## Popular use cases
- Web session caching, caching DB query results, and real-time leaderboards.
