Overview
Data discovery and classification service focused on S3 to find PII and sensitive data.

Tasks
- Enable Macie for S3 buckets, configure classifiers and alerting.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom DLP scripts | Flexible | Higher false positives & ops |
| Third‑party DLP | Enterprise features | Cost & integration |

Limitations
- S3-focused; false positives require tuning and review.

Price info
- Charges based on data classified and number of S3 objects.

Network & multi-region
- Enable per-region and aggregate results centrally for multi-account setups.

Popular use cases
- Finding PII in S3, compliance scans, data inventory.

When Not to Use
- Data stores outside S3 require other tools.

DR strategy
- Archive classification reports to versioned S3 buckets and include in compliance runbooks.
