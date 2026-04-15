# Q02: GuardDuty, Security Hub, Inspector, and Macie — Integrated Threat Detection

## Question

A company with 80 AWS accounts has experienced three security incidents in the past quarter:
1. An EC2 instance was compromised and used for cryptocurrency mining for 5 days before detection
2. An S3 bucket containing PII was made public via a misconfigured bucket policy — no one noticed for 2 weeks
3. A critical CVE in Apache Log4j was present on 15 EC2 instances for 3 months because patching was manual

The CISO requires a unified security posture that:
- Detects anomalous behavior (crypto mining, unusual API calls) across all accounts within hours
- Continuously discovers and alerts on PII stored in S3 buckets
- Automatically scans EC2 instances and container images for CVEs without installing agents
- Aggregates all findings in a single dashboard with automated compliance checks against CIS AWS Foundations Benchmark
- Automatically remediates critical findings (e.g., blocking a compromised instance's internet access)

Which combination of AWS security services achieves all requirements?

## Options

- **A.** Enable GuardDuty with delegated administrator in a Security account — detects crypto mining, anomalous API calls, and compromised instances. Enable Amazon Macie with sensitive data discovery jobs — scans S3 for PII. Enable Amazon Inspector v2 for agentless scanning — assesses EC2 and ECR images for CVEs. Aggregate all findings in AWS Security Hub with CIS AWS Foundations Benchmark enabled. Use Security Hub custom actions + EventBridge + Lambda/SSM Automation for auto-remediation (e.g., modify security group to block egress).
- **B.** Use AWS Config Rules for all detections: `ec2-instance-no-public-ip`, `s3-bucket-public-read-prohibited`, custom rules for CVE scanning. Aggregate Config compliance in a custom CloudWatch dashboard. Use Config auto-remediation with SSM Automation.
- **C.** Deploy a third-party SIEM (Splunk/Elastic) ingesting CloudTrail logs. Write custom detection rules for crypto mining patterns. Use S3 Access Analyzer for PII detection. Use Systems Manager Patch Manager for CVE scanning. Build custom dashboards in QuickSight.
- **D.** Enable GuardDuty for threat detection. Use S3 Access Analyzer for public bucket detection. Use Systems Manager for CVE scanning. Aggregate findings in CloudWatch Logs with metric filters for alerting. Manual remediation via runbooks.

## Answers

### A. GuardDuty + Macie + Inspector v2 + Security Hub + automated remediation — ✅ Correct

This is the AWS-native security stack, purpose-built for each detection domain:

- **Amazon GuardDuty — threat detection**:
  - GuardDuty analyzes CloudTrail management/data events, VPC Flow Logs, DNS logs, and EKS audit logs using ML and threat intelligence feeds.
  - **Crypto mining detection**: `CryptoCurrency:EC2/BitcoinTool.B!DNS` finding type detects DNS queries to known mining pool domains. `UnauthorizedAccess:EC2/MaliciousIPCaller.Custom` detects communication with C2 servers.
  - **Anomalous API calls**: `Recon:IAMUser/MaliciousIPCaller`, `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` detect compromised credentials or unusual access patterns.
  - **Delegated administrator**: In an Organization, a Security account acts as the delegated administrator — it enables GuardDuty in all 80 accounts and receives all findings centrally. No per-account setup needed.
  - **Detection timeline**: GuardDuty analyzes logs continuously and generates findings within minutes to hours — the crypto mining incident (5 days undetected) would be caught within hours.

- **Amazon Macie — PII/sensitive data discovery**:
  - Macie uses ML and pattern matching to identify sensitive data in S3 buckets: PII (SSN, credit card numbers, passport numbers), PHI, financial data.
  - **Automated sensitive data discovery**: Macie continuously samples and analyzes S3 objects, generating findings for buckets containing PII. This catches the "public bucket with PII" scenario — Macie identifies both the PII presence AND the public access configuration.
  - **S3 bucket-level assessment**: Macie evaluates bucket policies, ACLs, and encryption settings — flagging buckets that are public, unencrypted, or shared externally.
  - Macie is the purpose-built tool for "is there PII in this bucket?" — S3 Access Analyzer only tells you "is this bucket publicly accessible?" without analyzing the data content.

- **Amazon Inspector v2 — vulnerability assessment**:
  - Inspector v2 provides **agentless scanning** for EC2 instances (using EBS snapshots — no SSM agent needed) and automatic scanning of ECR container images when pushed.
  - **CVE detection**: Inspector checks installed packages against the National Vulnerability Database (NVD) and vendor advisories. A critical CVE like Log4j (CVE-2021-44228) would generate a Critical severity finding immediately after scan.
  - **Continuous scanning**: Inspector v2 automatically re-scans when new CVEs are published or when instances change. No scheduled scan jobs needed — it's event-driven.
  - **No agent installation**: The original Inspector (v1) required the SSM agent. Inspector v2 supports agentless mode using EBS volume analysis — meeting the "without installing agents" requirement.

- **AWS Security Hub — aggregation and compliance**:
  - Security Hub ingests findings from GuardDuty, Macie, Inspector, IAM Access Analyzer, Firewall Manager, and Config — providing a single-pane-of-glass dashboard.
  - **CIS AWS Foundations Benchmark**: Security Hub includes built-in compliance standards including CIS AWS Foundations Benchmark v1.4, AWS Foundational Security Best Practices, PCI DSS. It automatically evaluates your accounts against these benchmarks.
  - **Cross-account aggregation**: With Organization integration, the Security account receives findings from all 80 accounts in one dashboard.

- **Automated remediation**:
  - Security Hub custom actions trigger EventBridge events when a finding is selected (or automatically via EventBridge rules matching finding patterns).
  - Example: GuardDuty `CryptoCurrency:EC2/BitcoinTool.B!DNS` finding → EventBridge rule → Lambda function → modify the instance's security group to deny all egress traffic + create a snapshot for forensics + notify the security team via SNS.
  - SSM Automation documents can also be triggered for more complex remediation (isolate instance, revoke compromised IAM credentials, enable S3 block public access).

### B. AWS Config Rules for everything — ❌ Incorrect

- **Config Rules are compliance checks, not threat detection**: `ec2-instance-no-public-ip` checks configuration compliance — it doesn't detect crypto mining, anomalous API calls, or compromised credentials. Config evaluates "is the resource configured correctly?" not "is the resource behaving maliciously?"
- **No CVE scanning capability**: Config doesn't scan OS packages or container images for vulnerabilities. A custom Config rule could check if instances have the SSM agent (pre-requisite for patching), but it can't identify which CVEs are present.
- **No PII discovery**: `s3-bucket-public-read-prohibited` checks if a bucket is public — it doesn't scan the bucket's CONTENTS for PII. A bucket can be private and still contain improperly stored PII.
- Config is an important compliance tool but doesn't replace purpose-built security services (GuardDuty, Macie, Inspector).

### C. Third-party SIEM + custom rules — ❌ Incorrect

- **Third-party SIEM (Splunk/Elastic)**: Capable but expensive and operationally complex. Requires: deploying and managing SIEM infrastructure, writing and tuning detection rules, ingesting high-volume CloudTrail/VPC Flow Log data (costs $2-5/GB ingested), and maintaining expertise. GuardDuty provides equivalent detection out-of-the-box with zero infrastructure.
- **S3 Access Analyzer for PII**: IAM Access Analyzer (which includes S3 access analysis) identifies publicly accessible or cross-account shared resources — it does NOT scan data contents for PII. That's Macie's job.
- **Systems Manager for CVE scanning**: SSM Patch Manager checks patch compliance (which patches are installed vs. which are missing) — it doesn't scan for CVEs by CVE ID or prioritize by CVSS score like Inspector does. Patch Manager is for remediation (applying patches), not for vulnerability assessment.
- **Custom dashboards in QuickSight**: Requires building custom data pipelines from CloudTrail → Athena → QuickSight. Security Hub provides this dashboard natively with built-in compliance standards.

### D. GuardDuty + Access Analyzer + SSM + CloudWatch — ❌ Incorrect

- **S3 Access Analyzer for public bucket detection**: Correct for identifying publicly accessible S3 buckets, but doesn't discover PII content. The requirement is "discover and alert on PII stored in S3" — not just public access detection.
- **SSM for CVE scanning**: As above, SSM Patch Manager isn't a CVE scanner. It also requires the SSM agent to be installed on instances — violating the "without installing agents" requirement. Inspector v2's agentless mode is needed.
- **CloudWatch Logs with metric filters**: This approach requires writing custom metric filters for each threat pattern — fragile, hard to maintain, and prone to false positives/negatives. Security Hub aggregates findings from purpose-built services in a structured format.
- **Manual remediation via runbooks**: Manual steps introduce delay — the crypto mining instance ran for 5 days because humans didn't act. Automated remediation (Lambda + SSM triggered by EventBridge) reduces response time to minutes.

## Recommendations

- **Enable all four services** (GuardDuty, Macie, Inspector, Security Hub) as a baseline security stack for any multi-account Organization.
- **GuardDuty costs** are based on log volume analyzed — typical cost is $1-4/account/month for moderate workloads. Macie costs $1/bucket assessed + $1/GB scanned. Inspector pricing is per assessment. All three are far cheaper than the cost of a single security incident.
- **Security Hub finding suppression**: Use suppression rules to filter out known false positives — this keeps the dashboard actionable.
- **Start with automated alerting, progress to automated remediation**: Begin by sending findings to Slack/email. Once you trust the finding accuracy, add auto-remediation for high-confidence finding types (crypto mining, public S3 buckets).
- **Macie one-time vs. automated discovery**: Use automated discovery for continuous monitoring. One-time discovery jobs are for initial assessment of existing buckets.
- **Inspector CVE findings include CVSS scores** and fix recommendations — prioritize Critical and High severity findings for immediate patching.

## Relevant Links

- [Amazon GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)
- [Amazon Macie](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html)
- [Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html)
- [AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html)
- [Security Hub Automated Remediation](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cloudwatch-events.html)
- [CIS AWS Foundations Benchmark](https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html)
