# Q02: Multi-Region Active-Active with DynamoDB Global Tables and Global Accelerator

## Question

A global gaming company needs to deploy a multiplayer leaderboard and player profile service across us-east-1, eu-west-1, and ap-northeast-1. Requirements:

- Players must connect to the nearest Region with < 50 ms latency
- Player profiles must be readable and writable from ANY Region
- Leaderboard writes must be strongly consistent within a Region and eventually consistent across Regions (< 1 second replication)
- If one Region fails, players in that Region must be automatically rerouted to the next nearest Region within 30 seconds
- The solution must handle "last writer wins" conflicts for player profile updates (rare — players typically play from one Region)

Which architecture achieves all requirements?

## Options

- **A.** API Gateway (Regional endpoints) in each Region → Lambda → DynamoDB Global Tables (version 2019.11.21). AWS Global Accelerator with endpoint groups in each Region and health checks. DynamoDB Global Tables handle multi-Region replication with last-writer-wins conflict resolution. Each Region's Lambda reads from and writes to the local DynamoDB table.
- **B.** Single API Gateway (edge-optimized) → Lambda in us-east-1 → DynamoDB in us-east-1 with DynamoDB Streams replicating to eu-west-1 and ap-northeast-1 via Lambda. CloudFront for global low-latency routing. Manual DNS failover via Route 53 if us-east-1 fails.
- **C.** ALB in each Region → ECS Fargate → Aurora Global Database (PostgreSQL) with write forwarding. Route 53 latency-based routing. Aurora handles multi-Region replication and conflict resolution.
- **D.** AppSync with multi-Region deployment → DynamoDB Global Tables. CloudFront with Lambda@Edge for routing. Each Region has an AppSync API backed by the local DynamoDB Global Table replica.

## Answers

### A. Regional API Gateway + Lambda + DynamoDB Global Tables + Global Accelerator — ✅ Correct

This architecture provides true active-active multi-Region with the simplest operational model:

- **DynamoDB Global Tables (version 2019.11.21)**:
  - Global Tables replicate data across all specified Regions automatically. Each Region has a full read/write replica — no primary/secondary distinction.
  - **Replication latency**: Typically < 1 second across Regions. Each write in one Region is replicated to all other Regions asynchronously. Within a Region, reads are strongly consistent if using `ConsistentRead=true`.
  - **Last-writer-wins conflict resolution**: If the same item is written in two Regions simultaneously, the write with the later timestamp wins. This is built into Global Tables — no application logic needed. For player profiles (rarely written from multiple Regions), this is an acceptable semantic.
  - **No manual replication setup**: Just add Regions to the Global Table — AWS handles replication, consistency, and conflict resolution.

- **AWS Global Accelerator**:
  - Provides static anycast IP addresses. Player traffic enters the AWS global network at the nearest edge location and is routed to the optimal Region endpoint via AWS backbone (not the public internet).
  - **< 50 ms latency**: Global Accelerator reduces first-mile latency by entering the AWS network early. Combined with Regional API Gateway, total request latency stays well under 50 ms for in-Region requests.
  - **30-second failover**: Global Accelerator health checks endpoints every 10 seconds with a 3-check threshold. If a Region's health check fails, traffic is rerouted to the next nearest healthy Region within ~30 seconds. This is faster than DNS-based failover (Route 53 has 60-second TTL minimum + propagation).
  - **Unlike CloudFront**: Global Accelerator routes to application endpoints (ALB, NLB, EC2, EIP) — not cached content. It's designed for dynamic API traffic (gaming, real-time applications).

- **Regional API Gateway + Lambda per Region**:
  - Each Region has its own API Gateway (Regional endpoint type) and Lambda functions. Requests are processed locally — no cross-Region API calls.
  - Lambda functions read/write to the local DynamoDB Global Table replica. All data is local to the Region, providing consistent low latency.

### B. Single-Region with DynamoDB Streams replication — ❌ Incorrect

- **Single API Gateway edge-optimized in us-east-1**: All writes go to us-east-1 regardless of player location. A player in Tokyo has 150-200 ms latency to us-east-1 — far above the 50 ms requirement. Edge-optimized API Gateway caches API infrastructure at CloudFront edge locations but still routes requests to the us-east-1 backend.
- **DynamoDB Streams + Lambda replication**: Custom replication via Streams requires writing and maintaining replication Lambda functions, handling failures, managing ordering, and dealing with conflicts. DynamoDB Global Tables does all of this natively with zero custom code.
- **Manual DNS failover**: If us-east-1 fails, manual Route 53 update requires human intervention + DNS propagation delay. This doesn't meet the 30-second automatic failover requirement.
- **Single-Region design is fundamentally not active-active** — it's a single point of failure with eventual replication.

### C. ALB + ECS + Aurora Global Database — ❌ Incorrect

- **Aurora Global Database with write forwarding**: Aurora Global Database has ONE writable primary Region. All other Regions are read-only. "Write forwarding" sends writes from secondary Regions to the primary via the network — adding cross-Region latency (100-200 ms) to every write. This violates the "writable from ANY Region" with low latency requirement.
  - In contrast, DynamoDB Global Tables allow direct writes to any Region's local replica — no cross-Region write forwarding needed.
- **Aurora for leaderboard/profile data**: Leaderboard data (player ID, score, timestamp) and player profiles (key-value lookups) are access patterns that DynamoDB handles more efficiently than a relational database. Aurora's strength is complex queries with joins — unnecessary here.
- **Route 53 latency-based routing**: Provides Region selection based on latency, but Route 53 failover depends on DNS TTL (minimum 60 seconds) + health check interval. Global Accelerator provides faster failover (30 seconds) via anycast IP routing.
- **ECS Fargate vs. Lambda**: For a stateless API with variable traffic (gaming has spiky patterns — tournaments, peak hours), Lambda auto-scales to zero and handles bursts. ECS Fargate requires maintaining minimum task counts in all 3 Regions — more expensive at low traffic.

### D. AppSync + DynamoDB Global Tables + CloudFront — ❌ Incorrect

- **AppSync multi-Region**: AppSync is a managed GraphQL service and is Regional — there's no native multi-Region AppSync deployment with unified routing. You'd need to deploy separate AppSync APIs in each Region and manage routing externally.
- **CloudFront with Lambda@Edge for routing**: CloudFront is designed for content delivery (caching), not dynamic API routing. Lambda@Edge has a 5-second timeout (viewer-facing) and limited runtime — not suitable for routing logic that needs to evaluate Region health and latency.
- **DynamoDB Global Tables portion is correct** but the routing layer (CloudFront + Lambda@Edge) is unsuitable for gaming API traffic that requires low-latency, non-cached, bi-directional communication.

## Recommendations

- **DynamoDB Global Tables** is the simplest multi-Region active-active data layer — zero replication code, built-in conflict resolution, < 1 second replication.
- **Global Accelerator** for multi-Region active-active routing — faster failover than DNS (30 sec vs. 60+ sec), static IPs, anycast routing, TCP/UDP support (gaming often uses UDP).
- **Global Accelerator vs. CloudFront**: Use Global Accelerator for dynamic API/gaming traffic. Use CloudFront for cacheable content (static assets, media). They can coexist — CloudFront for static, Global Accelerator for API.
- **Last-writer-wins** is acceptable when conflicts are rare. For applications requiring stricter conflict resolution (financial transactions), use conditional writes (DynamoDB `ConditionExpression`) or a single-writer-Region pattern.
- **Cost tip**: DynamoDB Global Tables charge for replicated write units in addition to local write units. Budget for 2× (2 Regions) or 3× (3 Regions) write capacity vs. single-Region.

## Relevant Links

- [DynamoDB Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)
- [Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)
- [Global Accelerator Health Checks](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-health-check-options.html)
- [DynamoDB Global Tables Conflict Resolution](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables_HowItWorks.html)
