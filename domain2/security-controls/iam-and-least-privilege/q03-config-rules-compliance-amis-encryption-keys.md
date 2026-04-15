# Q03: AWS Config Rules for Compliance — Approved AMIs, Encryption Policies, and Credential Auditing

## Question

A financial services company has 200 AWS accounts in an Organization. The compliance team has identified three recurring violations:

1. **Unapproved AMIs in production**: Developers launch EC2 instances from community AMIs that haven't passed security scanning. Production accounts must only allow instances from 15 approved AMIs (maintained by the platform team). Development accounts can use any AMI for experimentation.
2. **Unencrypted EBS volumes**: Some teams create EBS volumes without encryption. All new EBS volumes must use a KMS customer-managed key. Existing unencrypted volumes must be detected and reported.
3. **Long-lived IAM access keys**: Several service accounts have IAM access keys that haven't been rotated in over 180 days. Access keys older than 90 days must be flagged, and keys older than 180 days must be automatically disabled.

The compliance team wants: automated detection (not manual audits), centralized visibility across all 200 accounts, and automatic remediation where possible. All compliance state must be queryable for audit reports.

Which solution addresses all three requirements?

## Options

- **A.** Deploy AWS Config across all accounts using an Organization-wide aggregator. Create three Config Rules: (1) `approved-amis-by-id` — evaluates whether running EC2 instances use approved AMI IDs (applied only to production OUs via SCP-enforced tagging), (2) `encrypted-volumes` — checks that all EBS volumes are encrypted with KMS, (3) `access-keys-rotated` with `maxAccessKeyAge: 90` — flags keys older than 90 days. For remediation: attach SSM Automation documents — the access key rule triggers an SSM runbook that disables keys older than 180 days via `iam:UpdateAccessKey`. Use a Config Aggregator in the Security account for centralized dashboard and compliance reporting.
- **B.** Use AWS Security Hub with CIS Benchmark checks enabled. Security Hub checks for unencrypted volumes and long-lived access keys. For AMI enforcement, use an SCP that denies `ec2:RunInstances` unless the AMI is in the approved list. Security Hub aggregation provides the centralized dashboard.
- **C.** Deploy a Lambda function in each account that runs daily, checking EC2 instances for approved AMIs, EBS volumes for encryption, and IAM access keys for age. Lambda writes findings to a central DynamoDB table. A CloudWatch dashboard displays compliance status.
- **D.** Use AWS Trusted Advisor checks for security best practices. Trusted Advisor already checks for exposed IAM keys, unencrypted EBS volumes, and security group configurations. Enable Trusted Advisor Organization view for centralized reporting.

## Answers

### A. AWS Config Rules + SSM remediation + Config Aggregator — ✅ Correct

AWS Config is the purpose-built service for continuous compliance evaluation and automated remediation:

**Rule 1: `approved-amis-by-id`**

- **AWS Managed Rule**: `approved-amis-by-id` is a pre-built Config rule. You provide a comma-separated list of approved AMI IDs as a parameter. Config evaluates every running EC2 instance and marks it `NON_COMPLIANT` if its AMI is not in the approved list.
- **Production-only enforcement**: Deploy this rule only to production OU accounts. Methods:
  - Use AWS Config's deployment via CloudFormation StackSets targeted at the Production OU — the rule only exists in production accounts.
  - Alternatively, use the rule in all accounts but combine with SCP: attach an SCP to the Production OU that denies `ec2:RunInstances` unless `ec2:ImageId` matches the approved list. Config detects existing violations; SCP prevents new ones.
- **Development flexibility**: Development accounts don't have this Config rule deployed (or have it in monitor-only mode without remediation), so developers can experiment freely.
- **Configuration snapshots**: Config records every resource configuration change. You can query: "When did instance i-1234 launch? Which AMI was it using? Who launched it?" — full resource history for audit.

**Rule 2: `encrypted-volumes`**

- **AWS Managed Rule**: `encrypted-volumes` checks that all attached EBS volumes have encryption enabled. Detects both unencrypted volumes and volumes encrypted with AWS-managed keys (if you require CMK specifically, use the `kms:KeyId` condition).
- **Proactive prevention**: Enable EBS encryption by default (`ec2:EnableEbsEncryptionByDefault`) in every account — all new volumes are encrypted automatically. Config catches any existing unencrypted volumes that predate this setting.
- **Remediation**: Creating an encrypted copy of an unencrypted volume requires: snapshot → copy with encryption → create new volume → swap. This is complex and disruptive for running instances, so Config reports it for manual remediation rather than auto-remediation.

**Rule 3: `access-keys-rotated`**

- **AWS Managed Rule**: `access-keys-rotated` accepts a `maxAccessKeyAge` parameter (in days). Every IAM user with access keys older than the specified age is marked `NON_COMPLIANT`.
- **Tiered response with SSM remediation**:
  - Config detects keys > 90 days → marks `NON_COMPLIANT` → sends finding to Security Hub → notification to the team
  - Keys > 180 days → Config triggers an SSM Automation document (remediation action) that calls `iam:UpdateAccessKey` with `Status=Inactive` to disable the key
  - The SSM Automation document runs with a remediation execution role that has permission to disable IAM access keys (not delete — disabling is reversible)
- **Audit trail**: Every key status change is logged in CloudTrail. Config records the compliance timeline — you can prove when a key became non-compliant and when it was remediated.

**Centralized visibility: Config Aggregator**

- **Organization-wide aggregator**: Deployed in the Security account (delegated administrator). Automatically collects Config data from all 200 accounts and all Regions.
- **Aggregator dashboard**: View compliance status across all accounts: number of non-compliant resources per rule, trend over time, drill down to specific accounts/resources.
- **Compliance reports**: Config supports conformance packs (collections of rules mapped to compliance frameworks). Export compliance data to S3 for audit reports.
- **Integration with Security Hub**: Config findings automatically flow to Security Hub, providing a single pane of glass alongside GuardDuty, Inspector, and Macie findings.

### B. Security Hub + SCP — ❌ Incorrect

- **Security Hub CIS checks**: Security Hub does check for some of these (IAM access key rotation, unencrypted EBS volumes) via CIS AWS Foundations Benchmark controls. However:
  - Security Hub evaluates compliance checks **periodically** (every 12-24 hours for most controls), not continuously on every resource change like Config does.
  - Security Hub's checks are pre-defined — you can't customize parameters like specific AMI IDs allowed. There is no CIS Benchmark check for "approved AMIs."
  - Security Hub is an **aggregator and dashboard** — it consumes findings from Config, GuardDuty, Inspector. It's not the primary evaluation engine.

- **SCP for AMI enforcement**: An SCP with `Deny ec2:RunInstances` unless `ec2:ImageId` matches the approved list DOES prevent launches of unapproved AMIs. This is a valid **preventive** control. However:
  - SCPs don't detect existing non-compliant instances (launched before the SCP was applied). Config is needed for **detective** control — identifying existing violations.
  - SCPs alone don't provide compliance reporting or resource configuration history. Config provides the queryable audit trail.
  - The best practice is SCP (prevent) + Config (detect + report) together. SCP alone is incomplete.

- **No automated remediation in Security Hub alone**: Security Hub supports custom actions (manual) and automated actions (via EventBridge). But the evaluation engine is Config — Security Hub displays Config findings. Remediation is configured on Config rules, not Security Hub checks.

### C. Lambda per-account daily scan — ❌ Incorrect

- **Custom Lambda approach**: This works functionally but is a significant operational burden:
  - 200 accounts × custom Lambda = 200 Lambda functions to deploy, maintain, update, and monitor.
  - Daily execution misses violations created between scans (e.g., an unapproved AMI instance launched at 9 AM isn't detected until the next day's scan).
  - Must write custom code for AMI validation, volume encryption checks, and access key age evaluation — all of which are pre-built Config managed rules.
  - DynamoDB as the compliance data store requires custom query interfaces, access controls, and retention policies. Config provides this natively with S3 delivery channels and Athena integration.
  - No built-in remediation framework — must build custom notification and remediation logic.
- **When custom Lambda is appropriate**: When Config doesn't have a managed rule for a very specific, organization-unique compliance check. For standard checks (AMI approval, encryption, key rotation), use Config managed rules.

### D. Trusted Advisor — ❌ Incorrect

- **Trusted Advisor limitations**:
  - Trusted Advisor checks are fixed — you can't customize them (no "approved AMI list" check).
  - Trusted Advisor provides recommendations, not compliance evaluation with non-compliant/compliant states.
  - IAM key rotation check exists but with a fixed threshold (not configurable like Config's `maxAccessKeyAge`).
  - No automated remediation capability — Trusted Advisor is advisory only.
  - Trusted Advisor Organization view provides multi-account visibility but with limited detail compared to Config Aggregator.
  - Trusted Advisor is best used for cost optimization and service limit recommendations, not for continuous compliance enforcement.
  - Enterprise Support plan required for full Trusted Advisor API access — significant cost.

## Recommendations

- **Config is the primary compliance evaluation service** for AWS. It records resource configurations and evaluates them against rules continuously. Use it as the foundation for compliance automation.
- **Layer defenses**: SCP (prevent) + Config (detect/report) + remediation (fix). No single mechanism is sufficient alone.
- **Config conformance packs**: Use pre-built conformance packs (CIS, PCI-DSS, HIPAA) for compliance frameworks. These bundle relevant Config rules together.
- **Enable EBS encryption by default**: `aws ec2 enable-ebs-encryption-by-default` — prevents the problem at the source, rather than detecting it after creation.
- **Config costs**: Config charges per configuration item recorded ($0.003 each) and per Config rule evaluation ($0.001 per evaluation). With 200 accounts, optimize by using targeted rules rather than enabling every managed rule.
- **Aggregator vs. individual account review**: Always use a centralized aggregator in the Security/Compliance account. Never require auditors to log into individual accounts.

## Relevant Links

- [AWS Config Managed Rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
- [approved-amis-by-id Rule](https://docs.aws.amazon.com/config/latest/developerguide/approved-amis-by-id.html)
- [encrypted-volumes Rule](https://docs.aws.amazon.com/config/latest/developerguide/encrypted-volumes.html)
- [access-keys-rotated Rule](https://docs.aws.amazon.com/config/latest/developerguide/access-keys-rotated.html)
- [Config Remediation with SSM](https://docs.aws.amazon.com/config/latest/developerguide/remediation.html)
- [Multi-Account Config Aggregator](https://docs.aws.amazon.com/config/latest/developerguide/aggregate-data.html)
- [Config Conformance Packs](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html)
