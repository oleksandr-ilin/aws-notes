Overview
Automated vulnerability assessment service for EC2 and container images.

Tasks
- Run assessments, review findings, and integrate with Security Hub for aggregation.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Third‑party vulnerability scanners | Broader language & ecosystem | Cost & integration |
| Self-scanning with Open-source tools | Low cost | Requires orchestration |

Limitations
- Coverage depends on supported resources and agent deployment.

Price info
- Charged per assessment run and resources scanned.

Network & multi-region
- Enable per-region; aggregate results centrally via Security Hub.

Popular use cases
- Periodic vulnerability scans, container image checks, and automated reporting.

When Not to Use
- Very custom binaries or unsupported OS images—use specialist scanners.

DR strategy
- Archive scan results and remediation runbooks; ensure automated patching playbooks are available.
