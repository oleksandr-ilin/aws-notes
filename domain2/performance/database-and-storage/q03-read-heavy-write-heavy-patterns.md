# Q03: Read-Heavy vs Write-Heavy Architecture Patterns

## Question

A company is designing two new microservices with opposing access patterns:

**Service A — Product Reviews (read-heavy)**
- 50,000 reads/second, 500 writes/second (100:1 read:write ratio)
- Reviews are text + metadata (2 KB average)
- Users browse reviews sorted by "most helpful" and "most recent"
- Reviews are immutable once posted (no edits) but can be soft-deleted
- 95th percentile read latency target: < 10 ms

**Service B — Real-Time Bidding Engine (write-heavy)**
- 200,000 writes/second (bid placements), 5,000 reads/second (bid lookups)
- Each bid is 500 bytes (bidder ID, item ID, amount, timestamp)
- Bids for a single item must be processed in order (highest bid wins)
- Bid data is only needed for 24 hours, then archived
- Write latency target: < 5 ms per bid

Both services run on ECS Fargate. Which data layer architecture is optimal for each?

## Options

- **A.** Service A: DynamoDB with DAX (DynamoDB Accelerator) for read caching. GSI on `product_id` with sort key `helpful_votes` for "most helpful" queries, another GSI with sort key `created_at` for "most recent." DAX handles the read amplification — most reads served from cache. Service B: DynamoDB with partition key `item_id` and sort key `timestamp`. Use DynamoDB Streams → Lambda → S3 for 24-hour archival. DynamoDB on-demand capacity for variable bid volumes. TTL attribute set to 24 hours for automatic deletion.
- **B.** Service A: Aurora PostgreSQL with 5 read replicas. Application routes reads to reader endpoint, writes to writer endpoint. Materialized views for "most helpful" and "most recent" sorting. Service B: Amazon Kinesis Data Streams for bid ingestion → Lambda → DynamoDB for bid state. Kinesis handles the write throughput; DynamoDB stores the current highest bid.
- **C.** Service A: ElastiCache Redis with write-through caching from Aurora PostgreSQL. Redis stores the latest 1,000 reviews per product as sorted sets. Aurora as the system of record. Service B: DynamoDB with write-heavy provisioned capacity. Use SQS FIFO queue upstream of DynamoDB for ordered bid processing per item.
- **D.** Service A: MongoDB on DocumentDB for flexible Schema. Service B: Apache Kafka on MSK for bid stream processing.

## Answers

### A. DynamoDB + DAX (reads) + DynamoDB with TTL (writes) — ✅ Correct

DynamoDB with DAX is the optimal pattern for both read-heavy and write-heavy workloads at this scale:

**Service A — Read-heavy with DAX:**

- **DynamoDB as base table**: Reviews are simple key-value/document data (text + metadata). DynamoDB provides consistent single-digit millisecond latency at any scale. Partition key: `product_id`, sort key: `review_id`.

- **DAX (DynamoDB Accelerator)**:
  - DAX is an in-memory write-through cache integrated with DynamoDB. Read requests first check DAX — on cache hit (typical for popular products), latency is **microseconds** (not milliseconds). On cache miss, DAX reads from DynamoDB and caches the result.
  - **50,000 reads/second**: DAX cluster with 3 nodes (r5.large) handles 500,000+ reads/second. The 100:1 ratio means DAX absorbs 99%+ of read traffic after warm-up.
  - **< 10 ms P95 latency**: DAX provides < 1 ms for cached reads. Even cache misses (DynamoDB reads) are 5-10 ms. P95 is comfortably under 10 ms.
  - **Write-through**: When a new review is posted, DAX writes to DynamoDB AND updates its cache — subsequent reads see the new review immediately. No cache invalidation logic needed.

- **GSIs for sorting**:
  - GSI 1: Partition key `product_id`, sort key `helpful_votes` (descending) → "most helpful" reviews
  - GSI 2: Partition key `product_id`, sort key `created_at` (descending) → "most recent" reviews
  - GSI queries are also cached by DAX. `Query` result sets are cached for the specified TTL.

**Service B — Write-heavy with DynamoDB:**

- **DynamoDB for 200,000 writes/second**:
  - DynamoDB on-demand mode auto-scales to handle any write volume. With partition key `item_id` and sort key `timestamp`, writes are distributed across partitions by item (hot items may require additional partitioning strategy via write sharding).
  - **< 5 ms write latency**: DynamoDB single-item puts are typically 5-10 ms. For guaranteed < 5 ms, use DynamoDB on-demand or provisioned with sufficient WCU headroom (no throttling).

- **In-order bid processing per item**:
  - DynamoDB sort key `timestamp` within partition `item_id` ensures bids are stored in chronological order. A conditional write (`ConditionExpression: attribute_not_exists(item_id) OR amount > :current_highest`) ensures only higher bids are accepted — atomic operation, no race conditions.

- **24-hour TTL + DynamoDB Streams archival**:
  - DynamoDB TTL attribute set to `created_at + 24 hours` — DynamoDB automatically deletes expired items at no cost (no WCU consumed for TTL deletions).
  - DynamoDB Streams → Lambda → S3 (Parquet format) for long-term archival before TTL deletion. Lambda processes stream records and writes to S3 in batches.

### B. Aurora + read replicas (A) + Kinesis → DynamoDB (B) — ❌ Incorrect

- **Aurora with 5 read replicas for Service A**: Aurora can achieve 50,000 reads/second across replicas, but PostgreSQL query latency for sorted reviews (ORDER BY helpful_votes DESC LIMIT 20) depends on table size, index efficiency, and connection pool management. At scale, maintaining indexes on frequently-changing sort columns (helpful_votes changes with every vote) causes index bloat and write amplification.
  - Materialized views for sorted data: Must be refreshed periodically (every N minutes). Between refreshes, new reviews are invisible. This adds staleness that DAX's write-through model avoids.
  - Aurora read replicas have 10-20 ms replication lag — acceptable but higher than DAX's microsecond reads.
  - Aurora is a valid choice but operationally heavier and higher latency than DynamoDB + DAX for this access pattern.

- **Kinesis Data Streams for bid ingestion**: Kinesis provides 1 MB/sec or 1,000 records/sec per shard write throughput. 200,000 writes/second requires 200+ shards. Kinesis → Lambda → DynamoDB adds processing latency (Lambda invocation + DynamoDB write) that may exceed the 5 ms write latency target.
  - Kinesis is designed for streaming analytics (time-windowed aggregations), not for low-latency transactional writes. Direct DynamoDB writes are simpler and faster.

### C. Redis write-through + SQS FIFO — ❌ Incorrect

- **ElastiCache Redis for Service A**: Redis sorted sets are excellent for "top N" queries — `ZREVRANGE product:123:reviews 0 19` returns the top 20 reviews by score in microseconds. However:
  - **Write-through from Aurora**: Application writes to Aurora AND Redis — dual-write consistency is hard. If Aurora write succeeds but Redis write fails (or vice versa), data is inconsistent. DAX handles this natively (it IS the cache + write-through path to DynamoDB).
  - **1,000 reviews per product**: Redis memory is expensive ($6/GB/month for r6g). If there are millions of products, storing 1,000 reviews × millions of products exceeds practical Redis memory limits. DynamoDB + DAX scales infinitely with on-demand pricing.
  - Redis + Aurora is a valid architecture for moderate scale, but DynamoDB + DAX is simpler and more scalable for this specific access pattern.

- **SQS FIFO for ordered bid processing (Service B)**:
  - SQS FIFO supports 300 messages/second per message group ID (3,000 with batching). For 200,000 bids/second across thousands of items, you'd need thousands of message group IDs (one per item) — and SQS FIFO's per-queue limit is 70,000 messages/second with high-throughput mode.
  - Adding SQS FIFO between the bidding API and DynamoDB introduces latency: API → SQS → Lambda consumer → DynamoDB = 50-200 ms round trip. This far exceeds the 5 ms target. DynamoDB conditional writes provide atomicity without queueing.

### D. DocumentDB + MSK — ❌ Incorrect

- **DocumentDB for reviews**: DocumentDB is MongoDB-compatible and handles flexible schemas, but it doesn't have DAX-equivalent read caching. At 50,000 reads/second, DocumentDB would require multiple instances (r5.4xlarge+). DocumentDB's read latency (5-20 ms) is higher than DynamoDB + DAX (microseconds). No automatic read caching layer equivalent to DAX.
- **MSK for bids**: Same issues as Kinesis (option B) — Kafka adds processing latency between bid submission and state persistence. Kafka's strength is stream processing (aggregations, joins, windowing), not low-latency transactional writes. MSK also requires cluster management (broker sizing, partition rebalancing) — operational overhead that DynamoDB serverless avoids.

## Recommendations

- **Read-heavy patterns**: Use DynamoDB + DAX for microsecond reads at any scale. DAX absorbs read amplification — your DynamoDB bill reflects only the write throughput.
- **Write-heavy patterns**: DynamoDB on-demand mode auto-scales to any write volume. Use conditional writes for atomic transactions. Use TTL for automatic cost-free cleanup.
- **DAX vs. ElastiCache Redis**: Use DAX when your primary database is DynamoDB — it's a transparent cache (same API, no code changes). Use Redis when you need advanced data structures (sorted sets, pub/sub, Lua scripting) or when caching for a non-DynamoDB database.
- **Conditional writes for concurrency**: `ConditionExpression` provides optimistic locking without external coordination — faster and simpler than SQS/FIFO queuing for ordered processing.
- **DynamoDB Streams** for event-driven archival, replication, and materialization — process changes as they happen rather than polling.

## Relevant Links

- [DynamoDB DAX](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html)
- [DynamoDB GSI](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html)
- [DynamoDB TTL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)
- [DynamoDB Conditional Writes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html)
- [DynamoDB On-Demand](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html#HowItWorks.OnDemand)
