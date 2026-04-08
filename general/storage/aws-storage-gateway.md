Overview
Hybrid storage appliance (virtual or hardware) to present local files/volumes backed by S3/EBS.

Tasks
- Deploy gateway VM/appliance, configure volumes or file shares, and set caching/backup policies.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Direct upload to S3 | Simpler | Requires app changes |
| Third-party gateway appliances | More features | Cost & vendor lock-in |

Limitations
- Appliance management, cache sizing, and network dependency for cloud-backed data.

Price info
- VM/hour + storage transfer and S3/EBS costs.

Network & multi-region
- Gateway connects to a region; plan bandwidth and use multipart transfers for large datasets.

Popular use cases
- Lift-and-shift backups, tape replacement, caching local datasets to S3.

When Not to Use
- Cloud-native apps that can natively use S3—skip the gateway.

DR strategy
- Use snapshots and replicate backed S3 data cross-region; test restores from cloud-backed volumes.
