# Q01: Loosely Coupled Architecture with SQS, SNS, and Step Functions

## Question

An e-commerce company processes orders through a monolithic application. The order flow involves: payment validation → inventory reservation → shipment scheduling → notification. Currently, if the shipment system is slow, the entire checkout flow blocks, causing timeouts. Requirements:
- Decouple the order processing steps so that a failure in one step doesn't block the entire flow
- Ensure payment validation completes synchronously (customer must see success/failure immediately)
- Inventory reservation and shipment scheduling can be asynchronous but must execute in order
- If shipment scheduling fails after 3 retries, the order must be flagged for manual review without losing the message
- Multiple downstream systems (analytics, fraud detection, CRM) must receive order events independently

Which architecture should the solutions architect design?

## Options

- **A.** Keep payment validation synchronous in the API layer. After successful payment, publish an "OrderPaid" event to an SNS topic. Subscribe an SQS FIFO queue (with message group ID = order ID) to the SNS topic for ordered processing of inventory → shipment steps, using a Step Functions workflow triggered by the queue. Configure the SQS FIFO queue with a dead-letter queue (maxReceiveCount=3). Subscribe separate SQS standard queues for analytics, fraud detection, and CRM consumers from the same SNS topic.
- **B.** Keep payment validation synchronous. After payment, directly invoke a Lambda function that sequentially calls inventory, shipment, and notification APIs. If shipment fails, the Lambda retries inline up to 3 times. Use EventBridge to broadcast order events to downstream systems.
- **C.** Use API Gateway with a synchronous Step Functions Express Workflow for the entire order flow (payment → inventory → shipment → notification). Configure retries in the Step Functions definition. Use Step Functions output to fan out to downstream systems.
- **D.** Keep payment synchronous. Place all subsequent steps in a single SQS Standard queue. Process messages with a Lambda function that handles inventory, shipment, and notification in sequence. Use SNS for fan-out to downstream systems.

## Answers

### A. Synchronous payment + SNS fan-out + SQS FIFO + Step Functions + DLQ — ✅ Correct

This properly decouples each concern:
- **Synchronous payment**: The API layer calls the payment service directly and returns success/failure to the customer immediately. This is the one step that must be synchronous — the customer needs confirmation.
- **SNS topic for fan-out**: After payment succeeds, an "OrderPaid" event is published to SNS. This decouples the producer (checkout API) from all consumers. Adding a new consumer (e.g., fraud detection) requires only a new SNS subscription — no API changes.
- **SQS FIFO queue for ordered processing**: FIFO guarantees that messages with the same group ID (order ID) are processed in order. Inventory reservation before shipment scheduling is enforced by message ordering, not application logic.
- **Step Functions for orchestration**: A Step Functions Standard Workflow triggered by the FIFO queue orchestrates inventory → shipment as sequential states. If shipment fails, Step Functions handles retries with exponential backoff and configurable max attempts.
- **Dead-letter queue (DLQ)**: After 3 failed processing attempts, the message moves to the DLQ — the order is flagged for manual review. The message is preserved (not lost), and the main queue is not blocked.
- **Separate SQS standard queues per consumer**: Analytics, fraud, and CRM each have their own queue subscribed to the SNS topic. One consumer's slowness doesn't affect others. Standard queues are sufficient for these consumers (order doesn't matter for analytics).

### B. Lambda for sequential processing — ❌ Incorrect

- A single Lambda calling APIs sequentially re-creates the monolith's coupling problem. If shipment is slow, the Lambda runs longer (timeout risk) and payment completion is delayed.
- Inline retries in Lambda consume execution time and can cause Lambda timeout (15-minute max).
- If the Lambda fails mid-execution (after inventory but before shipment), there's no built-in mechanism to resume from the failed step — the entire flow must restart.
- EventBridge for fan-out works but the sequential processing coupling is the fundamental issue.

### C. Step Functions Express for entire flow — ❌ Incorrect

- **Express Workflows** have a 5-minute maximum duration — if any step is slow, the entire workflow times out.
- Making the entire flow synchronous (including inventory and shipment) means the customer waits for all steps — slow shipment scheduling blocks the checkout response.
- Step Functions Express output can fan out, but the synchronous nature defeats the decoupling purpose.
- **Standard Workflows** (up to 1 year duration) are appropriate for asynchronous orchestration, not Express.

### D. Single SQS Standard queue for all steps — ❌ Incorrect

- **SQS Standard** does not guarantee ordering — inventory and shipment steps could execute out of order.
- A single queue means one Lambda processes everything sequentially per message — if shipment processing is slow, it delays inventory processing for subsequent orders.
- No separation of concerns between ordered steps and independent consumers.
- DLQ handling would flag the entire multi-step message, not just the failed step.

## Recommendations

- **Synchronous vs asynchronous**: Only steps requiring immediate customer feedback should be synchronous. Everything else benefits from async processing.
- **SNS + SQS pattern**: SNS publishes once; each subscriber gets its own copy of the message. This is the canonical fan-out pattern in AWS.
- **SQS FIFO** ensures ordered processing but has lower throughput (300 TPS without batching, 3000 with). Use message group IDs to partition ordering by order ID, enabling parallelism across different orders.
- **Step Functions** adds visibility: the execution history shows exactly which step succeeded/failed, with input/output for each state. Critical for debugging order processing issues.
- **DLQ monitoring**: Set a CloudWatch alarm on `ApproximateNumberOfMessagesVisible` on the DLQ. Any message in the DLQ means a processing failure that needs attention.
- Consider **EventBridge Pipes** as an alternative: Pipe SQS → enrichment (Lambda) → Step Functions in a single managed integration.

## Relevant Links

- [Amazon SQS FIFO Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)
- [Amazon SNS Fan-Out](https://docs.aws.amazon.com/sns/latest/dg/sns-sqs-as-subscriber.html)
- [Step Functions Standard vs Express](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-standard-vs-express.html)
- [SQS Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
