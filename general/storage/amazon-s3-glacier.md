Overview
Low-cost archival storage classes within S3 (Glacier Instant/Deep Archive) for long-term retention.

Tasks
- Configure lifecycle transitions to Glacier storage classes and manage retrieval policies.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Third-party cold storage | Customized SLAs | Integration overhead |
| On-prem tape | Offline storage | Logistics & restore time |

Limitations
- Retrieval times and costs vary by retrieval tier; not suitable for frequent access.

Price info
- Very low storage cost; retrieval and early deletion fees apply.

Network & multi-region
- Use with S3 replication to copy archives across regions for added resilience.

Popular use cases
- Compliance archives, long-term backups, infrequently accessed archives.

When Not to Use
- Frequently-accessed data—use S3 Standard or Infrequent Access classes.

DR strategy
- Keep at least one copy in a separate region if regulatory/availability needs demand.
