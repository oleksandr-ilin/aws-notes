# Q03: Serverless Decoupling — API Gateway, Lambda, SQS, and DynamoDB

## Question

A company runs an order management system as a monolithic EC2 application. The application handles three types of requests with very different traffic patterns:

1. **Product catalog reads**: 10,000 TPS during peak, highly cacheable (catalog changes weekly), latency requirement < 50 ms. Currently hits a PostgreSQL database for every request.
2. **Order placement writes**: 500 TPS average, spikes to 5,000 TPS during flash sales (3x per month). Must be durable — no orders can be lost. Currently, if the database is overwhelmed, orders fail with HTTP 500 and customers must retry. Average processing time per order: 8 seconds (payment validation, inventory check, shipment scheduling).
3. **Order status queries**: 3,000 TPS, low latency (< 100 ms), reads against order data.

The team wants to decompose into serverless microservices. Requirements:
- No servers to manage (no EC2, no ECS)
- Each workload scales independently
- Flash sale order spikes must NOT cause failures — orders must be accepted instantly and processed asynchronously
- Product catalog reads must be cached to avoid database load
- Cost-optimized: pay only for actual usage (no idle capacity)

Which serverless architecture meets all requirements?

## Options

- **A.** **Product catalog**: API Gateway REST API with caching enabled (TTL = 24 hours) → Lambda → DynamoDB. API Gateway cache absorbs repeated reads — only cache misses invoke Lambda. DynamoDB on-demand for catalog data. **Order placement**: API Gateway → Lambda → SQS standard queue → Lambda (backend workers). The front-end Lambda validates the request and immediately sends it to SQS, returning HTTP 202 Accepted within 100 ms. Backend Lambda processes orders from SQS at a controlled concurrency (reserved concurrency = 100). SQS absorbs spikes — 5,000 TPS writes are queued instantly, processed gradually by 100 concurrent Lambda workers. DLQ for failed orders. **Order status**: API Gateway → Lambda → DynamoDB (on-demand, point reads by `orderId`). DynamoDB single-digit ms latency meets < 100 ms requirement.
- **B.** API Gateway HTTP API → Lambda for all three workloads. Lambda concurrency handles scaling. DynamoDB for all data. No caching — Lambda is fast enough. Orders processed synchronously by Lambda (8-second execution per order).
- **C.** Application Load Balancer → Lambda targets for all three workloads. ALB-based caching for product catalog. SQS for order buffering. Aurora Serverless v2 instead of DynamoDB.
- **D.** API Gateway → Step Functions Express Workflow for each request. Step Functions orchestrates Lambda, DynamoDB, and SNS. Product catalog cached in ElastiCache Redis (managed via Step Functions).

## Answers

### A. API Gateway caching + SQS buffering + DynamoDB — ✅ Correct

This architecture applies the right serverless pattern for each workload:

**Product Catalog — API Gateway Caching + DynamoDB:**

- **API Gateway cache**:
  - REST API supports built-in response caching (0.5 GB to 237 GB cache sizes). When enabled, API Gateway caches Lambda responses keyed by the request URL (and optionally headers, query parameters).
  - **First request**: API Gateway invokes Lambda → Lambda reads from DynamoDB → returns product data → API Gateway caches the response.
  - **Subsequent requests** (within TTL): API Gateway returns the cached response **without invoking Lambda at all**. Zero Lambda invocations, zero DynamoDB reads.
  - At 10,000 TPS with 24-hour TTL and a catalog of 5,000 products: after warmup, 99%+ of requests are served from cache. Lambda handles only ~50 TPS (cache misses for invalidated/expired items).
  - **Cache invalidation**: When the catalog is updated (weekly), call the API Gateway `InvalidateCache` API or deploy a new stage. The cache clears and next requests repopulate it.
  - **Cost impact**: The 0.5 GB cache costs $0.02/hour ($14.40/month). Without caching: 10,000 TPS × Lambda + DynamoDB costs = thousands per month. Caching reduces Lambda costs by 99%.

- **DynamoDB on-demand for catalog**:
  - Product catalog data (item descriptions, prices, images URLs) fits well in DynamoDB's key-value model.
  - On-demand capacity: DynamoDB scales read capacity based on the cache-miss traffic (not the full 10,000 TPS). Cost-efficient because most reads hit the API Gateway cache.
  - Read latency: DynamoDB `GetItem` (single partition key) returns in < 10 ms. Lambda adds ~50 ms (cold start: 100-500 ms for first invocation, then warm). Total: < 50 ms for cache misses.

**Order Placement — SQS Buffering Pattern:**

- **Synchronous acceptance, asynchronous processing**:
  1. Client sends `POST /orders` → API Gateway invokes the front-end Lambda
  2. Front-end Lambda: validates the request (check required fields, authenticate the user) — takes 10-50 ms
  3. Front-end Lambda sends the order to SQS: `sqs.sendMessage({QueueUrl: orderQueue, MessageBody: JSON.stringify(order)})` — takes 10-20 ms
  4. Front-end Lambda returns `HTTP 202 Accepted` with `{"orderId": "uuid", "status": "processing"}` — total response time: < 100 ms
  5. Client receives immediate confirmation — the order is durably stored in SQS

- **Backend processing**:
  - A separate Lambda function is triggered by SQS (event source mapping). Lambda polls SQS and processes orders in batches.
  - **Reserved concurrency = 100**: Limits the number of concurrent Lambda invocations processing orders. This controls: database connection count, downstream API call rate, and prevents overwhelming payment gateways.
  - Each Lambda invocation processes 1 order in 8 seconds. With 100 concurrent workers: 100 orders processed per 8 seconds = 12.5 orders/second sustained.
  - **Flash sale spike handling**: During 5,000 TPS spikes, SQS absorbs ALL 5,000 messages per second instantly (SQS has near-unlimited throughput). The 100 Lambda workers process the backlog gradually — a 5-minute spike of 5,000 TPS = 1.5 million orders queued, processed in ~33 hours by 100 workers.
  - To process faster: increase reserved concurrency to 500 → 62.5 orders/second → backlog cleared in ~6.7 hours. Balance processing speed vs. downstream system capacity.

- **SQS guarantees**:
  - **Durability**: SQS stores messages in multiple AZs — no message loss even during AZ failure.
  - **At-least-once delivery**: SQS may deliver a message more than once. The processing Lambda must be idempotent (check if orderId was already processed in DynamoDB).
  - **Visibility timeout**: Set to 60 seconds (> 8-second processing time + margin). If Lambda fails mid-processing, the message becomes visible again and is retried.
  - **DLQ**: After 3 failed attempts, the message moves to a dead-letter queue. Alarm on DLQ depth → operational team investigates failed orders.

- **Why not process synchronously?**:
  - Synchronous: API Gateway → Lambda (8 seconds per order) → returns result. At 5,000 TPS: 5,000 concurrent Lambda invocations × 8 seconds each = 40,000 concurrent executions. Default Lambda concurrent execution limit is 1,000 — requests throttled. Even with increased limit (10,000+), paying for 40,000 concurrent executions is extremely expensive.
  - Asynchronous with SQS: Front-end Lambda runs for 50 ms (not 8 seconds) → 5,000 TPS needs only 250 concurrent Lambda invocations (5,000 × 0.05s). Massive reduction in concurrent execution requirements.

**Order Status — DynamoDB Direct:**

- Simple `GetItem` by `orderId` — DynamoDB delivers single-digit ms latency. Lambda overhead adds 10-50 ms. Total: well under 100 ms.
- On-demand capacity handles 3,000 TPS reads without configuration.
- Eventually consistent reads ($0.25/million RCU) are acceptable for status queries — the order status was written seconds ago.

### B. Single Lambda for everything, no caching, synchronous orders — ❌ Incorrect

- **No caching**: 10,000 TPS product catalog reads → 10,000 Lambda invocations per second + 10,000 DynamoDB reads. Lambda at $0.20/million invocations + DynamoDB at $0.25/million reads = high cost. API Gateway caching reduces this by 99%.
- **HTTP API vs REST API**: API Gateway HTTP APIs are cheaper ($1.00 vs $3.50 per million requests) but do NOT support response caching. For the product catalog workload that benefits enormously from caching, REST API's built-in cache justifies the cost difference.
- **Synchronous 8-second order processing**: API Gateway's maximum integration timeout is 29 seconds (REST API) or 30 seconds (HTTP API). 8 seconds per order works — but at 5,000 TPS during flash sales: 5,000 × 8s = 40,000 concurrent Lambda invocations. This exceeds most account limits and costs significantly more than the SQS buffering pattern. Also, any failure during the 8 seconds returns HTTP 500 — the customer's order is lost unless they retry.

### C. ALB + Lambda + Aurora Serverless — ❌ Incorrect

- **ALB with Lambda targets**: ALB does support Lambda as a target, but there are key differences from API Gateway:
  - ALB does NOT support response caching — no equivalent to API Gateway's built-in cache. You'd need a separate caching layer (CloudFront or ElastiCache) — more complexity and cost.
  - ALB pricing: $0.008/LCU-hour + $0.0225/hour fixed. For Lambda targets, this adds baseline cost that API Gateway doesn't have (API Gateway charges per request only — zero idle cost).
  - ALB is the right choice when fronting EC2/ECS/Fargate. For Lambda-only backends, API Gateway is purpose-built.

- **Aurora Serverless v2**: Aurora Serverless scales compute automatically, but:
  - It's a relational database — requires connection management (connection pools, RDS Proxy for Lambda).
  - Minimum capacity: 0.5 ACU ($0.12/hour = $87.60/month even idle). DynamoDB on-demand charges nothing at zero traffic.
  - Startup latency: Aurora Serverless v2 scales in ~15 seconds for capacity increases. DynamoDB on-demand handles traffic instantly.
  - For key-value lookups (order by ID, product by ID), DynamoDB's single-digit ms latency with zero connection overhead is simpler and cheaper. Aurora Serverless is appropriate for complex relational queries, not key-value patterns.

### D. Step Functions for every request — ❌ Incorrect

- **Step Functions Express Workflow per request**: Express Workflows support up to 100,000 executions per second — sufficient throughput. But:
  - Express Workflow pricing: $0.000025 per state transition. A 3-step workflow (validate → process → respond) = 3 transitions × 10,000 TPS = 30,000 transitions/second = $64.80/day. Lambda direct invocation for read-heavy workloads is much cheaper.
  - Step Functions Express Workflows have a 5-minute maximum duration — insufficient for the 8-second order processing if combined with retries.
  - Step Functions are designed for multi-step **orchestrations** (order processing with branching logic, error handling, parallel steps). Using them for simple request-response patterns (read a DynamoDB item and return it) is over-engineering.
  - Step Functions don't have response caching — same problem as ALB for the product catalog workload.

- **ElastiCache Redis managed via Step Functions**: ElastiCache is NOT serverless in the traditional sense — it requires node selection, VPC configuration, and has hourly pricing. ElastiCache Serverless exists but still runs in a VPC (Lambda needs VPC configuration, adding cold start latency). API Gateway caching is simpler for HTTP response caching.

## Recommendations

- **API Gateway REST API caching** is the simplest way to add response caching to a serverless API. No additional infrastructure — enable it in the stage settings.
- **SQS for spike absorption** is a foundational pattern: accept immediately, process later. Use it whenever processing time > 1 second or input rate can exceed processing capacity.
- **Lambda reserved concurrency** on SQS consumers protects downstream systems. Without it, Lambda auto-scales to process the queue as fast as possible — potentially overwhelming databases or APIs.
- **Idempotency**: When using SQS (at-least-once delivery), ALWAYS implement idempotent processing. Use DynamoDB conditional writes (`PutItem` with `attribute_not_exists(orderId)`) to prevent duplicate order processing.
- **HTTP API vs REST API decision**: Use HTTP API for simple, cost-sensitive APIs. Use REST API when you need: caching, request validation schemas, request/response transformation, usage plans with API keys, or WAF integration.
- **Serverless decision criteria**: No servers to manage + pay-per-use + automatic scaling = API Gateway + Lambda + DynamoDB + SQS. Add ECS Fargate when you need: long-running processes (> 15 min), container-based workloads, or persistent connections.

## Relevant Links

- [API Gateway Caching](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-caching.html)
- [Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [Lambda Reserved Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)
- [SQS Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [DynamoDB On-Demand](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html#HowItWorks.OnDemand)
- [Serverless Application Patterns](https://docs.aws.amazon.com/lambda/latest/dg/applications-usecases.html)
