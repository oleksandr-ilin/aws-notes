# Q01: Automated Cost-Effective Backup Strategy Across AZs and Regions

## Question

A media company stores video assets in S3, metadata in DynamoDB, and encoding job state in RDS MySQL. The CFO requires:
- All data must be backed up to a secondary Region for compliance (eu-central-1 backup from us-west-2)
- Backup retention: 7 days for operational recovery, 1 year for compliance
- Backup costs must be minimized — most archived backups will never be accessed
- Backup operations must be fully automated with no manual scripts
- A monthly compliance report showing backup coverage and any gaps

Which backup architecture should the solutions architect implement?

## Options

- **A.** Use AWS Backup with a backup plan that includes S3, DynamoDB, and RDS. Configure two backup rules: a daily rule with 7-day retention in us-west-2, and a daily rule that copies to eu-central-1 with 1-year retention using cold storage tier. Enable AWS Backup Audit Manager with the `BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN` framework for compliance reporting. Use AWS Backup Vault Lock in compliance mode to prevent backup deletion.
- **B.** Configure S3 Cross-Region Replication to eu-central-1 with S3 Glacier Deep Archive lifecycle rules. Create RDS automated backups with cross-Region replication. Export DynamoDB tables to S3 nightly via Data Pipeline. Build a custom Lambda function to generate monthly backup reports.
- **C.** Write a nightly Lambda function that creates DynamoDB on-demand backups, RDS snapshots, and S3 batch copy jobs to eu-central-1. Store the Lambda execution logs in CloudWatch Logs. Create a CloudWatch dashboard for backup monitoring.
- **D.** Use AWS Backup for RDS and DynamoDB only. For S3, rely on S3 versioning with lifecycle rules to transition non-current versions to Glacier. Store compliance reports manually in a shared drive.

## Answers

### A. AWS Backup unified plan + cross-Region copy + Audit Manager + Vault Lock — ✅ Correct

This is the optimal approach:
- **AWS Backup unified plan**: A single backup plan covers S3 (backup of S3 buckets), DynamoDB (on-demand backups), and RDS (snapshots). All managed centrally — no custom scripts.
- **Two backup rules**: Daily with 7-day retention for operational recovery (local Region), and daily cross-Region copy to eu-central-1 with 1-year retention. Cross-Region copy backup rules move recovery points automatically.
- **Cold storage tier**: AWS Backup supports transitioning backup copies to cold storage after a configurable period (e.g., 1 day) — minimizing long-term storage costs. Cold storage is significantly cheaper for the 358 days when the backup just sits for compliance.
- **Audit Manager**: The `BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN` control checks that all specified resources have at least one backup plan. Generates automated compliance reports — no custom Lambda or manual processes.
- **Vault Lock (compliance mode)**: Prevents anyone (including root) from deleting backups before retention expires — meets compliance immutability requirements.

### B. CRR + RDS cross-Region + Data Pipeline — ❌ Partially incorrect

- **S3 CRR** replicates objects but doesn't provide point-in-time restore — if an object is deleted or corrupted in the source, the deletion/corruption replicates too. CRR costs include storage + replication per-GB charges. Glacier Deep Archive lifecycle on replicated data is cost-effective but recovery is slow (12+ hours).
- **RDS cross-Region automated backups** work but are a separate mechanism from the DynamoDB backup — no unified management.
- **Data Pipeline** for DynamoDB export is a legacy approach — DynamoDB has native export to S3 and AWS Backup supports DynamoDB natively.
- **Custom Lambda for reports** adds code to maintain. No unified compliance framework.
- This approach works but fragments backup management across multiple tools.

### C. Custom Lambda scripts — ❌ Incorrect

- Custom Lambda functions for backups introduce code maintenance, error handling, retry logic, and monitoring overhead.
- If the Lambda function fails (timeout, throttle, IAM issue), backups are missed silently unless custom alerting is built.
- CloudWatch dashboard shows Lambda execution status, not backup compliance coverage.
- No cold storage tier for cost optimization — all snapshots stored at standard rates.
- No immutability protection — backups can be deleted by anyone with IAM permissions.

### D. AWS Backup partial + S3 versioning — ❌ Incorrect

- **S3 versioning** retains previous versions but is not a backup — it stays in the same bucket/Region. If the bucket is deleted or the account is compromised, versions are lost.
- S3 versioning with Glacier lifecycle for non-current versions adds complexity and doesn't provide cross-Region protection.
- **Not using AWS Backup for S3** misses the unified management benefit and cross-Region copy capability.
- **Manual compliance reports** violate the automation requirement and are error-prone for ongoing compliance.

## Recommendations

- **AWS Backup** now supports S3, DynamoDB, RDS, EFS, EBS, Aurora, FSx, Neptune, DocumentDB, and more — use it as the single pane of glass for all backup operations.
- **Cold storage transition**: Set transition to cold storage after 1 day for long-retention compliance copies — dramatic cost savings.
- **Vault Lock compliance mode** is irrevocable once the cooling-off period ends — test thoroughly before enabling.
- **Backup Audit Manager** provides pre-built frameworks and custom controls — schedule reports for automated delivery.
- For **DynamoDB**, AWS Backup creates on-demand backups and supports point-in-time recovery (PITR). Enable both: PITR for operational recovery (35-day window), Backup for long-term/cross-Region.
- Use **Organizations integration** to apply backup policies across all accounts from the management account.

## Relevant Links

- [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [AWS Backup Cross-Region Copy](https://docs.aws.amazon.com/aws-backup/latest/devguide/cross-region-backup.html)
- [Backup Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)
- [Backup Audit Manager](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-audit-manager.html)
- [AWS Backup for S3](https://docs.aws.amazon.com/aws-backup/latest/devguide/s3-backups.html)
