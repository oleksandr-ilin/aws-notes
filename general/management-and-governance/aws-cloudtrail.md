Overview
Audit trail of API activity and resource changes across AWS accounts.

Tasks
- Enable CloudTrail, configure S3/CloudWatch logging, and set up event selectors and insights.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom logging via CloudWatch Events | Flexibility | More configuration |

Limitations
- Large log volumes and storage costs; fine-grained event selection required.

Price info
- Management events often free; data events and insights billed.

Network & multi-region
- Global service for control plane events; configure multi-region trails for full coverage.

Popular use cases
- Security auditing, forensics, and compliance reporting.

When Not to Use
- For simple monitoring—use CloudWatch metrics/alarms.

DR strategy
- Archive trails to versioned S3 buckets and ensure cross-account secure access for auditors.
