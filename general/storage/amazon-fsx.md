Overview
Managed high-performance file systems (Lustre, Windows File Server, ONTAP) for demanding workloads.

Tasks
- Choose FSx variant, size storage/throughput, configure backups and Windows integration if needed.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| EFS | General NFS | Lower peak IO performance |
| Self-hosted file servers on EC2 | Full control | Operational cost and complexity |

Limitations
- Each FSx flavor targets specific workloads; choose based on OS/IOPS needs.

Price info
- Per-GB-month + throughput/IOPS depending on flavor; backup/storage costs apply.

Network & multi-region
- Regional service; cross-region DR via backups or replication features where available.

Popular use cases
- HPC, Windows file shares, high-throughput media processing.

When Not to Use
- Simple Linux shared storage—EFS is usually simpler and cheaper.

DR strategy
- Use built-in backups and cross-region copy where supported; test recovery.
