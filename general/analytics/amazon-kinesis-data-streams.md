Overview
Amazon Kinesis Data Streams provides a sharded, durable log for real-time streaming data ingestion.

Tasks
- Create stream with appropriate shard count, produce/consume records, manage retention and scaling.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Amazon MSK / Kafka | Kafka ecosystem | More operational overhead

Limitations
- Shard limits and management; retention and throughput planning required.

Price info
- Charged per shard-hour and per-PUT payload units.

Network & multi-region
- Streams are regional; use cross-region replication for failover or fanout.

Popular use cases
- High-throughput telemetry, clickstreams, and real-time analytics pipelines.

When Not to Use
- Very high-fidelity Kafka features are required.

DR strategy
- Use enhanced fan-out, cross-region replication, and durable S3 archiving of stream data.
