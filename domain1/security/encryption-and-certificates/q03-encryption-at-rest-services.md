# Q03: Encryption at Rest — Service-Specific Behavior

## Question

A company's security policy requires all data at rest to be encrypted using customer-managed KMS keys (CMKs). The solutions architect needs to implement this across multiple AWS services. For which services does encryption at rest require **explicit configuration** rather than being enabled by default?

**(Select THREE.)**

## Options

- **A.** Amazon S3 — requires enabling default bucket encryption with SSE-KMS and specifying the CMK.
- **B.** Amazon EBS — requires enabling encryption at the account level or per-volume and specifying the CMK.
- **C.** Amazon DynamoDB — all tables are encrypted at rest by default using AWS-owned keys. Switching to a CMK requires table-level configuration.
- **D.** AWS CloudTrail logs — CloudTrail automatically encrypts log files in S3 with SSE-S3.
- **E.** Amazon RDS — requires enabling encryption at database creation time. Encryption cannot be enabled on an existing unencrypted database directly.
- **F.** Amazon SQS — messages are always encrypted at rest with no configuration needed.

## Answers

### A. Amazon S3 — ✅ Correct

S3 now encrypts all new objects by default using **SSE-S3** (S3-managed keys). However, to use a **customer-managed KMS key** (SSE-KMS), you must explicitly configure it:
- Set the default bucket encryption to SSE-KMS and specify the CMK ARN.
- Or include the `x-amz-server-side-encryption: aws:kms` header on each PutObject request.
Without explicit SSE-KMS configuration, objects are encrypted with AWS-managed SSE-S3 keys, which don't meet the "customer-managed CMK" requirement.

### B. Amazon EBS — ✅ Correct

EBS volumes are **not encrypted by default** unless you enable the account-level setting "Always encrypt new EBS volumes." To use a CMK:
- Enable EBS encryption by default in the account settings and specify the default CMK.
- Or select encryption and the CMK when creating each volume.
Existing unencrypted volumes cannot be directly encrypted — you must create a snapshot, copy it with encryption enabled, and create a new volume from the encrypted snapshot.

### E. Amazon RDS — ✅ Correct

RDS encryption must be enabled at **database creation time** — it cannot be added to an existing unencrypted DB instance. To use a CMK:
- Select "Enable encryption" and choose the CMK during database creation.
- To encrypt an existing unencrypted database: create a snapshot, copy the snapshot with encryption enabled (specifying the CMK), restore from the encrypted snapshot. This creates a new DB instance.

### C. Amazon DynamoDB — ❌ Incorrect (it IS encrypted by default)

DynamoDB encrypts all tables at rest by default using **AWS-owned keys** at no extra cost. You **can** change this to a customer-managed CMK or AWS-managed key, but encryption itself is automatic. The question asks which services require "explicit configuration" — DynamoDB is encrypted by default (though not with a CMK by default).

### D. CloudTrail — ❌ Incorrect (it IS encrypted by default)

CloudTrail log files delivered to S3 are encrypted by default using **SSE-S3**. You can optionally configure SSE-KMS encryption with a CMK, but encryption itself is automatic. CloudTrail also encrypts logs in transit.

### F. Amazon SQS — ❌ Incorrect (server-side encryption is optional)

SQS does support server-side encryption (SSE) with KMS keys, but it is **not enabled by default** for all queues (note: new queues created via the console may have SSE enabled by default, but API-created queues may not). However, regardless of SSE configuration, SQS messages are always encrypted at the infrastructure level by AWS. The distinction between AWS-managed infrastructure encryption and application-level SSE-KMS is important for the exam.

## Recommendations

- **S3**: SSE-S3 is now default. For CMK requirement, configure SSE-KMS at the bucket level.
- **EBS**: Enable "Encrypt new EBS volumes" at the **account level** in each Region — this ensures all new volumes use the specified CMK.
- **RDS**: Plan encryption before creating databases — encryption cannot be added retroactively without downtime.
- **DynamoDB**: Default AWS-owned encryption is transparent and free. CMK encryption adds cost per request.
- Use **AWS Config** rule `encrypted-volumes` and `rds-storage-encrypted` to monitor encryption compliance.
- Consider **organizational SCP** to deny `ec2:RunInstances` when encryption is not specified — prevents creation of unencrypted resources.

## Relevant Links

- [S3 Default Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-bucket-encryption.html)
- [EBS Encryption](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html)
- [RDS Encryption](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)
- [DynamoDB Encryption](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EncryptionAtRest.html)
- [KMS Concepts](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html)
