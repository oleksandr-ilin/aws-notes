Overview
Managed service to ingest, store, and analyze video streams from devices and cameras.

Tasks
- Configure streams, ingest video from producers, and integrate with analytics or playback services.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Self-hosted ingest + object storage | Full control | Scalability & ops burden |
| Third-party video platforms | Integrated features | Cost & integration differences |

Limitations
- Storage and playback costs can grow with retention; integration for analytics requires additional services.

Price info
- Charged for ingest, storage, and data transfer; additional charges for playback and analytics.

Network & multi-region
- Regional ingestion endpoints; plan for edge gateways and cross-region replication for DR.

Popular use cases
- Surveillance ingest, real-time analytics and playback, and IoT camera streams.

When Not to Use
- Bulk archival-only video storage—use S3 with lifecycle policies.

DR strategy
- Store raw streams and processed artifacts in versioned S3 buckets and replicate to a secondary region.
