# Q01: KMS vs. CloudHSM for Encryption Key Management

## Question

A financial services company must meet strict compliance requirements (FIPS 140-2 Level 3) for encryption key management. The company encrypts data at rest in Amazon S3, Amazon EBS, and Amazon RDS. They also need to: generate and store encryption keys in hardware security modules they fully control, support custom key stores for KMS-managed keys, and maintain full audit logs of all key usage.

Which approach should the solutions architect recommend?

## Options

- **A.** Use AWS KMS with default AWS-managed keys for all services. KMS is FIPS 140-2 Level 2 validated and provides built-in auditing via CloudTrail.
- **B.** Use AWS CloudHSM to create a cluster of HSMs. Configure a KMS custom key store backed by CloudHSM. Create KMS customer-managed keys (CMKs) in the custom key store. Use these keys for S3, EBS, and RDS encryption.
- **C.** Use AWS CloudHSM standalone (without KMS integration). Generate keys in CloudHSM and use the CloudHSM client SDK to encrypt data before storing in S3. For EBS and RDS, use CloudHSM-generated keys via application-level encryption.
- **D.** Use AWS KMS with imported key material. Generate keys in an on-premises HSM, import into KMS, and use for all service encryption. This provides full key control while using KMS-native integrations.

## Answers

### B. CloudHSM custom key store + KMS — ✅ Correct

This approach combines CloudHSM's compliance-grade key storage with KMS's service integration:
- **FIPS 140-2 Level 3**: CloudHSM is validated at Level 3 (vs. KMS at Level 2). Keys are generated and stored in customer-dedicated, tamper-evident HSMs.
- **Custom key store**: KMS can use CloudHSM as its key storage backend. Keys never leave the HSM — all cryptographic operations happen within the HSM.
- **KMS integration**: Because the keys are still KMS CMKs (backed by CloudHSM), they work natively with S3 SSE-KMS, EBS encryption, and RDS encryption. No application code changes needed.
- **Full control**: The company manages the CloudHSM cluster — they control physical HSM access and can delete keys definitively.
- **Audit trail**: CloudTrail logs all KMS API calls (encrypt, decrypt, key usage); CloudHSM logs provide additional HSM-level audit data.

### A. KMS default keys — ❌ Incorrect

KMS with AWS-managed keys is FIPS 140-2 **Level 2**, not Level 3. AWS-managed keys also don't give the company full key control (they can't rotate or delete them on their own schedule). For compliance requiring Level 3, CloudHSM is necessary.

### C. CloudHSM standalone — ❌ Incorrect

CloudHSM without KMS integration requires application-level encryption for every service. S3 would need client-side encryption (encrypt data before upload). EBS and RDS don't support direct CloudHSM key integration for server-side encryption — you'd need to encrypt at the application layer, which is complex and may not be transparent to all services. Using a KMS custom key store provides the same HSM security with native service integration.

### D. KMS with imported key material — ❌ Incorrect

Imported key material in KMS is stored in KMS's multi-tenant HSMs (FIPS 140-2 Level 2), not in dedicated HSMs. This doesn't meet the Level 3 requirement. Also, imported keys have operational limitations: they can't be auto-rotated by KMS, and if the key material is deleted, the data is unrecoverable (no KMS backup).

## Recommendations

- **KMS default** (Level 2): Sufficient for most workloads. Use AWS-managed or customer-managed keys.
- **KMS custom key store (CloudHSM-backed)** (Level 3): For compliance requirements mandating Level 3 and native AWS service integration.
- **CloudHSM standalone** (Level 3): For custom cryptographic operations (code signing, TLS offload, Oracle TDE) where KMS integration is not needed.
- CloudHSM operates as a **cluster** (minimum 2 HSMs recommended for HA across AZs).
- **CloudHSM pricing**: ~$1.60/hr per HSM (~$1,170/month per HSM). Minimum 2 for production = ~$2,340/month.
- KMS custom key store operations have higher latency than standard KMS due to HSM round-trips.

## Relevant Links

- [AWS KMS Custom Key Stores](https://docs.aws.amazon.com/kms/latest/developerguide/custom-key-store-overview.html)
- [AWS CloudHSM](https://docs.aws.amazon.com/cloudhsm/latest/userguide/introduction.html)
- [KMS FIPS Validation](https://docs.aws.amazon.com/kms/latest/developerguide/compliance.html)
- [CloudHSM Use Cases](https://docs.aws.amazon.com/cloudhsm/latest/userguide/use-cases.html)
