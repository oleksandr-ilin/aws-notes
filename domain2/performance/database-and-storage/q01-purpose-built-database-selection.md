# Q02: Purpose-Built Database Selection for a Multi-Model Application

## Question

A fintech company is building a new platform with four distinct data workloads:

1. **Transaction ledger** — Immutable record of every financial transaction. Must be cryptographically verifiable and support historical queries for auditing. 200,000 transactions per day.
2. **User social graph** — Tracks relationships (referrals, shared accounts, trusted contacts). Queries like "find all users within 3 degrees of connection who also use product X." 5 million user nodes, 50 million edges.
3. **Real-time fraud scoring** — Ingests transaction events, computes risk scores using sliding time-window aggregations (last 5 min, 1 hour, 24 hours). Must respond within 10 ms for each transaction. Scores are based on velocity checks (e.g., "how many transactions from this card in the last 5 minutes?").
4. **Product catalog and configuration** — Standard CRUD operations, complex joins across product features, pricing tiers, and eligibility rules. Referenced by multiple microservices.

The team wants to use purpose-built AWS databases rather than forcing all workloads into a single relational database. Which combination of databases is most appropriate?

## Options

- **A.** (1) Amazon QLDB for the transaction ledger, (2) Amazon Neptune for the social graph, (3) Amazon ElastiCache for Redis with Sorted Sets for fraud scoring time-window aggregations, (4) Amazon Aurora PostgreSQL for the product catalog.
- **B.** (1) Amazon DynamoDB with DynamoDB Streams for the ledger, (2) Amazon Neptune for the social graph, (3) Amazon Timestream for fraud scoring, (4) Amazon Aurora MySQL for the product catalog.
- **C.** (1) Amazon Aurora PostgreSQL with immutable table design for the ledger, (2) Amazon Neptune for the social graph, (3) Amazon DynamoDB with TTL for fraud scoring, (4) Amazon DocumentDB for the product catalog.
- **D.** (1) Amazon QLDB for the transaction ledger, (2) Amazon DynamoDB with GSIs for the social graph, (3) Amazon Kinesis Data Analytics (Managed Flink) for fraud scoring, (4) Amazon RDS PostgreSQL Multi-AZ for the product catalog.

## Answers

### A. QLDB + Neptune + ElastiCache Redis + Aurora PostgreSQL — ✅ Correct

Each database is matched to the access pattern it was designed for:

- **Amazon QLDB for the transaction ledger**: QLDB is a purpose-built ledger database with an immutable, append-only journal. Every change is cryptographically hashed and chained — you can mathematically prove that no record has been altered since insertion. This is exactly what "cryptographically verifiable" requires. QLDB supports SQL-like queries (PartiQL) for historical auditing. 200K transactions/day is well within QLDB's capacity.

- **Amazon Neptune for the social graph**: Neptune is a graph database supporting both Gremlin (property graph) and SPARQL (RDF). "Find all users within 3 degrees of connection who also use product X" is a classic graph traversal query — expressed in Gremlin as `g.V(userId).repeat(both('connected').simplePath()).times(3).has('product','X')`. In a relational database, this would require recursive CTEs or self-joins that degrade exponentially with depth. Neptune handles multi-hop traversals in milliseconds.

- **ElastiCache for Redis with Sorted Sets for fraud scoring**: Redis Sorted Sets allow `ZRANGEBYSCORE` queries — "give me all transactions for card X with timestamp between now-5min and now." This is O(log(N) + M) where M is the result set. Combined with `ZCARD` for count, you can compute velocity checks in sub-millisecond latency — well under the 10 ms requirement. Redis TTL on the sorted set members automatically evicts old data (e.g., transactions older than 24 hours). This is a proven pattern for real-time sliding window aggregation.

- **Aurora PostgreSQL for the product catalog**: Complex joins across product features, pricing tiers, and eligibility rules are textbook relational queries. Aurora PostgreSQL supports advanced SQL (window functions, CTEs, JSONB for semi-structured attributes) and scales to 128 TB. Being referenced by multiple microservices is handled by Aurora's reader endpoints for read scaling.

### B. DynamoDB + Neptune + Timestream + Aurora MySQL — ❌ Incorrect

- **DynamoDB for the ledger**: DynamoDB is not immutable — items can be updated and deleted. DynamoDB Streams provide change tracking but not cryptographic verification. You could implement append-only patterns, but the application must enforce immutability — QLDB enforces it at the database layer.
- **Timestream for fraud scoring**: Timestream is designed for IoT telemetry and DevOps metrics with scheduled queries and time-series aggregation — not for sub-10ms point queries during transaction processing. Timestream's query latency is typically 100ms+, which exceeds the 10ms requirement. Timestream excels at analytical time-series queries (avg, p99, trends over time), not real-time velocity checks per individual transaction.
- Neptune and Aurora MySQL are reasonable for their workloads, but the ledger and fraud components are mismatched.

### C. Aurora immutable design + Neptune + DynamoDB TTL + DocumentDB — ❌ Incorrect

- **Aurora with immutable table design**: You can implement append-only tables in PostgreSQL (revoke UPDATE/DELETE, use triggers), but this is application-level enforcement. There's no cryptographic chain proving data integrity — a DBA with direct access could modify records. QLDB provides this guarantee at the database engine level.
- **DynamoDB with TTL for fraud scoring**: TTL only deletes expired items — it doesn't provide time-window aggregation queries. You'd need to implement windowing logic in the application. DynamoDB's eventual consistency for reads (or the cost of strongly consistent reads at scale) also complicates real-time scoring. No native sorted-by-time index within a single partition key that supports range scans as efficiently as Redis Sorted Sets.
- **DocumentDB for product catalog**: DocumentDB is a document database (MongoDB-compatible). "Complex joins across pricing tiers and eligibility rules" are relational queries — DocumentDB doesn't support joins. You'd need to denormalize everything or perform application-level joins, which is fragile and harder to maintain.

### D. QLDB + DynamoDB GSIs + Kinesis Flink + RDS PostgreSQL — ❌ Incorrect

- **DynamoDB with GSIs for the social graph**: Graph traversals in DynamoDB (adjacency list pattern) work for 1-hop queries but become extremely expensive for multi-hop (3 degrees of separation). Each hop requires a separate query, and the fan-out grows exponentially: 1 user → 50 connections → 2,500 connections → 125,000 connections. Neptune handles this natively with index-free adjacency.
- **Kinesis Data Analytics (Managed Flink) for fraud scoring**: Flink is a stream processing engine, not a database. It can compute sliding window aggregations but operates on streams, not point queries. For "score this specific transaction right now," you'd need to query Flink's state — which is not designed for low-latency random-access reads. Flink is better suited for continuous aggregate computation that writes results elsewhere (e.g., to Redis or DynamoDB).
- **RDS PostgreSQL Multi-AZ for catalog**: Functional but inferior to Aurora for this use case. Aurora provides auto-scaling storage, up to 15 read replicas (vs. 5 for RDS), faster failover (~30s vs. 60-120s for RDS Multi-AZ), and reader endpoints for load balancing read traffic from multiple microservices.
- QLDB is correct for the ledger.

## Recommendations

- **Purpose-built database selection rule**: Identify the primary access pattern first, then choose the database optimized for that pattern. Don't force-fit a general-purpose database.
- **QLDB** → immutable, auditable ledger with cryptographic verification (finance, supply chain, HR records)
- **Neptune** → highly connected data with multi-hop traversals (social networks, fraud rings, recommendation engines, knowledge graphs)
- **ElastiCache Redis** → sub-millisecond key-value lookups, counters, leaderboards, sliding window aggregation (real-time scoring, rate limiting, session stores)
- **Timestream** → high-volume time-series ingestion with scheduled analytical queries (IoT telemetry, DevOps monitoring) — not for real-time point queries
- **Aurora** → relational workloads with complex joins, transactions, and read scaling
- **DynamoDB** → single-digit-millisecond key-value/document access at any scale, when access patterns are well-defined

## Relevant Links

- [Amazon QLDB](https://docs.aws.amazon.com/qldb/latest/developerguide/what-is.html)
- [Amazon Neptune](https://docs.aws.amazon.com/neptune/latest/userguide/intro.html)
- [ElastiCache for Redis Sorted Sets](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Strategies.html)
- [Amazon Timestream](https://docs.aws.amazon.com/timestream/latest/developerguide/what-is-timestream.html)
- [Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html)
- [Purpose-Built Databases on AWS](https://aws.amazon.com/products/databases/)
