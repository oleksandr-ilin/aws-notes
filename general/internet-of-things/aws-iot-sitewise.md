Overview
Managed service for collecting, organizing and visualizing industrial equipment data (OT to IT bridge).

Tasks
- Model assets, ingest telemetry, create dashboards and export data for analytics.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Time-series DB + custom dashboards | Flexible | More development effort |
| Third‑party industrial platforms | Domain features | Cost & integration |

Limitations
- Focused on industrial telemetry and asset modeling; not a general-purpose time-series DB.

Price info
- Charged for data ingestion, storage and gateway usage.

Network & multi-region
- Regional service; gateways often deployed on-prem with cloud aggregation.

Popular use cases
- Industrial monitoring, equipment KPIs, and predictive maintenance pipelines.

When Not to Use
- Generic IoT telemetry without asset modeling needs.

DR strategy
- Export asset models and data to S3 and ensure gateway configurations are versioned.
