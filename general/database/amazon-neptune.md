# Amazon Neptune

## Service Overview
Amazon Neptune is a managed graph database supporting Gremlin and SPARQL for relationship-centric data.

## Tasks this service can solve
- Graph queries for recommendations, social graphs, and knowledge graphs
- Relationship analytics and connected data queries

## Alternatives
| Name | Type | Short description | Pro vs Neptune | Cons vs Neptune | Price comparison |
|---|---|---:|---|---|---|
| JanusGraph on EC2 | OSS | Graph DB on custom infrastructure | Choice of storage/index backends | Ops overhead | Higher ops cost; Neptune managed simplifies operations |
| Neo4j (self-hosted/managed) | OSS/Commercial | Graph DB with native features | Rich ecosystem and tooling | Licensing or ops cost | Commercial editions add license costs |

## Limitations
- Query patterns and scaling require careful design; not ideal for broad table scans or non-graph workloads.

## Price info
- Billed for instance hours, storage, and backups.

## Network & Multi-region considerations
- Regional service; for DR, use snapshots and cross-region backups or export mechanisms.

## When Not to Use This Service
- Avoid for purely relational or document workloads where RDS or DocumentDB are better fits.

## DR strategy
- Regular snapshots and cross-region backups, plus automated cluster recreation scripts to recover from region-level failures.

## Popular use cases
- Recommendation engines, fraud detection with relationship analysis, and knowledge graphs.
