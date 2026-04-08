# Amazon SNS

## Service Overview
Amazon SNS is a fully managed pub/sub messaging service for high-throughput push notifications to subscribers and endpoints.

## Tasks this service can solve
- Fan-out notifications to multiple endpoints (email, SMS, HTTP, SQS)
- Application alerts and event fan-out
- Mobile push notifications

## Alternatives
| Name | Type | Short description | Pro vs SNS | Cons vs SNS | Price comparison |
|---|---|---:|---|---|---|
| EventBridge | AWS | Event routing with filtering | Advanced routing and schema | Less optimized for simple fan-out | SNS often cheaper for high-volume push notifications |
| SQS | AWS | Durable queueing | Guarantees delivery ordering with FIFO queues | Not a push-notification service | Different pricing model (per-request) |
| Third-party providers (Twilio, SendGrid) | Commercial | Specialized messaging | Rich delivery features | External dependency/cost | Varies; sometimes higher than SNS for SMS/email |

## Limitations
- Message size limits and delivery retries; SMS/phone delivery costs vary by region.
- No built-in message persistence beyond short retries (use SQS for durability).

## Price info
- Charged per request and per notification delivery type (e.g., SMS has per-message carrier costs); data transfer may apply.

## Network & Multi-region considerations
- SNS is regional; use cross-region replication patterns or publish to multiple regions for global coverage.
- Integrates with SQS, Lambda, HTTP endpoints, and mobile push services.

## Popular use cases
- Application alerting and monitoring notifications
- Fan-out to multiple downstream systems
- Mobile push and SMS messaging for user engagement

## When Not to Use This Service
- Not suitable when durable queuing and guaranteed ordered processing are required; use SQS or Kafka/MSK for those patterns.

## DR strategy
- For critical notifications, publish to multiple regions or route messages through durable stores (S3 or cross-region queues). Combine SNS with SQS subscriptions to ensure message durability and replayability across regions.
