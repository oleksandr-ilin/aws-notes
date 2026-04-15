# Q01: Systems Manager for Patch Management and Configuration Compliance

## Question

A financial services company runs 500 EC2 instances across 8 AWS accounts in 3 Regions. The security team requires:
- OS patches classified as "Critical" and "Important" must be applied within 72 hours of release
- Patch compliance status must be visible in a single dashboard across all accounts
- Instances that fail to patch within the window must be automatically flagged and isolated from production traffic
- Application configuration files must be consistent across all instances running the same application tier
- Patching must not cause downtime during business hours (8 AM – 6 PM local time)

Which combination of AWS services should the solutions architect use?

## Options

- **A.** Use Systems Manager Patch Manager with patch baselines scoped to "Critical" and "Important" classifications. Create maintenance windows scheduled outside business hours per Region. Use Systems Manager State Manager associations to enforce application configuration files across tiers. Enable Systems Manager Inventory and use Systems Manager Explorer (or Resource Data Sync to a central S3 + Athena) for cross-account compliance visibility. Create a Config rule that flags non-compliant instances and triggers an SSM Automation runbook to move them to an isolated security group.
- **B.** Use AWS Inspector to scan for missing patches across all accounts. Create a Lambda function on a cron schedule to run `yum update` via SSM Run Command on all instances simultaneously. Monitor patch status with individual CloudWatch dashboards per account.
- **C.** Deploy a third-party patch management tool (WSUS/SCCM) on EC2 in each account. Configure agents on all instances. Build a custom dashboard in QuickSight pulling from each tool's database. Use EventBridge rules to detect unpatched instances.
- **D.** Use AWS Patch Manager with default patch baselines. Run patching during business hours with `NoReboot` option to avoid downtime. Use CloudTrail to track patching activity. Manually review patch compliance monthly.

## Answers

### A. Patch Manager + State Manager + Explorer + Config auto-remediation — ✅ Correct

This addresses every requirement:
- **Patch Manager with custom baselines**: Define baselines that approve "Critical" and "Important" patches with a 0-day auto-approval delay. Patches matching these classifications are approved immediately on release, giving the 72-hour window for scheduled application.
- **Maintenance windows**: Schedule patching outside business hours — different windows per Region to respect local time zones. Maintenance windows orchestrate Run Command tasks against target instance groups.
- **State Manager associations**: Declaratively enforce configuration files — State Manager continuously checks and re-applies desired state (e.g., application config files, agent configurations) on a schedule. Drift is auto-corrected.
- **Cross-account visibility**: Systems Manager Explorer with delegated administrator, or Resource Data Sync (replicates inventory and compliance data to a central S3 bucket for Athena/QuickSight analysis) provides organization-wide patching dashboards.
- **Auto-isolation**: AWS Config custom rule detects instances non-compliant with patch baseline > 72 hours → triggers SSM Automation runbook that modifies the instance's security group to an isolated group blocking production traffic.

### B. Inspector + Lambda + Run Command — ❌ Incorrect

- **Inspector** scans for vulnerabilities (CVEs) but does not manage or apply patches — it's a detection tool, not a remediation tool.
- Running `yum update` simultaneously on all instances risks outages and doesn't respect maintenance windows or business hours.
- Per-account CloudWatch dashboards don't provide centralized cross-account visibility.
- No configuration management capability (State Manager equivalent).
- No automatic isolation mechanism.

### C. Third-party tools — ❌ Incorrect

Deploying WSUS/SCCM in every account introduces significant operational overhead: infrastructure to manage, licenses, agent deployments, and cross-account networking. Custom QuickSight dashboards require building and maintaining ETL pipelines from each tool. EventBridge cannot natively detect "unpatched instances" — it needs a source event. This is over-engineered when AWS-native tools provide all required functionality.

### D. Default baselines + NoReboot + manual review — ❌ Incorrect

- **Default patch baselines** may not match the required severity scope — custom baselines ensure exactly "Critical" and "Important" are targeted.
- **Patching during business hours** directly violates the no-downtime-during-business-hours requirement.
- **NoReboot** means kernel patches aren't applied until the next reboot — the instance is technically unpatched and may remain vulnerable.
- **CloudTrail** logs API calls but doesn't show patch compliance status.
- **Monthly manual review** violates the 72-hour requirement — gaps of up to 30 days are possible.

## Recommendations

- **Custom patch baselines** per OS: separate baselines for Amazon Linux, Windows, Ubuntu — each with classification filters matching your policy.
- **Maintenance windows** support rate expressions and cron — use Region-aware schedules to avoid business hours globally.
- **State Manager** goes beyond configuration: it can ensure SSM Agent versions, run compliance checks, and enforce desired state continuously.
- **Resource Data Sync** centralizes inventory + patch compliance data — combine with **Athena + QuickSight** for executive dashboards.
- For **auto-isolation**, use the `AWS-MoveInstanceToIsolationSecurityGroup` SSM Automation runbook or build a custom one that also notifies the security team via SNS.
- Enable **Systems Manager Default Host Management Configuration** for automatic SSM Agent management without manual instance profile setup.

## Relevant Links

- [Systems Manager Patch Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-patch.html)
- [Patch Baselines](https://docs.aws.amazon.com/systems-manager/latest/userguide/about-patch-baselines.html)
- [Maintenance Windows](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-maintenance.html)
- [State Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-state.html)
- [Resource Data Sync](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-inventory-datasync.html)
