# Q02: GuardDuty for Threat Detection

## Question

A company has been experiencing unauthorized cryptocurrency mining on EC2 instances. The security team discovers that compromised IAM credentials are being used to launch instances in Regions the company doesn't normally use. They need a solution that:
- Detects anomalous API calls (e.g., `RunInstances` in unusual Regions)
- Identifies EC2 instances communicating with known cryptocurrency mining pools
- Detects compromised credentials based on unusual usage patterns
- Works across all 50 accounts in the Organization without per-account agent installation

Which service should the solutions architect implement?

## Options

- **A.** Enable Amazon GuardDuty with a delegated administrator account. Enable all detection types (CloudTrail, VPC Flow Logs, DNS logs, EKS, S3, RDS, Lambda). Configure organization-wide auto-enrollment.
- **B.** Deploy Amazon Inspector across all accounts to continuously scan for EC2 vulnerabilities and detect cryptocurrency mining activity.
- **C.** Enable AWS CloudTrail Insights to detect unusual API activity patterns. Create CloudWatch alarms for `RunInstances` events in unused Regions.
- **D.** Install Amazon CloudWatch agent on all EC2 instances. Create custom metrics for CPU utilization anomalies (cryptocurrency mining causes high CPU). Use CloudWatch anomaly detection alarms.

## Answers

### A. Amazon GuardDuty — ✅ Correct

GuardDuty is the AWS threat detection service specifically designed for these scenarios:

- **Anomalous API detection**: GuardDuty analyzes CloudTrail management and data events using ML models that learn normal patterns. It detects unusual API calls like `RunInstances` in Regions the company doesn't use, or API calls from unfamiliar IP addresses/principals.
- **Cryptocurrency detection**: GuardDuty maintains threat intelligence feeds that include known mining pool IPs and domains. Finding type `CryptoCurrency:EC2/BitcoinTool.B` is specifically for this.
- **Credential compromise**: GuardDuty detects: API calls from known malicious IPs, Tor exit nodes, impossible travel (API calls from geographically distant locations), and unusual principal behavior.
- **Agentless**: GuardDuty analyzes VPC Flow Logs, DNS queries, and CloudTrail logs — no agent installation needed on EC2 instances.
- **Organization-wide**: Delegated administrator enables GuardDuty across all 50 accounts from a single account. New accounts are automatically enrolled.

Specific finding types for this scenario:
- `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration`
- `CryptoCurrency:EC2/BitcoinTool.B!DNS`
- `Recon:EC2/Portscan`
- `Impact:EC2/BitcoinDomainRequest.Reputation`

### B. Amazon Inspector — ❌ Incorrect

Inspector performs vulnerability scanning (CVEs, software vulnerabilities, network exposure) — it doesn't detect runtime threats like cryptocurrency mining, anomalous API usage, or credential compromise. Inspector is a preventive tool (find vulnerabilities before exploitation), while GuardDuty is a detective tool (detect active threats).

### C. CloudTrail Insights — ❌ Incorrect

CloudTrail Insights detects unusual API volume (e.g., 10x normal `RunInstances` calls), but it doesn't: identify cryptocurrency mining, detect credential compromise patterns, or analyze network traffic. It's a narrow tool for API volume anomalies, not a comprehensive threat detection service. GuardDuty uses CloudTrail as one of many data sources with much richer analysis.

### D. CloudWatch CPU monitoring — ❌ Incorrect

Monitoring CPU utilization might detect mining activity on individual instances, but: it requires an agent on every instance, it doesn't detect the root cause (compromised credentials), it doesn't identify the mining pool connections, and it generates false positives (legitimate high-CPU workloads). GuardDuty identifies mining by DNS queries and network connections to known mining infrastructure.

## Recommendations

- **GuardDuty** should be enabled in **every Region**, even Regions you don't use — attackers specifically target unused Regions where they're less likely to be noticed.
- Enable all GuardDuty **protection types**: S3, EKS, RDS, Lambda, Malware Protection — each adds detection coverage.
- Configure **automated remediation**: GuardDuty finding → EventBridge → Step Functions/Lambda → quarantine EC2 instance (remove from security group, add to isolation SG).
- Suppress known false positives using **GuardDuty suppression rules** to reduce noise.
- Use **GuardDuty Malware Protection** to automatically scan EBS volumes of instances flagged with suspicious findings.
- Combine GuardDuty with **Detective** for investigation: Detective builds security graphs showing the relationships between findings, IAM principals, and resources.

## Relevant Links

- [Amazon GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)
- [GuardDuty Finding Types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html)
- [GuardDuty in Organizations](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_organizations.html)
- [GuardDuty Malware Protection](https://docs.aws.amazon.com/guardduty/latest/ug/malware-protection.html)
