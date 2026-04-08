Overview
Highly durable object storage for general-purpose data, hosting, analytics and backups.

Tasks
- Manage buckets, lifecycle rules, versioning, encryption and access policies.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Azure Blob / GCS | Multi-cloud parity | Different APIs and features |
| On-prem object stores | Full control | Ops overhead |

Limitations
- Object semantics (eventual consistency for some ops historically) and request costs for many small objects.

Price info
- Per-GB-month with tiered classes; requests and data transfer billed separately.

Network & multi-region
- Global namespace; use cross-region replication for DR and multi-region durability.

Popular use cases
- Static website hosting, data lakes, backups, archive storage.

When Not to Use
- Low-latency block or POSIX file needs—use EBS or EFS/FSx.

DR strategy
- Enable versioning, lifecycle to Glacier, and cross-region replication for critical buckets.
