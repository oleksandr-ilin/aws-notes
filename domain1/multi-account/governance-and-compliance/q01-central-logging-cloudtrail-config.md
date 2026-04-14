# Q01: Central Logging with CloudTrail and Config

## Question

An enterprise with 80 AWS accounts needs a centralized logging strategy. The security team requires: all API activity from all accounts aggregated in a single, immutable log repository; configuration change history for all resources across all accounts; real-time alerts when specific API calls are made (e.g., `DeleteTrail`, `StopLogging`, `DisableKey`); and log data retained for 7 years for regulatory compliance. No individual account administrator should be able to disable logging or delete logs.

Which combination of services and configurations should the solutions architect implement? **(Select THREE.)**

## Options

- **A.** Create an AWS CloudTrail organization trail in the management account. Configure the trail to deliver logs to an S3 bucket in a dedicated Log Archive account. Enable CloudTrail log file validation and encrypt logs with a KMS key managed in the security account.
- **B.** Enable AWS Config across all accounts using an AWS Config aggregator in the security account. Record configuration changes for all supported resource types. Deliver Config snapshots and history to the same Log Archive S3 bucket.
- **C.** Create an Amazon EventBridge rule in each account that matches the high-risk API calls (`DeleteTrail`, `StopLogging`, `DisableKey`). Route the events to a centralized Amazon SNS topic in the security account for real-time alerts.
- **D.** Deploy AWS CloudTrail individually in each of the 80 accounts. Each account stores its own trail in a local S3 bucket. Use a custom Lambda function to periodically copy logs to the central Log Archive bucket.
- **E.** Apply an SCP to deny `cloudtrail:DeleteTrail`, `cloudtrail:StopLogging`, and `s3:DeleteObject` on the Log Archive bucket for all accounts except the management account. Apply S3 Object Lock with compliance mode and a 7-year retention period on the Log Archive bucket.
- **F.** Use Amazon CloudWatch Logs as the central log repository. Stream all CloudTrail and Config logs to a central CloudWatch Logs group with a 7-year retention policy.

## Answers

### A. Organization trail + Log Archive + encryption — ✅ Correct

An **organization trail** is created once in the management account and automatically applies to all current and future member accounts. Logs are delivered to a centralized S3 bucket in a dedicated Log Archive account, keeping log data separate from the management account. KMS encryption and log file validation ensure integrity and confidentiality. This is the simplest, most scalable approach for centralized CloudTrail logging.

### B. Config aggregator + centralized delivery — ✅ Correct

AWS Config with an **organization-level aggregator** collects configuration data from all 80 accounts into a single view. This provides configuration change history for all resources, compliance evaluation results, and resource inventory. Delivering to the Log Archive bucket centralizes all compliance data alongside CloudTrail logs.

### E. SCP protection + S3 Object Lock — ✅ Correct

**SCPs** prevent any account administrator from disabling CloudTrail or deleting logs — this is critical for immutability. Denying `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail` means even account root users (when SCPs are enforced) cannot disable logging. **S3 Object Lock in compliance mode** provides WORM (write-once-read-many) protection — objects cannot be deleted or overwritten by any user, including the AWS account root user, for the specified retention period (7 years). This meets the regulatory retention requirement.

### C. EventBridge rules per account — ❌ Incorrect

While EventBridge can detect specific API calls, creating rules in each of 80 accounts is operationally burdensome. A better approach is to use the organization trail's CloudWatch Logs integration or create EventBridge rules at the organization level. Also, the organization trail combined with metric filters or CloudWatch alarms on specific events provides the alerting capability more cleanly.

### D. Individual CloudTrail per account — ❌ Incorrect

Organization trails eliminate the need to configure CloudTrail in each account individually. Managing 80 separate trails with a custom Lambda for log copying is error-prone and operationally expensive. Organization trails centralize this by design.

### F. CloudWatch Logs as central repository — ❌ Incorrect

CloudWatch Logs is significantly more expensive than S3 for long-term storage, especially for 7 years of log data across 80 accounts. CloudWatch Logs maximum retention is 10 years, but the cost per GB stored is much higher than S3 with lifecycle policies (e.g., transition to Glacier Deep Archive after 90 days).

## Recommendations

- **Organization trail** = single trail covering all accounts. Always use this instead of per-account trails.
- **Log Archive account** should be a highly restricted account — only the security team should have access.
- **S3 Object Lock compliance mode** provides the strongest immutability guarantee — not even the root user can delete objects.
- Use **S3 lifecycle policies** to transition logs to Glacier/Glacier Deep Archive for cost-effective long-term retention.
- Enable **CloudTrail Insights** for detecting unusual API activity patterns at the organization level.

## Relevant Links

- [AWS CloudTrail Organization Trail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-trail-organization.html)
- [AWS Config Aggregator](https://docs.aws.amazon.com/config/latest/developerguide/aggregate-data.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [SCP Examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples.html)
