# Amazon EventBridge

## Service Overview
Amazon EventBridge is a serverless event bus that routes events between AWS services, SaaS partners, and custom applications with filtering and schema support.

## Tasks this service can solve
- Central event routing and fan-out
- SaaS integration event ingestion
- Building loosely coupled event-driven architectures

## Alternatives
| Name | Type | Short description | Pro vs EventBridge | Cons vs EventBridge | Price comparison |
|---|---|---:|---|---|---|
| Amazon SNS + SQS | AWS | Pub/sub + durable queueing | Simpler pub/sub patterns | Less routing and schema features | Often cheaper for simple fan-out; EventBridge adds filtering and bus features |
| Kafka / MSK | OSS/AWS | Stream platform with topic semantics | Strong ordering & throughput | More ops | Higher infra cost; EventBridge is serverless per-event billing |
| Confluent | Commercial | Managed Kafka | Enterprise tooling | Costly licensing | Higher than native EventBridge for equivalent events |

## Limitations
- Per-event throughput quotas and API limits; schema registry is regional.
- Event size limits and retention constraints.

## Price info
- Charged per event published/ingested and for custom event buses and replay features.

## Network & Multi-region considerations
- EventBridge is regional. For cross-region propagation, use event bus replication or route events through cross-region EventBridge rules or use SNS for cross-region topics.
- Integrates with many AWS targets (Lambda, Step Functions, Kinesis, SQS, HTTP endpoints).

## Popular use cases
- SaaS event ingestion and routing
- Cross-account event buses for multi-account architectures
- Orchestrating serverless workflows via rules and targets

## When Not to Use This Service
- Not optimal for extremely high-throughput ordered streaming where Kafka/MSK is preferable; for simple pub/sub needs, SNS/SQS may be cheaper.

## DR strategy
- Use cross-region event replication patterns or route critical events through durable stores (S3, Kinesis) that are replicated. Maintain event schemas and use replay features where available; deploy rules in multiple regions if needed.
