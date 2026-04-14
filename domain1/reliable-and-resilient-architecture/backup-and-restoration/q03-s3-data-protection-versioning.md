# Q03: S3 Data Protection and Versioning Strategy

## Question

A genomics research company stores petabytes of research data in Amazon S3. Researchers occasionally **accidentally delete or overwrite** critical datasets. The company needs:

1. Protection against accidental deletion of objects
2. Ability to recover any version of an object from the **last 90 days**
3. Older versions should be automatically moved to cheaper storage after 30 days
4. Protection against **malicious bulk deletion** (e.g., compromised credentials)
5. Compliance with data retention regulations requiring certain datasets to be immutable for 7 years

Which combination of S3 features should be configured? (Select FOUR)

## Options

- **A.** Enable S3 Versioning on the bucket.
- **B.** Enable MFA Delete on the bucket.
- **C.** Configure S3 Lifecycle rules to transition non-current versions to S3 Glacier after 30 days and expire them after 90 days.
- **D.** Configure S3 Lifecycle rules to delete all objects older than 90 days.
- **E.** Enable S3 Object Lock in **Compliance mode** with a 7-year retention period for regulated datasets.
- **F.** Enable S3 Object Lock in **Governance mode** for all objects.
- **G.** Enable S3 Cross-Region Replication to a vault bucket.
- **H.** Configure S3 bucket policy to deny `s3:DeleteObject` for all principals.

## Answers

### A. S3 Versioning — ✅ Correct

Versioning is the foundation for object recovery. When enabled, S3 keeps all versions of an object. Deleting an object creates a **delete marker** (soft delete) — the previous versions still exist and can be restored. Overwriting creates a new version while preserving the old one. This directly addresses requirements 1 and 2.

### B. MFA Delete — ✅ Correct

MFA Delete requires **multi-factor authentication** to permanently delete an object version or change the versioning state of the bucket. This protects against malicious bulk deletion from compromised credentials alone — the attacker would also need the MFA device. This addresses requirement 4.

### C. Lifecycle for non-current versions — ✅ Correct

S3 Lifecycle rules for **non-current versions** can transition old versions to cheaper storage classes (S3 Glacier after 30 days) and expire them after 90 days. This meets requirements 2 and 3: 90-day recovery window with cost optimization by moving older versions to Glacier.

### E. Object Lock in Compliance mode — ✅ Correct

S3 Object Lock in Compliance mode provides **WORM** (write-once-read-many) protection. In Compliance mode, no one — including root — can delete or overwrite the object until the retention period expires. A 7-year retention period meets the immutability requirement for regulated datasets (requirement 5).

### D. Delete all objects after 90 days — ❌ Incorrect

This would delete the **current** (live) objects after 90 days, not just old versions. The requirement is to expire non-current versions after 90 days, not to delete current data. Option C correctly targets non-current versions.

### F. Object Lock in Governance mode — ❌ Incorrect

Governance mode allows users with **special IAM permissions** (`s3:BypassGovernanceRetention`) to override the lock and delete objects. This doesn't meet the compliance requirement for true immutability. Compliance mode (option E) is required for regulatory immutability.

### G. Cross-Region Replication — ❌ Incorrect

While CRR provides geographic redundancy, it replicates deletions too (including delete markers). CRR alone doesn't protect against accidental deletion — a delete in the source bucket would replicate to the destination. Versioning + MFA Delete is the correct approach for deletion protection.

### H. Deny DeleteObject — ❌ Incorrect

A blanket deny on `s3:DeleteObject` would prevent legitimate data management and cleanup. It also doesn't protect against overwriting objects (PutObject). This is too restrictive for operational use and doesn't address version recovery or compliance retention.

## Recommendations

- **S3 data protection layers** (know all of them):
  1. **Versioning:** Soft delete protection + version history
  2. **MFA Delete:** Protects against credential compromise
  3. **Object Lock:** Immutability (Governance = soft, Compliance = hard)
  4. **Lifecycle rules:** Automate version management and cost optimization
  5. **CRR:** Geographic redundancy (not deletion protection)
- **Object Lock modes:**
  - **Governance:** Can be bypassed with IAM permissions — for operational protection
  - **Compliance:** Cannot be bypassed by anyone — for regulatory requirements
- **Lifecycle tip:** Rules apply to either "current" or "non-current" versions. Know the difference.
- **MFA Delete + Versioning** is the standard answer for "protect against malicious/accidental deletion."

## Relevant Links

- [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [S3 MFA Delete](https://docs.aws.amazon.com/AmazonS3/latest/userguide/MultiFactorAuthenticationDelete.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [S3 Lifecycle Rules for Non-Current Versions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-configuration-examples.html)
