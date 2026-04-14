# Q01: Centralized Backup Strategy with AWS Backup

## Question

A large enterprise operates **150 AWS accounts** organized under AWS Organizations. Each account runs a mix of Amazon EC2 instances, Amazon RDS databases, Amazon EFS file systems, and Amazon DynamoDB tables. The company's compliance team requires:

1. **All resources** across all accounts must be backed up daily with 30-day retention.
2. Backup copies must be stored in a **separate Region** from the source.
3. A **central team** must manage and monitor backup policies — individual account owners should NOT be able to modify or delete backups.
4. Backup compliance must be auditable.

Which solution meets ALL requirements?

## Options

- **A.** Deploy an AWS Backup vault in each account with a backup plan configured by AWS CloudFormation StackSets. Use IAM policies to prevent account owners from modifying vaults. Enable cross-Region copy in each backup plan.
- **B.** Use **AWS Backup** with **AWS Organizations integration**. Create backup policies in the management account and apply them across OUs. Enable cross-Region copy rules. Configure vault lock in compliance mode. Use AWS Backup Audit Manager for compliance reporting.
- **C.** Write a custom AWS Lambda function triggered daily by Amazon EventBridge to create snapshots of all resources in each account. Copy snapshots to a secondary Region using another Lambda function. Store audit logs in Amazon S3.
- **D.** Use AWS Backup in each account individually. Have each account owner configure their own backup plans following a documented standard. Use AWS Config rules to check compliance.

## Answers

### B. AWS Backup with Organizations — ✅ Correct

AWS Backup with Organizations integration provides **centralized, delegated backup management**:
- **Backup policies** can be created at the Organization or OU level and automatically applied to all member accounts.
- **Cross-Region copy** rules replicate backups to a secondary Region.
- **Vault Lock (compliance mode)** prevents anyone — including root — from deleting backups before the retention period expires, meeting the immutability requirement.
- **AWS Backup Audit Manager** generates compliance reports showing which resources are backed up according to policy.
This is the purpose-built, fully managed solution for multi-account centralized backup.

### A. CloudFormation StackSets — ❌ Incorrect

StackSets can deploy backup configurations across accounts, but this approach has gaps:
1. IAM policies alone **cannot prevent a root user** in a member account from modifying backup vaults. Only Vault Lock provides true immutability.
2. StackSets require ongoing maintenance — new accounts must be manually added or detected.
3. No built-in compliance auditing — you'd need additional tooling.
AWS Backup's native Organizations integration is simpler and more secure.

### C. Custom Lambda solution — ❌ Incorrect

Building custom backup logic for 150 accounts across multiple services (EC2, RDS, EFS, DynamoDB) is a massive engineering effort. It's error-prone, requires maintenance, doesn't support vault lock, and lacks built-in compliance reporting. This re-invents what AWS Backup provides natively.

### D. Per-account self-managed — ❌ Incorrect

Letting account owners manage their own backups violates requirements 3 and 4. Individual owners could misconfigure or skip backups. AWS Config rules can detect non-compliance but not prevent it. The central team has no enforcement mechanism.

## Recommendations

- **AWS Backup + Organizations** is the go-to for multi-account backup governance. Know these features:
  - **Backup policies:** Define what/when/how-long at the Org/OU level
  - **Cross-Region copy:** Built into backup plans
  - **Vault Lock:** Compliance mode = WORM (write-once-read-many), prevents deletion
  - **Audit Manager:** Compliance reporting framework
- **Vault Lock modes:**
  - **Governance mode:** Can be removed by users with specific IAM permissions
  - **Compliance mode:** Cannot be removed by anyone, including root — truly immutable
- Supported services: EC2 (EBS), RDS, Aurora, DynamoDB, EFS, FSx, Storage Gateway, S3, and more.
- The exam strongly favors **managed services over custom solutions** for operational requirements.

## Relevant Links

- [AWS Backup with Organizations](https://docs.aws.amazon.com/aws-backup/latest/devguide/manage-cross-account.html)
- [AWS Backup Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)
- [AWS Backup Audit Manager](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-audit-manager.html)
- [AWS Backup Supported Resources](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html#supported-resources)
