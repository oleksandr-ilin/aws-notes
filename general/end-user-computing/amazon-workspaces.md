Overview
Managed virtual desktop service for persistent Windows or Linux desktops.

Tasks
- Provision WorkSpaces bundles, configure user directory integration, manage storage and bundles.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| RDP on EC2 | Full control | Operational burden |
| On-prem VDI | Local performance | Hardware & ops cost |

Limitations
- Latency-sensitive; storage and GPU bundle costs can be high.

Price info
- Per-user monthly or hourly pricing depending on bundle type.

Network & multi-region
- Deploy WorkSpaces in-region; plan for bandwidth and network paths for end-users.

Popular use cases
- Remote workforce, contractors, secure access to corporate desktops.

When Not to Use
- High-performance local workstations or offline workflows.

DR strategy
- Export user data regularly, ensure directory recovery options and IAM/service account backups.
