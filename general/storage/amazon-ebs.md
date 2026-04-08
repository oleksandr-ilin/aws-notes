Overview
Block storage for EC2 instances with snapshot capability.

Tasks
- Attach volumes to EC2, manage IOPS/throughput, take snapshots for backups.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| EFS | Shared filesystem | Higher latency for block-like workloads |
| Local instance store | Lower latency | Ephemeral, not durable |

Limitations
- EBS volumes are AZ-scoped; cross-AZ access requires snapshot/attach workflows.

Price info
- Charged per-GB provisioned; extra for provisioned IOPS.

Network & multi-region
- Use snapshots to copy across regions; cross-AZ access via instance migration.

Popular use cases
- Databases, boot volumes, stateful services requiring low-latency block storage.

When Not to Use
- Shared file access across many instances—prefer EFS or FSx.

DR strategy
- Regular snapshot schedule, cross-region snapshot copy, and AMI-based recovery.
