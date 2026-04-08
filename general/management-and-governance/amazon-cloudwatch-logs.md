Overview
Centralized log ingestion, retention and query capabilities for application and infra logs.

Tasks
- Configure log groups, retention, metric filters and queries for analysis and alerting.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| ELK / OpenSearch | Powerful query features | Management overhead |

Limitations
- Cost for ingestion and retention; query costs for log insights.

Price info
- Per-GB ingestion + storage + query costs.

Network & multi-region
- Regional; consider cross-region replication or central aggregation for multi-region setups.

Popular use cases
- Application logging, audit logs, and operational troubleshooting.

When Not to Use
- Very large-scale log stores where specialized data lakes are more cost-effective.

DR strategy
- Export logs to S3 for archival and compliance purposes.
