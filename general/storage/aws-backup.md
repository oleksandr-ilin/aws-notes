Overview
Centralized backup orchestration service for supported AWS resources and on‑prem via Storage Gateway.

Tasks
- Schedule and manage backups, lifecycle policies, and restores across services.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom scripts + S3 lifecycle | Flexible | Operational overhead |
| Third‑party backup appliances | Feature-rich | Extra cost & integration |

Limitations
- Not all AWS services have first‑class integration.
- Restore granularity varies by service.

Price info
- Billed for backup storage and restores; tiered storage classes affect cost.

Network & multi-region
- Backups stored in-region; replicate snapshots manually or use cross‑region copy for DR.

Popular use cases
- Central policy-driven backups for RDS, EBS, EFS, DynamoDB exports.

When Not to Use
- Very custom backup workflows or exotic file systems—use dedicated tools.

DR strategy
- Combine with cross-region snapshot copy and test restores as part of runbooks.
