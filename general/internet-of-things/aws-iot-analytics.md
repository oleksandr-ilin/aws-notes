Overview
Managed analytics service tailored for IoT device data, simplifying time-series analysis and transformations.

Tasks
- Ingest device telemetry, run pipelines, and export processed datasets for downstream analytics.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Kinesis + custom analytics | Flexible | More ops and plumbing |
| Self-hosted time-series DB | Control | Operational overhead |

Limitations
- Best suited for IoT-specific ingestion patterns; not a general-purpose analytics platform.

Price info
- Charged per ingestion and storage/processing of data.

Network & multi-region
- Typically region-scoped; plan ingestion endpoints near device gateways and replicate results as needed.

Popular use cases
- Pre-processing telemetry for machine learning, aggregations for dashboards, and anomaly detection.

When Not to Use
- Non-IoT data at large scale—use generalized analytics stacks instead.

DR strategy
- Export processed datasets to S3 and replicate across regions; keep pipeline definitions in IaC.
