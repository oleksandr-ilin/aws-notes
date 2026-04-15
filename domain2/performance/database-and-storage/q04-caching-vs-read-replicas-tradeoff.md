# Q04: Database Caching vs Read Replicas — Trade-off Analysis

## Question

A company runs a customer support platform with two database access patterns that need performance optimization:

**Pattern 1 — Knowledge Base search (Aurora PostgreSQL)**
- Support agents search a 2 million article knowledge base
- 80% of queries are the same 500 searches repeated thousands of times per day (e.g., "reset password", "billing dispute", "cancel subscription")
- Articles are updated weekly by the content team
- Current: 8,000 queries/second against the Aurora writer — P95 latency is 180 ms (target: < 50 ms)
- Available budget: $2,000/month for performance improvement

**Pattern 2 — Customer interaction history (Aurora MySQL)**
- Agents view a customer's full interaction history (tickets, calls, chats) during live support calls
- Each query joins 5 tables (customers, tickets, interactions, agents, satisfaction_scores) and returns unique results — no two agents query the same customer simultaneously
- Data changes constantly — new interactions are written every second as agents handle calls
- Current: 3,000 queries/second against the writer — P95 latency is 250 ms (target: < 100 ms)

Which performance optimization strategy is correct for each pattern?

## Options

- **A.** Pattern 1: **ElastiCache Redis** (cache-aside pattern) — cache the top 500 query results with 24-hour TTL. Cache hit rate will be ~80% (repeated queries return cached results in < 1 ms). Cache misses hit Aurora (~180 ms). Blended P95: ~35 ms. Cost: r6g.large Redis node = ~$200/month. Pattern 2: **Aurora read replicas** (2 replicas) — route read queries to the reader endpoint. Each replica handles 1,500 queries/sec. Replicas execute the full 5-table join query (~100 ms). No stale data concern — replication lag < 20 ms for Aurora. Cost: 2 × db.r6g.xlarge = ~$1,400/month.
- **B.** Pattern 1: Aurora read replicas (3 replicas). Pattern 2: ElastiCache Redis caching.
- **C.** Pattern 1: DAX (DynamoDB Accelerator) for caching. Pattern 2: Aurora parallel query.
- **D.** Pattern 1: Aurora Serverless v2 with more ACUs. Pattern 2: Increase Aurora writer instance size from r6g.xlarge to r6g.4xlarge.

## Answers

### A. ElastiCache cache for repeated queries + read replicas for unique queries — ✅ Correct

This answer applies the correct optimization pattern for each access pattern:

**Pattern 1 — Why caching is the right choice:**

- **High query repetition = high cache hit rate**:
  - 80% of queries are the same 500 searches. Once cached, these queries return from Redis in < 1 ms (vs. 180 ms from Aurora). The cache "absorbs" 80% of traffic.
  - **Cache hit math**: 8,000 queries/sec × 80% hit rate = 6,400 from cache (< 1 ms) + 1,600 from Aurora (180 ms). P95 latency drops from 180 ms to < 50 ms (most requests are cache hits).
  - Without caching: 8,000 queries/sec all hit Aurora. Even with 3 read replicas (~2,700 queries each), each query still takes 180 ms — replicas don't make individual queries faster, they distribute the load.

- **Infrequent data changes = long TTL**:
  - Articles update weekly. A 24-hour TTL means cached results are at most 24 hours stale — acceptable for a knowledge base. When the content team publishes updates, cache invalidation clears stale entries immediately (or TTL expires naturally).
  - If data changed every second (like Pattern 2), a cache would serve stale results constantly — requiring very short TTLs that reduce hit rates.

- **Cache-aside pattern**:
  1. Application checks Redis for the query result
  2. **Cache hit**: Return result (< 1 ms) — Aurora is never touched
  3. **Cache miss**: Query Aurora (180 ms), store result in Redis with TTL, return to user
  4. Subsequent identical queries return from cache until TTL expires

- **Cost-effectiveness**: A single r6g.large Redis node (~$200/month) handles the entire caching workload. 3 Aurora read replicas (to achieve similar throughput reduction) would cost ~$2,100/month — 10x more expensive for this pattern.

**Pattern 2 — Why read replicas are the right choice:**

- **Unique queries = zero cache hit rate**:
  - Each query is for a different customer — no two agents query the same customer at the same time. A cache would store query results that are never requested again. Cache hit rate ≈ 0%.
  - Caching is only effective when the same query is repeated. For unique queries, a cache adds latency (cache miss check: Redis lookup → miss → Aurora query → cache write) without any benefit.

- **Frequently changing data = stale cache**:
  - Interactions are written every second. A cached result for customer-123 becomes stale within seconds as new interactions are added. Short TTL (5 seconds) reduces staleness but also reduces hit rate. With unique queries AND short TTLs, caching provides almost no benefit.

- **Read replicas distribute load, not reduce query time**:
  - 2 read replicas split the 3,000 queries/sec load: each replica handles ~1,500 queries/sec. This reduces per-instance CPU utilization, allowing each query to execute faster (less resource contention).
  - **Aurora replication lag**: Aurora replicates data to read replicas within the same storage layer — typical replication lag is < 20 ms (often < 10 ms). For support agents viewing interaction history, 20 ms lag is imperceptible and doesn't affect data accuracy.

- **Read replicas execute the full query**:
  - Unlike a cache (which stores pre-computed results), read replicas execute the actual 5-table join query. This means they return the most up-to-date data and handle complex query logic correctly.
  - Each query takes ~100 ms on a replica (reduced from 250 ms due to less contention on the writer) — meets the < 100 ms target.
  - The writer, now offloaded from read traffic, focuses on writes — improving write latency too.

- **Cost**: 2 × db.r6g.xlarge replicas ≈ $1,400/month. Total (cache + replicas): $1,600/month — within the $2,000 budget.

**When to use cache vs. read replicas — decision framework:**

| Factor | Favors Cache | Favors Read Replicas |
|---|---|---|
| Query repetition | High (same queries repeated) | Low (unique queries) |
| Data change frequency | Low (hourly/daily updates) | High (real-time writes) |
| Staleness tolerance | Tolerant (seconds/minutes OK) | Intolerant (must be current) |
| Query complexity | Simple key-value lookups | Complex joins, aggregations |
| Cost sensitivity | Budget-constrained (cache is cheaper) | Performance-critical (replicas more predictable) |
| Latency target | Sub-millisecond needed | Sub-100ms acceptable |

### B. Read replicas for Pattern 1 + cache for Pattern 2 — ❌ Incorrect

This is the **exact opposite** of the correct strategy:

- **Read replicas for Pattern 1 (repeated queries)**: 3 Aurora read replicas handle 8,000 queries/sec (2,700 each). Each query still takes ~180 ms (replicas have the same hardware, execute the same query). P95 latency: still ~180 ms. Replicas reduce load but don't speed up individual queries. The target is < 50 ms — only caching achieves sub-millisecond response for repeated queries.
  - Cost: 3 × db.r6g.xlarge = ~$2,100/month — exceeds budget alone.

- **Cache for Pattern 2 (unique queries)**: Cache hit rate ≈ 0% (unique customer queries). Every request is a cache miss: check cache (2 ms) → miss → query Aurora (250 ms) → write to cache (2 ms). Total: 254 ms — worse than without caching. The cached results are evicted (never read again) before anyone requests them.
  - Stale data problem: Even if a query IS a cache hit (unlikely), the cached interaction history is stale — missing interactions added since caching. An agent sees incomplete customer history — a serious support quality issue.

### C. DAX + Aurora parallel query — ❌ Incorrect

- **DAX for Pattern 1**: DAX is DynamoDB Accelerator — it caches DynamoDB results, not Aurora PostgreSQL results. The knowledge base is on Aurora PostgreSQL. DAX cannot be used with Aurora. ElastiCache Redis is the correct caching service for Aurora.
  - Common exam trap: DAX sounds like "database cache" but it's exclusively for DynamoDB.

- **Aurora parallel query for Pattern 2**: Aurora parallel query pushes query processing to the storage layer, parallelizing complex analytical queries across storage nodes. It's designed for:
  - Large table scans and aggregations (OLAP-style queries)
  - Queries that scan millions of rows
  - It does NOT help with indexed lookups (OLTP-style), which the customer interaction query likely uses (lookup by customer_id with indexed joins).
  - Parallel query adds overhead for small, indexed queries — making them slower, not faster.
  - Parallel query is available for Aurora MySQL only (not PostgreSQL) and requires specific engine versions.

### D. Aurora Serverless v2 + writer instance upsizing — ❌ Incorrect

- **Aurora Serverless v2 for Pattern 1**: Serverless v2 auto-scales ACUs (Aurora Capacity Units) based on demand. More ACUs = more CPU and memory. However:
  - Scaling ACUs doesn't reduce query time for repeated queries — the same query still executes in ~180 ms. It only prevents CPU saturation at high concurrency.
  - At 8,000 queries/sec, Serverless v2 would scale to many ACUs — costing significantly more than a cache ($0.12/ACU-hour × 100+ ACUs = $8,640+/month vs. $200/month for Redis).
  - Caching eliminates 80% of queries entirely — Serverless v2 still processes all queries, just with more compute.

- **Upsizing the writer for Pattern 2**: Upgrading from r6g.xlarge (4 vCPU, 32 GB) to r6g.4xlarge (16 vCPU, 128 GB) provides more CPU for query processing. But:
  - A 4x larger instance costs 4x more (~$2,800/month additional). 2 read replicas ($1,400/month) are cheaper and provide better fault tolerance (if one replica fails, the other handles all reads).
  - Upsizing is a vertical scaling approach — it has a ceiling (largest instance). Read replicas are horizontal scaling — add more replicas as traffic grows.
  - Upsizing doesn't offload reads from the writer — the writer still handles both reads and writes. Read replicas separate read and write traffic.

## Recommendations

- **Default decision**: Start with ElastiCache caching for read-heavy, repeated query patterns. Add read replicas when caching can't help (unique queries, complex joins, real-time data).
- **Combine cache + replicas**: For some workloads, use both — ElastiCache for hot/repeated queries + read replicas for unique/complex queries. Route traffic based on query type.
- **Cache key design**: Use the normalized SQL query as the cache key (or a hash of it). Ensure query parameters generate unique keys: `knowledgebase:search:reset_password` vs. `knowledgebase:search:billing_dispute`.
- **Cache invalidation**: Prefer TTL-based expiration for simplicity. Use explicit invalidation (delete cache keys) only when data changes are critical to reflect immediately.
- **Aurora reader endpoint**: Routes read queries across all available replicas using DNS round-robin. Use the reader endpoint in your application — no custom load balancing needed.
- **Monitor cache effectiveness**: Track `CacheHitRate` in CloudWatch (ElastiCache metric). If hit rate drops below 60%, re-evaluate whether caching is the right pattern for that workload.

## Relevant Links

- [ElastiCache Caching Strategies](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Strategies.html)
- [Aurora Read Replicas](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.html)
- [Aurora Reader Endpoint](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html)
- [DAX (DynamoDB Only)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html)
- [Aurora Parallel Query](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-mysql-parallel-query.html)
