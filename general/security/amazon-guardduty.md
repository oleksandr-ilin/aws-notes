Overview
Managed threat detection service that continuously monitors for malicious activity and unauthorized behavior.

Tasks
- Enable across accounts, configure findings export and automated response integrations.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| SIEM + custom detection | Centralized visibility | Heavy ops & tuning |
| Third‑party detection | Specialized features | Extra cost & integration |

Limitations
- Focused on AWS data sources; may miss non-AWS telemetry without integrations.

Price info
- Billed based on data analyzed (CloudTrail, VPC Flow Logs, DNS logs) and accounts/regions.

Network & multi-region
- Enable per-region; use centralized findings aggregation across accounts/regions.

Popular use cases
- Continuous threat detection, baseline anomaly alerts, and automated remediation playbooks.

When Not to Use
- Small dev accounts where manual monitoring is sufficient.

DR strategy
- Archive findings to S3 and ensure playbooks/response runbooks are stored in versioned repos.
