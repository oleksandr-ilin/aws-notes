# Q04: Multi-Account Event Notification Architecture

## Question

A company with 60 AWS accounts needs a centralized event notification system. The security team must receive real-time alerts when any of the following events occur across any account: root user login, IAM access key creation, changes to security groups, CloudTrail being disabled, or GuardDuty findings with HIGH severity. The alerts should be delivered to a PagerDuty integration for on-call response and to an Amazon S3 bucket for historical analysis.

Which architecture should the solutions architect recommend?

## Options

- **A.** Configure an Amazon EventBridge organization event bus in the security account. Create EventBridge rules in each member account to forward matching events to the central event bus. In the security account, create rules to route events to an Amazon API Gateway endpoint (PagerDuty webhook) and Amazon Kinesis Data Firehose (for S3 delivery).
- **B.** Enable GuardDuty with a delegated administrator account. Create an EventBridge rule in the delegated admin account for HIGH severity findings. For other events, deploy per-account EventBridge rules with SNS targets that fan out to SQS queues in the security account.
- **C.** Create a CloudWatch Logs destination in the security account. Configure CloudTrail in each account to send logs to both S3 and the central CloudWatch Logs destination. Create metric filters and alarms for the target events.
- **D.** Deploy a centralized SIEM (Security Information and Event Management) solution on Amazon OpenSearch Service. Ingest CloudTrail logs from all accounts and create alerting rules for the target events.

## Answers

### A. EventBridge organization event bus — ✅ Correct

Amazon EventBridge with an **organization event bus** architecture provides:
- **Centralized event routing**: Member accounts forward events to the central event bus without individual SNS/SQS configuration per account.
- **Event pattern matching**: Rules can match specific event patterns (root login, IAM key creation, security group changes, CloudTrail modifications, GuardDuty findings by severity).
- **Multiple targets**: A single event can trigger both the PagerDuty API Gateway webhook and Kinesis Data Firehose for S3 archival simultaneously.
- **Scalability**: Adding new accounts automatically works when using organization-level event forwarding.
- **Low latency**: Near real-time event delivery (seconds).

### B. Mixed GuardDuty + per-account EventBridge — ❌ Incorrect

This approach handles GuardDuty findings well (delegated admin aggregates findings natively) but creates a fragmented architecture for other event types. Per-account EventBridge rules with SNS topics and SQS queues in 60 accounts is operationally complex. A centralized event bus is simpler and more consistent.

### C. CloudWatch Logs with metric filters — ❌ Incorrect

CloudWatch metric filters work but are limited in pattern matching compared to EventBridge rules. Streaming CloudTrail to CloudWatch Logs across 60 accounts is expensive (CloudWatch Logs ingestion charges). EventBridge is specifically designed for event-driven architectures and provides richer routing capabilities.

### D. SIEM on OpenSearch — ❌ Incorrect

OpenSearch provides powerful search and analytics but requires significant infrastructure (instances, storage, management) and introduces higher complexity and cost. It's appropriate for deep investigation and correlation but is over-engineered for the real-time alerting use case described. EventBridge provides the alerting layer; OpenSearch might complement it for investigation.

## Recommendations

- Use **EventBridge cross-account event bus** patterns for centralized event processing across Organizations.
- For GuardDuty specifically, use the **delegated administrator** feature which natively aggregates findings — no custom event routing needed.
- **EventBridge API destinations** can directly integrate with third-party services (PagerDuty, Slack, Datadog) without API Gateway as an intermediary.
- Archive events using **EventBridge Archive and Replay** for debugging and audit purposes.
- Consider **EventBridge Pipes** for transforming events before delivery to targets.

## Relevant Links

- [EventBridge Cross-Account Events](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cross-account.html)
- [EventBridge API Destinations](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-api-destinations.html)
- [GuardDuty Delegated Admin](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_organizations.html)
- [EventBridge Event Patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)
