# Q02: KMS Encryption Strategy — CMK, Envelope Encryption, and Cross-Account Key Sharing

## Question

A company processes healthcare data (PHI) across 3 AWS accounts: Data Ingestion, Analytics, and Archive. Their compliance team mandates:

- All data at rest must be encrypted with customer-managed KMS keys (not AWS-managed keys)
- The Data Ingestion account encrypts data in S3 and Kinesis Data Streams
- The Analytics account must decrypt the ingested data, process it, and re-encrypt results with its own key
- The Archive account must access encrypted S3 data from both accounts for long-term retention
- Key rotation must occur annually — automatic where possible, manual for asymmetric keys
- If a key is compromised, the company must be able to immediately disable it and prevent all decryption
- Audit trail must show exactly who used which key, for which resource, and when

Which KMS architecture meets all requirements?

## Options

- **A.** Create a separate customer-managed symmetric KMS key (CMK) in each account. The Data Ingestion account's key policy grants `kms:Decrypt` and `kms:DescribeKey` to the Analytics and Archive accounts' IAM roles. The Analytics account uses its own CMK for re-encryption. The Archive account's role has `kms:Decrypt` grants on both accounts' keys. Enable automatic annual key rotation on all symmetric CMKs. CloudTrail logs all KMS API calls in each account. Use `kms:DisableKey` to immediately block decryption if a key is compromised.
- **B.** Create a single KMS multi-Region key in the Data Ingestion account and replicate it to all accounts. Use this single key for all encryption/decryption across all accounts. Enable automatic rotation. Grant all accounts full `kms:*` permissions on the key.
- **C.** Use AWS-managed keys (`aws/s3`, `aws/kinesis`) in each account. These are simpler — no key management needed. Use S3 bucket policies and Kinesis resource policies for cross-account access. AWS handles rotation automatically.
- **D.** Create KMS keys in a dedicated Security account. Use grants (not key policies) to provide time-limited access to each account's encryption operations. Use asymmetric RSA keys for S3 encryption. Enable automatic rotation for asymmetric keys.

## Answers

### A. Per-account CMK + cross-account key policy grants + automatic rotation + CloudTrail — ✅ Correct

This architecture provides proper key isolation with controlled cross-account access:

- **Per-account customer-managed symmetric CMKs**:
  - Each account has its own CMK. The key administrator is different from the key user — separation of duties. The Data Ingestion account's key encrypts ingested data; the Analytics account's key encrypts processed results.
  - **Symmetric keys (AES-256-GCM)**: Used for `Encrypt`, `Decrypt`, and `GenerateDataKey` operations. Symmetric keys support automatic rotation and envelope encryption — the standard for data-at-rest encryption.
  - **Key isolation**: If the Analytics account's key is compromised, only Analytics data is affected — Ingestion and Archive data remain protected. Single-key architectures have no blast radius containment.

- **Cross-account key policy grants**:
  - The Data Ingestion key policy includes:
    ```json
    {
      "Sid": "AllowAnalyticsDecrypt",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::ANALYTICS_ACCOUNT:role/DataProcessingRole"},
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "*"
    }
    ```
  - **Least privilege**: Only `Decrypt` and `DescribeKey` — the Analytics account cannot encrypt new data with the Ingestion key, cannot disable the key, and cannot modify the key policy. Minimum permissions for the task.
  - **Two-part authorization**: Cross-account KMS access requires BOTH: (1) key policy in the key's account allows the caller, AND (2) IAM policy in the caller's account allows the KMS action. Both must agree — neither alone is sufficient.

- **Envelope encryption** (how it works under the hood):
  - When S3 encrypts an object with SSE-KMS, it calls `GenerateDataKey` → KMS returns a plaintext data key + an encrypted copy of the data key. S3 encrypts the object with the plaintext data key (AES-256), then discards the plaintext key and stores the encrypted key alongside the object.
  - On decryption, S3 sends the encrypted data key to KMS → KMS decrypts it using the CMK → S3 uses the plaintext data key to decrypt the object. KMS never sees the data itself — only the data key.
  - **Why envelope encryption**: Encrypting large objects (GB+) directly with KMS would require sending all data to KMS (4 KB max per API call). Envelope encryption encrypts locally with a data key and only sends the small data key to KMS.

- **Automatic annual rotation**:
  - Symmetric CMKs support automatic rotation (annual). AWS creates new key material annually but retains all previous key material — old key material is used to decrypt data encrypted with older versions. No re-encryption needed.
  - The key ID and ARN remain the same after rotation — applications don't need code changes.

- **Immediate key disablement**:
  - `kms:DisableKey` immediately prevents ALL cryptographic operations (encrypt, decrypt, generate data key) using the key. This is the "emergency brake" for a compromised key.
  - Disabled keys can be re-enabled later if the compromise turns out to be a false alarm. Alternatively, schedule key deletion (7-30 day waiting period) for permanent destruction.

- **CloudTrail audit**:
  - Every KMS API call (`Encrypt`, `Decrypt`, `GenerateDataKey`, `DisableKey`, etc.) is logged in CloudTrail with: calling principal ARN, key ARN, encryption context, timestamp, and source IP. This provides the required audit trail of "who used which key, for which resource, when."

### B. Single multi-Region key for all accounts — ❌ Incorrect

- **Multi-Region keys**: KMS multi-Region keys are replicas of the same key material in different Regions — they're for cross-Region use (e.g., encrypting in us-east-1, decrypting in eu-west-1). They are NOT for cross-account use. Multi-Region key replicas exist within the same account, not across accounts.
- **Single key for all accounts with `kms:*` permissions**: Violates least privilege. If any account is compromised, the attacker has full key control — including `kms:DisableKey`, `kms:ScheduleKeyDeletion`, and `kms:PutKeyPolicy`. This could lead to total data loss (attacker deletes the key) or mass data exfiltration (attacker decrypts everything).
- **No blast radius containment**: One compromised key = all data across all accounts is compromised.

### C. AWS-managed keys — ❌ Incorrect

- **AWS-managed keys** (e.g., `aws/s3`): The key policy is managed by AWS and CANNOT be modified. You cannot add cross-account access to an AWS-managed key's policy. Cross-account decryption is impossible with AWS-managed keys.
- **Cannot audit key usage at granular level**: AWS-managed key usage is logged in CloudTrail, but you can't add custom conditions, grants, or key policies to control who uses them and how.
- **Cannot disable an AWS-managed key**: If the key is compromised (unlikely since AWS manages it, but the requirement stands), you cannot disable it. Only customer-managed keys support `DisableKey`.
- **Automatic rotation**: AWS-managed keys rotate automatically every year, which is good — but the cross-account and control limitations make them unsuitable for PHI compliance.

### D. Centralized Security account + grants + asymmetric keys — ❌ Incorrect

- **KMS grants**: Grants provide fine-grained, time-limited access to KMS keys. While grants are valid for cross-account access, they're more complex to manage than key policies for static, long-term permissions. Grants are better for temporary access (e.g., a Lambda function accessing a key during execution) — not for permanent cross-account data access patterns.
- **Asymmetric RSA keys for S3 encryption**: S3 SSE-KMS only supports SYMMETRIC KMS keys. You cannot use asymmetric keys (RSA, ECC) for SSE-KMS encryption. Asymmetric keys are for digital signatures, API authentication, and client-side encryption schemes — not for S3 server-side encryption.
- **Automatic rotation for asymmetric keys**: KMS does NOT support automatic rotation for asymmetric keys. Asymmetric keys must be rotated manually (create new key, update applications, retire old key). The option's claim that automatic rotation works for asymmetric keys is incorrect.

## Recommendations

- **Per-account CMKs** with cross-account key policy grants is the standard architecture for multi-account encryption — provides isolation, control, and auditability.
- **Cross-account KMS requires both key policy AND IAM policy** — forgetting either one causes "AccessDeniedException."
- **Always use symmetric CMKs** for data-at-rest encryption (S3, EBS, RDS, Kinesis). Asymmetric keys are for signing and client-side operations.
- **Encryption context** is a set of key-value pairs that KMS includes in the authentication tag. Use it to bind the data key to a specific resource (e.g., `{"bucket": "my-bucket", "key": "patient-123.dcm"}`). This prevents data key reuse across different resources.
- **Key policy + SCP + IAM = defense in depth**: SCP can deny `kms:ScheduleKeyDeletion` organization-wide to prevent accidental key destruction.
- **Monitor with CloudTrail + EventBridge**: Create EventBridge rules for sensitive KMS events (`DisableKey`, `ScheduleKeyDeletion`, `PutKeyPolicy`) with SNS alerts to the security team.

## Relevant Links

- [KMS Key Policies](https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html)
- [Cross-Account KMS Access](https://docs.aws.amazon.com/kms/latest/developerguide/key-policy-modifying-external-accounts.html)
- [Envelope Encryption](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#enveloping)
- [KMS Key Rotation](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html)
- [KMS Grants](https://docs.aws.amazon.com/kms/latest/developerguide/grants.html)
- [SSE-KMS Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html)
