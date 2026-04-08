Overview
Customer-managed hardware security modules (HSMs) for FIPS-compliant key storage and cryptographic operations.

Tasks
- Provision HSM clusters, manage partitions, and integrate with applications requiring HSM-backed keys.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| AWS KMS | Managed keys | Less control & not dedicated HW |
| On-prem HSM | Full control | Procurement & ops overhead |

Limitations
- Higher cost and operational management; requires HSM-aware applications.

Price info
- HSM cluster hourly billing plus network and management costs.

Network & multi-region
- Regional service; design for HA across AZs and cross-region key replication handled externally.

Popular use cases
- Payment processors, PKI backends, and FIPS-level cryptographic workloads.

When Not to Use
- General application encryption where KMS suffices.

DR strategy
- Backup HSM configurations, ensure key escrow and documented recovery with secure processes.
