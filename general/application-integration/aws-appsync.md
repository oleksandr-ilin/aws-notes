# AWS AppSync

## Service Overview
AWS AppSync is a managed GraphQL service that simplifies building APIs with real-time updates and offline sync.

## Tasks this service can solve
- GraphQL APIs for mobile and web apps
- Real-time subscriptions and delta syncs
- Offline-aware data sync for devices

## Alternatives
| Name | Type | Short description | Pro vs AppSync | Cons vs AppSync | Price comparison |
|---|---|---:|---|---|---|
| API Gateway + Lambda | AWS | REST or custom GraphQL server | More control over resolvers | More glue code | Can be cheaper for simple REST APIs; AppSync optimized for GraphQL features |
| Apollo Server on ECS/EKS | OSS/AWS | Self-managed GraphQL servers | Full control & ecosystem | Ops overhead | EC2/ECS costs vs AppSync per-request pricing |
| Hasura | OSS/Commercial | Instant GraphQL on DB | Fast dev velocity | Different auth/integration model | Self-host or commercial licensing |

## Limitations
- Resolver complexity and VTL templates can be challenging at scale.
- Pricing for real-time connections and resolver executions can grow with high usage.

## Price info
- Per-request and per-connection charges for real-time features; data transfer costs apply.

## Network & Multi-region considerations
- AppSync is regional. For multi-region, deploy API endpoints per region and use global DNS or edge routing; manage schema sync across regions.
- Integrates with DynamoDB, Lambda, Elasticsearch/OpenSearch, and HTTP data sources.

## Popular use cases
- Mobile apps needing offline sync
- Aggregated APIs combining multiple backends
- Real-time dashboards and collaboration apps

## When Not to Use This Service
- Not ideal for extremely high-throughput, low-latency APIs where a custom REST/gRPC stack or API Gateway + Lambda/ECS is better suited. Complex resolver logic may be harder to manage.

## DR strategy
- Deploy AppSync APIs in multiple regions with synchronized schema and backends; back up schema and resolver definitions in source control. For data, use globally replicated stores (DynamoDB global tables, S3 replication) and multi-region endpoints.
