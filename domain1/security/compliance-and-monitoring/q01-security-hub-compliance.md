# Q01: Security Hub for Centralized Compliance

## Question

A company with 80 AWS accounts needs a centralized security posture view. The CISO requires: aggregated security findings from GuardDuty, Inspector, Firewall Manager, and IAM Access Analyzer across all accounts, automated compliance checks against CIS AWS Foundations Benchmark and PCI-DSS, prioritized findings with severity scores, and integration with the company's existing SIEM (Splunk) for centralized event management.

Which architecture should the solutions architect implement?

## Options

- **A.** Enable AWS Security Hub with a delegated administrator in the security account. Enable organization-wide auto-enrollment. Activate CIS and PCI-DSS security standards. Configure Security Hub to send findings to Amazon EventBridge, then to Amazon Kinesis Data Firehose for delivery to Splunk.
- **B.** Enable Amazon GuardDuty as the central security service. GuardDuty aggregates findings from Inspector, Config, and IAM Access Analyzer. Export GuardDuty findings to Splunk via S3.
- **C.** Deploy AWS Config aggregator in the security account. Enable all Config rules for CIS and PCI-DSS. Send Config evaluations to Splunk via CloudWatch Logs.
- **D.** Create custom Lambda functions that query each security service's API across all 80 accounts. Aggregate findings in DynamoDB and build a custom dashboard for the CISO.

## Answers

### A. Security Hub with delegated admin — ✅ Correct

AWS Security Hub is the centralized security posture management service:

- **Aggregated findings**: Ingests findings from GuardDuty, Inspector, Firewall Manager, IAM Access Analyzer, Config, and third-party tools using the AWS Security Finding Format (ASFF).
- **Delegated administrator**: A member account (security account) manages Security Hub for the entire Organization without using the management account.
- **Auto-enrollment**: New accounts added to the Organization are automatically enrolled in Security Hub.
- **Security standards**: Built-in compliance checks for CIS AWS Foundations, PCI-DSS, AWS Foundational Security Best Practices (FSBP), and NIST 800-53.
- **Severity scoring**: Findings are normalized into severity levels (CRITICAL, HIGH, MEDIUM, LOW, INFORMATIONAL).
- **EventBridge integration**: Security Hub findings generate EventBridge events that can route to Kinesis Data Firehose → Splunk for SIEM integration.
- **Cross-Region aggregation**: A single administrator account can aggregate findings from multiple Regions.

### B. GuardDuty as aggregator — ❌ Incorrect

GuardDuty is a threat detection service, not a security posture aggregator. It does not ingest findings from Inspector, Config, or IAM Access Analyzer. GuardDuty generates its own findings (based on CloudTrail, VPC Flow Logs, DNS) but doesn't provide compliance checks against CIS or PCI-DSS standards. Security Hub aggregates GuardDuty findings alongside other services.

### C. Config aggregator only — ❌ Incorrect

AWS Config evaluates resource configurations but doesn't aggregate findings from GuardDuty, Inspector, or IAM Access Analyzer. While Config rules can check CIS and PCI-DSS controls, the compliance evaluation is narrower than Security Hub's integrated assessment. Config is one data source that feeds into Security Hub.

### D. Custom Lambda aggregation — ❌ Incorrect

Building a custom aggregation solution requires managing: Lambda functions across 80 accounts, API rate limiting, finding normalization, severity scoring, and SIEM integration code. Security Hub provides all of this as a managed service with native integrations. The custom approach is more expensive, harder to maintain, and slower to implement.

## Recommendations

- **Security Hub** is the central pane of glass for security across an Organization. Enable it as one of the first steps in account governance.
- Enable at minimum the **AWS Foundational Security Best Practices** (FSBP) standard — it has the broadest coverage of AWS-specific controls.
- Use **automated response and remediation** via EventBridge → Step Functions/Lambda to auto-remediate common findings (e.g., auto-enable S3 bucket encryption).
- **Cross-Region aggregation** allows consolidating findings from all Regions into a single Region for centralized review.
- Security Hub findings can also be exported to **Amazon S3** via EventBridge + Kinesis Data Firehose for long-term archival.
- Use **Security Hub custom actions** to create manual response workflows (e.g., "Investigate" button that creates a Jira ticket).

## Relevant Links

- [AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html)
- [Security Standards](https://docs.aws.amazon.com/securityhub/latest/userguide/security-standards.html)
- [Finding Aggregation](https://docs.aws.amazon.com/securityhub/latest/userguide/finding-aggregation.html)
- [EventBridge Integration](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cloudwatch-events.html)
- [Delegated Administrator](https://docs.aws.amazon.com/securityhub/latest/userguide/designate-orgs-admin-account.html)
