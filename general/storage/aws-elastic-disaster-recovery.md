Overview
Orchestrated disaster recovery service (replaces CloudEndure) to recover servers into AWS.

Tasks
- Configure replication, runbook automation, and recovery orchestration.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| CloudEndure (legacy) | Proven migrations | Legacy product lifecycle |
| Custom DR runbooks + backups | Flexible | Higher ops burden |

Limitations
- Requires replication agents and bandwidth; some OS/application specifics require tuning.

Price info
- Charges for replication and staging resources; recovery compute billed separately.

Network & multi-region
- Typically replicate to a target region; ensure adequate network bandwidth and testing windows.

Popular use cases
- Fast RTO for critical servers, cloud-first DR strategies.

When Not to Use
- Non-critical workloads where snapshot+restore is sufficient.

DR strategy
- Regular DR drills, validate failover procedures and DNS/Route53 cutover.
