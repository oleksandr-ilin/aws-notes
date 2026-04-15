# Q03: Amazon MQ vs SQS — Buffering Pattern, Queue Types, and Messaging Trade-offs

## Question

A company is migrating two messaging workloads to AWS:

**Workload 1 — Legacy order processing (currently on-premises RabbitMQ)**
- 15 microservices exchange messages using RabbitMQ with AMQP 0.9.1 protocol
- Message patterns: fan-out exchanges, topic-based routing with wildcards (e.g., `orders.*.completed`), message priorities (urgent orders processed first), and per-message TTL
- Messages are up to 50 KB each. Throughput: 2,000 messages/second sustained
- Requirement: Migrate to AWS with **minimal application code changes** — the team cannot rewrite 15 microservices to use a new messaging API in this release

**Workload 2 — New event ingestion pipeline (greenfield)**
- IoT sensors send telemetry events at 50,000 messages/second during peak
- Each message: 4 KB JSON payload
- Messages must be processed **exactly once** with **strict ordering per device** (events from device-123 must be processed in order, but events from different devices can be processed in parallel)
- Processing workers: Lambda functions consuming from the queue
- Requirement: **Fully managed, serverless scaling** — no broker instances to manage, no capacity planning

Which combination meets both workloads?

## Options

- **A.** Workload 1: **Amazon MQ for RabbitMQ** — managed RabbitMQ broker. Applications connect using the same AMQP 0.9.1 client libraries, exchange types (direct, topic, fanout, headers), routing keys, message priorities, and per-message TTL. Zero application code changes — only the broker endpoint URL changes. Deploy a cluster deployment (3 nodes across AZs) for HA. Workload 2: **SQS FIFO queue** with `MessageGroupId` set to the device ID. Messages with the same `MessageGroupId` are delivered in strict order. Enable content-based deduplication or provide `MessageDeduplicationId` for exactly-once processing. Lambda event source mapping polls the queue with a batch size of 10, processing messages per group. FIFO throughput: 3,000 messages/sec with batching (300 API calls/sec × 10 messages/batch) per queue — use multiple FIFO queues or high throughput mode (30,000 messages/sec) for the 50,000 peak. SQS scales automatically — no brokers to manage.
- **B.** Workload 1: Amazon SQS Standard queue with SNS for fan-out. Rewrite microservices to use the SQS API. Workload 2: Amazon MQ for ActiveMQ with message selectors for per-device ordering.
- **C.** Workload 1: Amazon MQ for ActiveMQ (supports AMQP). Workload 2: Amazon Kinesis Data Streams with per-device partition keys.
- **D.** Workload 1: Amazon MSK (Kafka) — supports topic-based routing. Workload 2: SQS Standard queue with Lambda. Use DynamoDB for deduplication.

## Answers

### A. Amazon MQ for RabbitMQ + SQS FIFO — ✅ Correct

**Workload 1 — Amazon MQ for RabbitMQ:**

- **What is Amazon MQ?**
  - Amazon MQ is a managed message broker service that supports **Apache ActiveMQ** and **RabbitMQ** engines. It runs the actual open-source broker software on managed infrastructure — you get the full protocol and feature compatibility without managing servers.
  - **Key difference from SQS/SNS**: SQS and SNS are AWS-proprietary services with AWS-specific APIs. Amazon MQ runs standard open-source brokers with industry-standard protocols (AMQP, MQTT, STOMP, OpenWire). Applications using these protocols can migrate to Amazon MQ with zero code changes.

- **Why RabbitMQ for this workload**:
  - **AMQP 0.9.1 protocol**: The 15 microservices use RabbitMQ client libraries (e.g., `amqplib` for Node.js, `pika` for Python). These libraries connect to the broker using AMQP 0.9.1. Amazon MQ for RabbitMQ speaks the same protocol — applications change only the broker hostname.
  - **Exchange types**: RabbitMQ supports direct, topic, fanout, and headers exchanges. The `orders.*.completed` routing pattern uses topic exchanges with wildcard binding keys. SQS/SNS can approximate fan-out (SNS → multiple SQS queues) but doesn't support wildcard topic routing natively.
  - **Message priorities**: RabbitMQ supports per-message priority (0-255). Consumers receive higher-priority messages first from the same queue. SQS has no native message priority — all messages are treated equally.
  - **Per-message TTL**: RabbitMQ supports per-message expiration (`expiration` property). Expired messages are dead-lettered or discarded. SQS has per-queue message retention (1 minute to 14 days) but not per-message TTL.

- **Deployment**:
  - **Cluster deployment** (recommended for production): 3 RabbitMQ nodes across 3 AZs. Queues are replicated across nodes (quorum queues or classic mirrored queues). If one node fails, the other two continue serving.
  - **Instance types**: mq.m5.large for 2,000 msg/sec sustained. RabbitMQ achieves 10,000-50,000 msg/sec depending on message size and instance type.
  - **Storage**: Amazon MQ uses EBS for message persistence. Messages are durable (survive broker restart).

- **When to use Amazon MQ vs. SQS/SNS**:

  | Criterion | Amazon MQ | SQS/SNS |
  |---|---|---|
  | Protocol | AMQP, MQTT, STOMP, OpenWire | AWS SDK (HTTP API) |
  | Migration | Lift-and-shift from on-prem brokers | Greenfield or willing to refactor |
  | Scaling | Manual (choose instance type) | Automatic (serverless) |
  | Throughput | Depends on broker instance | Nearly unlimited (SQS Standard) |
  | Message priority | ✅ Native | ❌ Not supported |
  | Wildcard topic routing | ✅ Native | ❌ SNS filter policies (limited) |
  | Per-message TTL | ✅ Native | ❌ Per-queue only |
  | Cost at scale | Higher (broker instances 24/7) | Lower (pay per request) |
  | Operational burden | Medium (broker upgrades, sizing) | Minimal (fully managed) |

**Workload 2 — SQS FIFO Queue:**

- **SQS FIFO (First-In-First-Out)**:
  - **Strict ordering per `MessageGroupId`**: All messages with the same group ID are delivered in the exact order they were sent. Setting `MessageGroupId = deviceId` ensures device-123's events are processed sequentially.
  - **Different group IDs process in parallel**: Messages from device-123 and device-456 have different group IDs — they're processed concurrently by different Lambda invocations. This provides per-device ordering without global serialization.

- **Exactly-once processing**:
  - **Content-based deduplication**: SQS calculates a SHA-256 hash of the message body. If two identical messages are sent within the 5-minute deduplication window, the second is discarded. Enable with `ContentBasedDeduplication = true`.
  - **`MessageDeduplicationId`**: Alternatively, provide an explicit deduplication ID (e.g., `deviceId-timestamp-sequenceNumber`). SQS deduplicates based on this ID within the 5-minute window.
  - This is **exactly-once delivery** — the consumer receives each unique message exactly once. SQS Standard queues provide at-least-once delivery (duplicates possible).

- **FIFO throughput**:
  - **Without batching**: 300 messages/second per queue (300 send, 300 receive API calls/sec)
  - **With batching** (10 messages per API call): 3,000 messages/second per queue
  - **High throughput mode**: Up to 30,000 messages/second per queue (with message group partitioning across multiple internal partitions)
  - For 50,000 msg/sec peak: Use high throughput mode + multiple FIFO queues sharded by device ID range. Alternatively, a single FIFO queue in high throughput mode handles 30,000/sec — add a second queue for the remaining 20,000/sec.

- **Lambda event source mapping with FIFO**:
  - Lambda polls the FIFO queue and processes messages in batches. With FIFO, Lambda processes messages per message group sequentially — it won't start processing the next batch for group-123 until the current batch succeeds.
  - Batch size: 1-10 messages. Batch window: 0-300 seconds. Lambda scales the number of concurrent invocations up to the number of active message groups.
  - **Scaling**: If there are 10,000 active devices, Lambda runs up to 10,000 concurrent invocations (one per message group with active messages). This is why `MessageGroupId` is critical — it defines the unit of parallelism.

- **SQS FIFO vs. Standard comparison**:

  | Feature | SQS Standard | SQS FIFO |
  |---|---|---|
  | Throughput | Nearly unlimited | 3,000-30,000 msg/sec |
  | Ordering | Best-effort | Strict per message group |
  | Delivery | At-least-once (duplicates possible) | Exactly-once |
  | Deduplication | ❌ | ✅ (content-based or explicit) |
  | Message groups | ❌ | ✅ |
  | Lambda scaling | Per-queue concurrency | Per-message-group concurrency |
  | Queue name | Any | Must end in `.fifo` |

### B. SQS Standard (rewrite) + Amazon MQ for ActiveMQ — ❌ Incorrect

- **Rewriting 15 microservices to SQS API**: This violates the "minimal application code changes" requirement. Migrating from RabbitMQ (AMQP client libraries) to SQS (AWS SDK) requires: replacing connection logic, changing message publishing code, replacing exchange/queue declarations with SQS queue/SNS topic creation, and reimplementing routing logic. This is a significant refactoring effort — exactly what Amazon MQ is designed to avoid.

- **Amazon MQ for ActiveMQ for the IoT pipeline**:
  - ActiveMQ is a message broker, not a serverless queuing service. It requires instance sizing, capacity planning, and broker management — violating the "no broker instances to manage" requirement.
  - ActiveMQ message selectors (SQL-like filtering on message headers) can filter messages by device, but ordering requires exclusive consumers on per-device queues — creating 10,000+ queues for 10,000 devices is impractical.
  - ActiveMQ throughput: 1,000-10,000 msg/sec depending on instance type and persistence. Handling 50,000 msg/sec requires a large broker cluster — expensive and operationally complex compared to SQS FIFO's serverless scaling.

### C. Amazon MQ for ActiveMQ + Kinesis Data Streams — ❌ Incorrect

- **Amazon MQ for ActiveMQ (not RabbitMQ)**: ActiveMQ supports AMQP 1.0 (not AMQP 0.9.1). RabbitMQ uses AMQP 0.9.1 — the protocol versions are incompatible. AMQP 0.9.1 clients cannot connect to an ActiveMQ broker via AMQP 1.0 without code changes. Amazon MQ for RabbitMQ is the correct choice for AMQP 0.9.1 compatibility.

- **Kinesis Data Streams for IoT ordering**: Kinesis provides strict ordering per shard (using partition keys). Setting partition key = device ID ensures per-device ordering. However:
  - Kinesis provides **at-least-once delivery**, not exactly-once. Duplicate processing is possible (e.g., on consumer retry after checkpoint failure). The requirement specifies exactly-once processing — SQS FIFO provides this natively.
  - Kinesis requires shard capacity planning: each shard handles 1,000 records/sec writes and 2 MB/sec reads. For 50,000 msg/sec, you need 50 shards — with manual resharding as traffic changes. SQS FIFO is serverless (no shard management).
  - Kinesis is ideal for real-time streaming analytics (multiple consumers, replay, analytics). For a simple queue-based processing pipeline, SQS FIFO is simpler and cheaper.

### D. MSK (Kafka) + SQS Standard + DynamoDB dedup — ❌ Incorrect

- **Amazon MSK for RabbitMQ migration**: Kafka uses a completely different messaging model (topics, partitions, consumer groups, offsets) from RabbitMQ (exchanges, queues, bindings, acknowledgements). Migrating from RabbitMQ to Kafka requires complete application rewrite — the opposite of "minimal code changes."
  - Kafka doesn't support message priority, per-message TTL, or exchange-type routing. Feature parity with RabbitMQ would require significant custom development.

- **SQS Standard + DynamoDB deduplication**: SQS Standard provides at-least-once delivery. Building exactly-once on top of it requires: reading the message, checking DynamoDB for the deduplication ID, conditionally processing, then writing the deduplication record — all within a transaction. This is complex, error-prone, and slower than SQS FIFO's built-in deduplication.
  - SQS Standard also doesn't guarantee ordering — messages may arrive out of order. Per-device ordering would require custom sequence numbers and reordering logic in the consumer. SQS FIFO provides both ordering and deduplication natively.

## Recommendations

- **Amazon MQ**: Use for lift-and-shift migrations from on-premises message brokers (RabbitMQ, ActiveMQ, IBM MQ). Choose RabbitMQ engine for AMQP 0.9.1 workloads, ActiveMQ for JMS/OpenWire workloads.
- **SQS**: Use for greenfield AWS-native messaging. Standard for maximum throughput with at-least-once delivery. FIFO for ordered, exactly-once processing.
- **SQS FIFO high throughput mode**: Enable it for new FIFO queues by default — it provides up to 30,000 msg/sec with no downside. Available in most Regions.
- **Message payload size**: SQS supports up to 256 KB per message. For larger payloads, use the Amazon SQS Extended Client Library — stores the payload in S3 and sends a pointer in the SQS message (up to 2 GB).
- **SQS batching**: Always use `SendMessageBatch` (up to 10 messages) and `ReceiveMessage` with `MaxNumberOfMessages = 10` to maximize throughput and reduce API costs.
- **Long polling**: Set `WaitTimeSeconds = 20` on `ReceiveMessage` to reduce empty responses and API costs. Short polling (`WaitTimeSeconds = 0`) returns immediately even if no messages are available — wasteful.

## Relevant Links

- [Amazon MQ for RabbitMQ](https://docs.aws.amazon.com/amazon-mq/latest/developer-guide/welcome.html)
- [SQS FIFO Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)
- [SQS FIFO High Throughput](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/high-throughput-fifo.html)
- [SQS Message Deduplication](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/using-messagededuplicationid-property.html)
- [Amazon MQ vs SQS](https://docs.aws.amazon.com/amazon-mq/latest/developer-guide/amazon-mq-migrating.html)
- [SQS Extended Client Library](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-s3-messages.html)
