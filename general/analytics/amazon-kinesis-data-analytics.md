Overview
Amazon Kinesis Data Analytics runs SQL or Apache Flink applications on streaming data for real-time analytics.

Tasks
- Create application, configure input/output streams, define SQL/Flink transformations and monitor application metrics.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Apache Flink on EKS | Full control | More ops

Limitations
- State size and scaling constraints; Flink skills may be required for advanced apps.

Price info
- Charged for application units or resources consumed by the runtime.

Network & multi-region
- Typical per-region deployments; consider cross-region replication for failover.

Popular use cases
- Real-time stream aggregations, anomaly detection, and analytics over clickstreams.

When Not to Use
- Very high stateful workloads better suited for self-managed Flink clusters.

DR strategy
- Use checkpointing and multi-region stream replication where supported.
