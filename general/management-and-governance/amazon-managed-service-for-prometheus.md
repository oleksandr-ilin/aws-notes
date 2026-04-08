Overview
Managed Prometheus-compatible service for ingesting and storing metrics at scale.

Tasks
- Configure scraping, remote write, and alerting rules; integrate with Grafana.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Self-hosted Prometheus | Full flexibility | Scaling & HA complexity

Limitations
- Ingest and storage limits; query performance considerations.

Price info
- Ingest + storage pricing; retention costs apply.

Network & multi-region
- Regional; plan metric federation or central aggregation for multi-region needs.

Popular use cases
- Container metrics, service-level monitoring, and alerting.

When Not to Use
- Very low-scale metrics where self-hosted may be cheaper.

DR strategy
- Archive metrics to long-term storage and export rules/alerts to IaC.
