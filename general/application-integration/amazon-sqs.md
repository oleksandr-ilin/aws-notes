# Amazon SQS

## Service Overview
Amazon SQS is a fully managed message queuing service offering standard and FIFO queues for decoupling microservices and distributed systems.

## Tasks this service can solve
- Durable buffering between services
- Work queue processing and asynchronous job handling
- Rate smoothing and retry buffering

## Alternatives
| Name | Type | Short description | Pro vs SQS | Cons vs SQS | Price comparison |
|---|---|---:|---|---|---|
| Kafka / MSK | OSS/AWS | Durable streaming platform | Ordering and replay features | More ops overhead | MSK cost higher for heavy throughput; SQS cheaper and simpler for basic queues |
| Amazon SNS | AWS | Pub/sub push service | Push notifications and fan-out | Not a durable queue | Different pricing model, often complementary |
| RabbitMQ / Amazon MQ | OSS/AWS | Broker-based queueing | Broker semantics & protocols | Ops or instance costs | Amazon MQ usually more expensive for broker features |

## Limitations
- Visibility timeout and delay parameters must be tuned; FIFO queues have throughput limits.
- Message size limits (256 KB for standard message payloads unless using S3 offload patterns).

## Price info
- Per-request pricing plus data transfer; FIFO queues may have different throughput considerations affecting cost.

## Network & Multi-region considerations
- SQS is regional; for cross-region durability, replicate messages or reroute via cross-region SNS/EventBridge.
- Can be used with VPC endpoints for private connectivity.

## Popular use cases
- Background job processing
- Decoupling services and rate-buffering
- Retry/work queues for resilient architectures

## When Not to Use This Service
- Avoid for ordered streaming semantics with replay and partitioning at scale—use Kafka/MSK or Kinesis for streaming workloads. For pub/sub-only needs without durability, SNS may be simpler.

## DR strategy
- Use cross-region message replication patterns or route messages through durable storage like S3 for archival. Combine SQS with DLQs and snapshot important messages to S3 for recovery; use multi-region producers/consumers where needed.
