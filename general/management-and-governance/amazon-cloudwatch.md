Overview
Monitoring, metrics, alarms, dashboards and operational insights for AWS resources.

Tasks
- Collect metrics, create alarms/dashboards, configure logs and set up insights/alarms.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Prometheus + Grafana | Open-source & flexible | Self-management required |

Limitations
- Cost for high cardinality metrics and large log volumes; query costs apply.

Price info
- Metrics, logs ingestion, dashboards and dashboards alarms billed separately.

Network & multi-region
- Regional; aggregate metrics centrally if needed for multi-region fleets.

Popular use cases
- Operational monitoring, SLO alerts, and system dashboards.

When Not to Use
- For deep custom analytics without AWS integration—consider external stacks.

DR strategy
- Archive logs/metrics to S3 for long-term retention and re-ingest if needed.
