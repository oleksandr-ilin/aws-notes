# Q01: Multi-Layer Caching Strategy for an E-Commerce Platform

## Question

A global e-commerce company serves 50 million daily active users. The platform has these characteristics:
- Product catalog (2 million SKUs) is read 500× more than it is written
- Product pages include personalized pricing per customer segment (5 segments)
- Search results must return within 200 ms globally
- Order processing writes average 15,000 orders per minute at peak
- Static assets (images, CSS, JS) total 8 TB

Current architecture: CloudFront → ALB → EC2 (catalog API) → Aurora MySQL (catalog DB). Every product page request hits the Aurora reader endpoint. P99 latency has grown to 1.8 seconds as traffic increased. The team has budget for caching but wants to minimize application code changes.

Which caching strategy best reduces database load and achieves the sub-200 ms latency target?

## Options

- **A.** Add an ElastiCache for Redis cluster between the EC2 fleet and Aurora. Implement cache-aside pattern in the catalog API: check Redis first, fetch from Aurora on miss, write to Redis with a 5-minute TTL. Use Redis hash structures to store per-segment pricing as hash fields within a single product key. Enable CloudFront caching of static assets with 24-hour TTL. Add API Gateway with 30-second response caching for the top 1,000 most-accessed product endpoints.
- **B.** Enable Aurora Serverless v2 to auto-scale reader ACUs during traffic spikes. Add five more Aurora read replicas (one per segment) and route each segment's queries to a dedicated replica. Enable CloudFront caching of all API responses with `Vary: X-Customer-Segment` header. Move static assets to S3 with CloudFront origin.
- **C.** Replace Aurora with DynamoDB with DAX (DynamoDB Accelerator) for the catalog. Store each product as a DynamoDB item with segment pricing as a map attribute. DAX provides automatic read-through caching with microsecond latency. Use CloudFront for static assets. Use DynamoDB Streams to invalidate DAX when products are updated.
- **D.** Deploy a Memcached cluster for session caching only. Scale Aurora vertically to a db.r6g.16xlarge instance. Add CloudFront with Lambda@Edge to personalize pricing at the edge by querying a DynamoDB global table. Cache rendered HTML pages in CloudFront with 60-second TTL.

## Answers

### A. ElastiCache Redis cache-aside + CloudFront static + API Gateway caching — ✅ Correct

This is the most effective and least disruptive approach:

- **ElastiCache for Redis cache-aside pattern**: The application checks Redis before querying Aurora. On a cache miss, it reads from Aurora and writes the result to Redis with a TTL. For a 500:1 read:write ratio, this offloads ~99.8% of reads from Aurora after warm-up. Redis operates at sub-millisecond latency, dramatically reducing P99.

- **Redis hash structures for per-segment pricing**: A single key `product:{id}` stores a hash with fields `segment:1:price`, `segment:2:price`, etc. This avoids duplicating the entire product record per segment and allows `HGET` for a specific segment's price — O(1) operation. One key per product × 2M products = manageable memory footprint.

- **5-minute TTL**: Balances freshness with cache hit ratio. Product data changes infrequently (read:write = 500:1), so stale data risk is low. On catalog update, the application can explicitly `DEL` the key for immediate consistency if needed.

- **CloudFront static asset caching**: 8 TB of images/CSS/JS served from edge locations with 24-hour TTL. This eliminates origin fetches for static content — the single largest latency contributor for page loads.

- **API Gateway response caching**: The top 1,000 products likely account for a power-law majority of traffic (Pareto distribution). 30-second API caching at the edge eliminates redundant API calls entirely for hot products. This requires minimal code changes — just move the API behind API Gateway.

- **Minimal code changes**: Cache-aside is a well-understood pattern requiring only a Redis client wrapper around the existing database call. No database migration needed.

### B. Aurora scaling + read replicas + CloudFront API caching — ❌ Incorrect

- **Five dedicated replicas per segment**: Over-provisioned and expensive. Aurora supports max 15 read replicas, and provisioning 5 dedicated ones means each replica handles only 20% of traffic — poor utilization. Routing logic adds application complexity.
- **Aurora Serverless v2 scaling**: Helps with capacity but doesn't reduce per-query latency — each request still hits the database. The fundamental problem is database round-trip latency, not database capacity.
- **CloudFront `Vary: X-Customer-Segment`**: This creates 5 cached copies per URL (one per segment) at every edge location. Cache hit ratio plummets because the cache is fragmented across 5 variants × millions of URLs × 400+ edge locations. Most variants would never be cached.
- This approach throws capacity at the problem instead of eliminating unnecessary database calls.

### C. DynamoDB + DAX migration — ❌ Incorrect

- **Complete database migration** from Aurora MySQL to DynamoDB is a massive undertaking — schema redesign, query rewrite, data migration, testing. This violates the "minimize application code changes" requirement.
- **DAX** provides excellent read-through caching but is tightly coupled to DynamoDB — the team would need to migrate first before getting any caching benefit.
- **DynamoDB Streams invalidating DAX**: DAX actually handles invalidation automatically when items are written through DAX. Streams-based invalidation adds unnecessary complexity.
- The correct answer achieves the same caching benefit (ElastiCache) without requiring a database migration.

### D. Memcached + vertical scaling + Lambda@Edge — ❌ Incorrect

- **Memcached for session caching only**: Doesn't address the core problem of product catalog read load on Aurora. Sessions aren't the bottleneck.
- **Vertical scaling (db.r6g.16xlarge)**: Expensive ($8+/hour), has a ceiling, and doesn't reduce per-query latency — just increases concurrent query capacity.
- **Lambda@Edge querying DynamoDB global table**: Lambda@Edge has a 5-second timeout (viewer-facing) and adds latency for DynamoDB queries from edge locations. Edge functions are best for lightweight transformations, not database-backed personalization.
- **Caching rendered HTML with 60-second TTL**: Price changes (promotions, flash sales) would be stale for up to 60 seconds — unacceptable for commerce. Also, 2M products × 5 segments = 10M cache entries of full HTML pages — massive cache storage.

## Recommendations

- **Cache-aside with ElastiCache Redis** is the default caching pattern for relational databases — it requires no database changes and minimal application changes.
- **Use Redis over Memcached** when you need: data structures (hashes, sorted sets, lists), persistence, replication, pub/sub for cache invalidation.
- **Use Memcached** when you need: simple key-value, multi-threaded performance, and you don't need persistence or advanced data structures.
- **DAX** is purpose-built for DynamoDB — if you're already on DynamoDB, use DAX instead of ElastiCache for read caching (it's transparent to the application).
- **Layer caching**: CDN (static + API) → API Gateway cache (hot endpoints) → Application cache (ElastiCache) → Database (Aurora). Each layer catches requests the previous layer missed.
- **Set TTL based on data volatility**: static assets (hours-days), product catalog (minutes), pricing (seconds-minutes), inventory (seconds or no-cache).

## Relevant Links

- [ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html)
- [Caching Strategies (cache-aside, write-through, etc.)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Strategies.html)
- [API Gateway Response Caching](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html)
- [CloudFront Cache Behavior](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/RequestAndResponseBehaviorCustomOrigin.html)
- [DAX (DynamoDB Accelerator)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html)
