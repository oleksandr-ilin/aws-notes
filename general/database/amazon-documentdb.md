# Amazon DocumentDB

## Service Overview
Amazon DocumentDB is a managed document database compatible with MongoDB APIs for JSON document workloads.

## Tasks this service can solve
- Document-model applications requiring MongoDB-like APIs
- Content management and user profile stores

## Alternatives
| Name | Type | Short description | Pro vs DocumentDB | Cons vs DocumentDB | Price comparison |
|---|---|---:|---|---|---|
| MongoDB Atlas | Commercial/OSS | Managed MongoDB with full feature parity | More up-to-date Mongo features | External vendor | Can be costlier depending on plan |
| DynamoDB | AWS | Fully managed key-value/document DB | Scales without capacity planning | Different access patterns and query model | Often cheaper at massive scale for simple patterns |

## Limitations
- Compatibility gaps with latest MongoDB features; version lag may exist.

## Price info
- Charged for instances, storage, I/O, and backups.

## Network & Multi-region considerations
- Use replicas and backups; DocumentDB is regional — implement cross-region replication or backups for DR.

## When Not to Use This Service
- If you need the latest MongoDB features or specific extensions, prefer MongoDB Atlas or self-hosted MongoDB.

## DR strategy
- Use automated snapshots and cross-region backups; maintain cluster replacement runbooks and export critical data regularly.

## Popular use cases
- Content stores, product catalogs, and applications migrated from MongoDB.
