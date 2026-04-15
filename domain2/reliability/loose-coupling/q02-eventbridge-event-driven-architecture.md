# Q02: Event-Driven Architecture with EventBridge for Loose Coupling

## Question

A logistics company is decomposing a monolithic order management system into microservices. The current architecture has direct synchronous HTTP calls between services:

```
Order Service → (HTTP) → Inventory Service → (HTTP) → Shipping Service → (HTTP) → Notification Service
```

Problems with the current architecture:
- If Notification Service is down, the entire order flow fails (tight coupling)
- Adding a new service (e.g., Analytics) requires modifying Order Service to add another HTTP call
- During peak periods (holiday shipping), Inventory Service is overwhelmed by synchronous calls from Order Service
- No audit trail of events — when issues occur, the team can't determine which service processed which event

The team wants an event-driven architecture where:
- Services are loosely coupled — failure of one service doesn't block others
- New consumers can be added without modifying producers
- Events are retained for 90 days for replay and auditing
- Each service processes events at its own pace (backpressure handling)
- Exactly-once processing is guaranteed for inventory updates

Which architecture achieves all requirements?

## Options

- **A.** Amazon EventBridge as the central event bus. Order Service publishes `OrderCreated` events to EventBridge. EventBridge rules route events: `OrderCreated` → Inventory Service SQS queue, → Shipping Service SQS queue, → Notification Service SQS queue. Each service polls its own SQS queue independently. EventBridge Archive stores events for 90 days with replay capability. Inventory Service uses SQS FIFO queue with deduplication for exactly-once processing. EventBridge schema registry documents event schemas for new consumer onboarding.
- **B.** Amazon SNS as the central event bus. Order Service publishes to an SNS topic. Subscriptions fan out to Lambda functions for each service. Dead-letter queues for failed deliveries. CloudWatch Logs retains events for 90 days. SNS message deduplication for exactly-once processing.
- **C.** Direct SQS queues between services. Order Service sends messages to 3 separate SQS queues (one per consumer). New consumers require new queues and Order Service code changes. Use S3 to archive messages for 90 days.
- **D.** Apache Kafka on Amazon MSK (Managed Streaming for Kafka). Order Service produces to a Kafka topic. Each service is a consumer group reading from the topic. Kafka retains messages for 90 days. Kafka's consumer group offsets provide exactly-once semantics.

## Answers

### A. EventBridge + SQS queues + Archive + FIFO deduplication + Schema Registry — ✅ Correct

This architecture provides true event-driven loose coupling with all required features:

- **Amazon EventBridge as central event bus**:
  - EventBridge is a serverless event bus that routes events based on rules. The Order Service publishes an `OrderCreated` event with a structured JSON payload. EventBridge matches events against rules and routes them to targets.
  - **Loose coupling**: The Order Service publishes once and doesn't know (or care) who consumes the event. If Notification Service is down, Order Service isn't affected — the event is still delivered to other targets.
  - **Adding new consumers**: To add an Analytics Service, create a new EventBridge rule matching `OrderCreated` events → target: Analytics Service SQS queue. Zero changes to Order Service or any other existing service. This is the key advantage over direct HTTP calls.

- **SQS queues per consumer (backpressure handling)**:
  - Each consumer service has its own SQS queue. If Inventory Service is slow during peak, messages accumulate in its queue — but Shipping and Notification Services continue processing at normal speed from their own queues.
  - **Backpressure**: Each service polls its SQS queue at its own rate. SQS retains messages for up to 14 days (configurable), providing a buffer during traffic spikes. No service is overwhelmed by synchronous calls.
  - **Dead-letter queues**: Failed messages (processing errors) are moved to DLQs after a configured number of retries — preventing poison messages from blocking the queue.

- **EventBridge Archive (90-day retention)**:
  - EventBridge Archives store events for a configurable retention period (up to indefinite). Events can be replayed to the event bus for re-processing — useful for debugging, backfilling new consumers, or recovering from processing failures.
  - **Audit trail**: Each archived event includes the source, detail-type, timestamp, and full payload — providing a complete audit trail of "which event was published when."

- **SQS FIFO queue for Inventory Service (exactly-once)**:
  - SQS FIFO queues provide **exactly-once delivery** using message deduplication IDs. If the same `OrderCreated` event is published twice (e.g., retry), the FIFO queue deduplicates based on the dedup ID (order ID) within the 5-minute deduplication window.
  - This prevents double-decrementing inventory — if an order event is processed twice, inventory would show incorrect counts. FIFO deduplication prevents this.
  - **Note**: EventBridge → SQS FIFO requires the EventBridge rule to set the MessageGroupId (e.g., based on the order's warehouse region for ordered processing within that group).

- **EventBridge Schema Registry**:
  - Schema Registry automatically discovers and stores event schemas. New consumers can browse available event schemas, understand the payload structure, and generate code bindings — reducing onboarding time from days to hours.

### B. SNS fan-out + Lambda + CloudWatch Logs — ❌ Incorrect

- **SNS for fan-out**: SNS does fan out to multiple subscribers, but it's push-based — SNS pushes messages to subscribers (Lambda, HTTP, SQS, email). If a subscriber is unavailable, SNS retries for a limited period (configurable, up to 23 days for SQS, but shorter for Lambda/HTTP). For Lambda targets, SNS invokes the function synchronously — if Lambda throttles, messages may be lost without a DLQ.
- **No native event archive/replay**: SNS doesn't retain messages after delivery. CloudWatch Logs can store log entries but doesn't provide event replay (you can't re-inject logged messages back into the pipeline). EventBridge Archive provides native replay.
- **SNS message deduplication**: Standard SNS topics do NOT have message deduplication. SNS FIFO topics support deduplication but can only deliver to SQS FIFO queues (not Lambda, HTTP, or email) — limiting subscriber flexibility.
- **Lambda for each service**: Invoking Lambda directly from SNS means each service is limited to Lambda (15-minute timeout, 10 GB memory). Some services might need ECS tasks, Step Functions, or EC2-based processing. SQS queues provide flexibility — any consumer can poll SQS regardless of compute platform.

### C. Direct SQS queues — ❌ Incorrect

- **Order Service sends to 3 separate queues**: The producer must know all consumers and maintain code to send to each queue. Adding a 4th consumer requires modifying Order Service — tight coupling at the producer level. This violates "new consumers without modifying producers."
- **No event routing/filtering**: Direct SQS means every consumer receives every message. There's no filtering — Notification Service receives inventory events it doesn't need. EventBridge rules filter events per consumer (e.g., only `OrderCreated` events with `status=shipped` trigger Notification Service).
- **S3 for archiving**: Requires custom code to copy messages from SQS to S3 before deletion. No replay capability — archived messages in S3 can't be re-injected into queues without custom automation. EventBridge Archive handles this natively.

### D. Apache Kafka on MSK — ❌ Incorrect

- **MSK is operationally heavy**: Managing a Kafka cluster (broker sizing, partition rebalancing, ZooKeeper/KRaft management, replica synchronization, storage scaling) requires dedicated expertise. EventBridge is fully serverless — zero infrastructure management.
- **90-day retention in Kafka**: Kafka can retain messages for 90 days, but this requires significant storage (500K messages/sec × 90 days = petabytes). Storage costs on MSK are high compared to EventBridge Archive (which compresses and stores events cost-effectively).
- **Kafka exactly-once semantics**: Kafka's exactly-once requires: idempotent producers + transactional consumers + specific client configurations. It's achievable but complex — SQS FIFO deduplication is simpler for event-level deduplication.
- **Kafka is the right choice** for high-throughput streaming (millions of events/sec with < 10 ms latency). For 500K messages/sec with event routing and serverless consumers, EventBridge + SQS is simpler, cheaper, and fully managed.
- If the company needs Kafka-level throughput and streaming analytics (windowed aggregations, stream joins), MSK would be appropriate. For event-driven microservice orchestration, EventBridge is the better fit.

## Recommendations

- **EventBridge + SQS** is the default event-driven architecture pattern for AWS microservices — EventBridge for routing, SQS for buffering.
- **Use EventBridge rules for content-based routing** — filter events by `source`, `detail-type`, or any JSON path in the event payload. This eliminates unnecessary processing by consumers.
- **SQS FIFO for exactly-once** where needed (inventory, payments). Standard SQS for at-least-once where idempotency is already handled by the consumer (notifications, analytics).
- **EventBridge Archive + Replay** is essential for debugging production issues — "replay the last hour of OrderCreated events to see what happened."
- **Dead-letter queues on EVERY SQS queue** — unprocessed messages should be captured, not silently dropped.
- **EventBridge has a 256 KB event size limit** — for larger payloads, store the data in S3 and include the S3 key in the event (claim-check pattern).

## Relevant Links

- [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)
- [EventBridge Archives and Replay](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-archive.html)
- [EventBridge Schema Registry](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-schema.html)
- [SQS FIFO Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)
- [EventBridge to SQS FIFO](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html)
