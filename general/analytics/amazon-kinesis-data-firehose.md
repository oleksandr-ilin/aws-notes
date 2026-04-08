Overview
Amazon Kinesis Data Firehose is a fully managed service to load streaming data into destinations like S3, Redshift, and OpenSearch.

Tasks
- Create delivery stream, configure buffering, set transforms (Lambda) and choose destination.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Kafka Connect | Flexibility | More ops

Limitations
- Limited transformation complexity; best suited for delivery rather than heavy processing.

Price info
- Charged per-GB ingested and optional Lambda transforms.

Network & multi-region
- Region-scoped delivery; plan destination endpoints and IAM policies.

Popular use cases
- Ingest logs, telemetry and stream to data lakes or analytics services.

When Not to Use
- Complex stream processing needs requiring full stream processing engines.

DR strategy
- Buffering plus delivery to durable S3 buckets and cross-region replication for resiliency.
