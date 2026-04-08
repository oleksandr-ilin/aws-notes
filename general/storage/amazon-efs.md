Overview
Managed NFS file system for Linux workloads with regional availability across AZ mount targets.

Tasks
- Create file systems, choose performance/throughput modes, manage mount targets.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| FSx for Lustre | High I/O | Less generic POSIX semantics |
| EBS + NFS server | Full control | Operational complexity |

Limitations
- Performance modes and throughput considerations; latency higher than local block.

Price info
- Per-GB-month plus optionally provisioned throughput charges.

Network & multi-region
- Mount targets exist per AZ; cross-region replication via DataSync or third-party tools.

Popular use cases
- Shared home directories, content repositories, lift-and-shift apps needing NFS.

When Not to Use
- Low-latency block workloads—use EBS.

DR strategy
- Use backups and cross-region replication with DataSync or lifecycle policies.
