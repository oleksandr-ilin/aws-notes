# Q02: S3 Cross-Region Replication with Versioning and Compliance

## Question

A healthcare company must replicate patient imaging data (DICOM files, 50-500 MB each) from a primary S3 bucket in us-east-1 to a DR bucket in eu-west-1. Compliance requirements:

- All objects must be replicated within 15 minutes of upload (SLA-backed)
- Replicated objects must retain the same storage class, encryption (SSE-KMS with a customer-managed key), and object tags
- Delete operations must NOT be replicated — if an object is accidentally deleted in the primary, the DR copy must survive
- Objects in the DR bucket must be immutable for 7 years (WORM compliance — cannot be deleted or overwritten by anyone, including root)
- Replication must continue working even if new object tags are added or existing tags are modified

Which S3 configuration meets all requirements?

## Options

- **A.** Enable S3 Cross-Region Replication (CRR) with S3 Replication Time Control (S3 RTC) on the source bucket. Configure the replication rule to: replicate SSE-KMS encrypted objects using a destination-Region KMS key, replicate object tags, and exclude delete markers from replication. Enable S3 Object Lock in Compliance mode on the destination bucket with a 7-year retention period. Enable versioning on both buckets.
- **B.** Enable S3 CRR without S3 RTC. Use S3 Batch Replication for any objects missed. Replicate with SSE-S3 encryption in the destination (simpler key management). Enable S3 Object Lock in Governance mode on the destination bucket. Enable versioning on both buckets.
- **C.** Use S3 Same-Region Replication (SRR) to a secondary bucket in us-east-1, then CRR from the secondary bucket to eu-west-1. Enable S3 RTC on both replication rules. Use S3 Glacier Vault Lock for WORM compliance on the destination.
- **D.** Use a Lambda function triggered by S3 event notifications to copy objects cross-Region using multi-part upload. Use DynamoDB to track replication status. Enable S3 Object Lock in Compliance mode on the destination. Disable versioning to prevent version proliferation.

## Answers

### A. CRR with S3 RTC + KMS re-encryption + exclude delete markers + Object Lock Compliance — ✅ Correct

Every requirement is addressed by a specific S3 feature:

- **S3 Cross-Region Replication (CRR) with Replication Time Control (S3 RTC)**:
  - S3 RTC provides an SLA: 99.99% of objects replicated within 15 minutes. Without RTC, replication is "best effort" with no time guarantee — most objects replicate within seconds, but some may take hours.
  - S3 RTC also provides replication metrics (bytes pending, replication latency) and CloudWatch notifications when objects don't replicate within the threshold — enabling monitoring and alerting.
  - **15-minute SLA requirement** directly maps to S3 RTC's guarantee.

- **SSE-KMS re-encryption with destination-Region key**:
  - Objects encrypted with SSE-KMS in the source Region must be decrypted and re-encrypted with a KMS key in the destination Region. CRR supports this natively — you specify the destination KMS key in the replication configuration.
  - **Why destination-Region key**: KMS keys are Regional. Using the same key ARN in the destination requires a multi-Region KMS key. Using a separate destination key is simpler and provides Region-independent key management.
  - **IAM permissions required**: The replication role needs `kms:Decrypt` on the source key and `kms:Encrypt` on the destination key.

- **Replicate object tags, exclude delete markers**:
  - CRR replication rules support granular control: `ReplicateDeletes: Disabled` means delete markers and object deletions are NOT replicated. If someone deletes a file in the source bucket, the DR copy survives.
  - Tag replication is enabled by default in CRR — tags are metadata that travels with the object.

- **S3 Object Lock in Compliance mode**:
  - **Compliance mode**: Objects cannot be deleted or overwritten by ANY user, including the AWS account root user, for the retention period (7 years). This meets WORM (Write Once Read Many) requirements for healthcare data.
  - **Governance mode**: Allows users with `s3:BypassGovernanceRetention` permission to delete/modify — this does NOT meet WORM compliance because a privileged user can bypass it.
  - Object Lock requires versioning enabled (which is already required for CRR).

- **Versioning on both buckets**: Required for both CRR (source and destination must have versioning enabled) and S3 Object Lock (locks are applied to specific object versions).

### B. CRR without RTC + SSE-S3 + Governance mode — ❌ Incorrect

- **No S3 RTC**: Without RTC, there's no SLA on replication timing. AWS documentation states replication "typically completes within seconds" but "some objects may take longer." For a compliance requirement of 15-minute replication SLA, "best effort" is insufficient — S3 RTC is required.
- **SSE-S3 instead of SSE-KMS**: Changing encryption from SSE-KMS to SSE-S3 means losing customer-managed key control. The DR copy uses Amazon-managed keys — the company can't control key rotation, key policies, or audit key usage. The requirement is to retain the same encryption type (SSE-KMS with CMK).
- **Governance mode**: Users with `s3:BypassGovernanceRetention` permission can delete objects before the retention period. A compromised admin account or insider threat can bypass Governance mode. Compliance mode is required for true WORM.
- **S3 Batch Replication**: Useful for backfilling objects that existed before CRR was enabled, or for retrying failed replications. But it's not a substitute for real-time CRR with SLA guarantees.

### C. SRR + CRR chain + Glacier Vault Lock — ❌ Incorrect

- **Two-hop replication (SRR then CRR)**: Adds unnecessary complexity and latency. Each replication hop adds time — two hops with S3 RTC means 15 minutes × 2 = potentially 30 minutes end-to-end. Direct CRR from source to destination is simpler and meets the 15-minute SLA in a single hop.
- **S3 Glacier Vault Lock**: Vault Lock is for S3 Glacier vaults (the legacy Glacier API), not S3 buckets. S3 Object Lock is the WORM mechanism for S3 buckets. Glacier Vault Lock applies to archives stored via the Glacier API — mixing these creates architectural confusion.
- **Why SRR in the first place?**: There's no stated requirement for same-Region replication. The secondary bucket in us-east-1 adds cost (2× storage in the primary Region) with no benefit.

### D. Lambda-based replication + disabled versioning — ❌ Incorrect

- **Lambda for cross-Region copy**: Re-implements what S3 CRR does natively. Custom Lambda replication requires: handling multi-part uploads for large files (50-500 MB), retry logic, error handling, DynamoDB tracking, IAM permissions management, and monitoring. S3 CRR handles all of this in a fully managed way.
- **Lambda limitations**: Lambda has a 15-minute maximum timeout. A 500 MB file over multi-part upload may approach this limit. Lambda's /tmp storage is 10 GB — sufficient but requires careful chunked streaming for large files.
- **Disabled versioning**: S3 Object Lock REQUIRES versioning enabled. You cannot enable Object Lock on a bucket without versioning. Disabling versioning to "prevent version proliferation" breaks Object Lock entirely. This configuration is technically impossible — the bucket creation would fail.
- **DynamoDB for tracking**: S3 RTC provides native replication metrics and CloudWatch integration — no custom tracking needed.

## Recommendations

- **S3 RTC** is required whenever replication has an SLA — always enable it for compliance-critical data.
- **S3 Object Lock Compliance mode** is the only way to achieve true WORM — Governance mode is insufficient for regulatory compliance (HIPAA, SEC 17a-4, etc.).
- **Exclude delete marker replication** for DR scenarios — accidental or malicious deletions in the source should not cascade to the DR copy.
- **Use separate KMS keys per Region** for cross-Region encrypted replication — simpler key management and avoids multi-Region key complexity.
- **Enable S3 Inventory** on both buckets to verify replication completeness — compare source and destination inventories weekly.
- **S3 Batch Replication** is the backfill tool — use it after enabling CRR on existing buckets to replicate objects uploaded before CRR was configured.

## Relevant Links

- [S3 Cross-Region Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [S3 Replication Time Control](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-time-control.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Replicating SSE-KMS Encrypted Objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html)
- [S3 Batch Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-batch-replication-batch.html)
