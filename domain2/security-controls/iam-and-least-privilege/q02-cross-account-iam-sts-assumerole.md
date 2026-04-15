# Q02: Cross-Account IAM Access with STS AssumeRole

## Question

A company has an AWS Organization with a Central Security account and 50 workload accounts. The security team (12 analysts) in the Central Security account needs read-only access to CloudTrail logs, GuardDuty findings, and VPC Flow Logs across all 50 workload accounts. Requirements:

- Analysts must authenticate ONCE in the Central Security account (SSO via IAM Identity Center)
- Access to workload accounts must use temporary credentials (no long-lived access keys)
- Analysts can only assume read-only roles — they must not be able to modify any resources
- Access must be auditable — every cross-account role assumption must be logged
- An analyst's session must expire after 1 hour maximum
- A compromised analyst credential must not be able to access more than 3 workload accounts simultaneously

Which IAM architecture meets all requirements?

## Options

- **A.** Create an `SecurityAuditRole` in each workload account with a trust policy allowing the Central Security account. Attach the `SecurityAudit` AWS managed policy (read-only). Set max session duration to 1 hour. In the Central Security account, create an IAM policy for analysts allowing `sts:AssumeRole` only on `arn:aws:iam::*:role/SecurityAuditRole` with a condition `aws:ResourceTag/AllowedAccounts` limiting to 3 accounts per analyst. Analysts use IAM Identity Center to authenticate, then assume cross-account roles via STS. CloudTrail in each account logs all `AssumeRole` events.
- **B.** Create IAM users in each of the 50 workload accounts for each analyst (600 accounts total). Attach read-only policies. Use password rotation every 90 days. Analysts log into each account separately via the console.
- **C.** Create a single IAM role in the Central Security account with a resource-based policy granting read access to all 50 workload accounts' CloudTrail, GuardDuty, and VPC Flow Logs. Analysts assume this single role to access all accounts.
- **D.** Use AWS Resource Access Manager (RAM) to share CloudTrail logs, GuardDuty findings, and VPC Flow Logs from all 50 accounts to the Central Security account. Analysts access only local resources — no cross-account role assumption needed.

## Answers

### A. Cross-account SecurityAuditRole + STS AssumeRole + session limits + CloudTrail audit — ✅ Correct

This follows the AWS-recommended hub-and-spoke cross-account access pattern:

- **Cross-account IAM roles with STS AssumeRole**:
  - Each workload account has a `SecurityAuditRole` with a trust policy:
    ```json
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::CENTRAL_ACCOUNT:root"},
      "Action": "sts:AssumeRole",
      "Condition": {"StringEquals": {"aws:PrincipalTag/Team": "SecurityAnalysts"}}
    }
    ```
  - Analysts call `sts:AssumeRole` to get temporary credentials (access key, secret key, session token) scoped to the workload account. Credentials expire after MAX 1 hour (set via `MaxSessionDuration` on the role).
  - **Temporary credentials only**: No long-lived access keys. STS tokens auto-expire. If a laptop is stolen, the token expires within 1 hour — no revocation needed.

- **IAM Identity Center (SSO) authentication**:
  - Analysts authenticate once via IAM Identity Center (SAML-based SSO to corporate IdP). Identity Center provides a session token for the Central Security account.
  - From the central account, analysts assume cross-account roles. One authentication → multiple account access via STS.

- **SecurityAudit managed policy**:
  - AWS managed policy `SecurityAudit` (ARN: `arn:aws:iam::aws:policy/SecurityAudit`) grants read-only access to security-relevant services: CloudTrail, GuardDuty, VPC Flow Logs, Config, IAM, and more. It does NOT grant modify/write permissions.
  - Using an AWS managed policy ensures it's updated as new AWS services launch — no maintenance required.

- **Limiting concurrent account access**:
  - The analyst's IAM policy in the Central Security account restricts which accounts they can `AssumeRole` into. Using `aws:ResourceAccount` or resource ARN conditions, each analyst is assigned a subset of accounts (e.g., 3 accounts). A compromised credential can only access those 3 accounts — not all 50.
  - Alternative: Use IAM Identity Center permission sets with account assignments to control which accounts each analyst can access.

- **CloudTrail audit logging**:
  - Every `sts:AssumeRole` call is logged in CloudTrail in BOTH accounts: the calling account (Central Security) and the target account (workload). The log includes: who assumed the role, which account, which role, timestamp, source IP, and session duration. This provides complete audit trail.

### B. IAM users in every workload account — ❌ Incorrect

- **600 IAM users (12 analysts × 50 accounts)**: Massive identity sprawl. Each user needs creation, credential management, password rotation, and eventual deprovisioning. When an analyst leaves, 50 accounts must be updated.
- **Long-lived credentials**: IAM user passwords and access keys are long-lived. Even with 90-day rotation, a compromised password provides access for up to 90 days.
- **No single sign-on**: Analysts must log into each account separately — 50 separate logins for full coverage. This is operationally unworkable.
- **Violates temporary credential requirement**: IAM user sessions use long-lived credentials (access keys or console passwords), not STS temporary credentials.

### C. Single role in Central Security with cross-account resource access — ❌ Incorrect

- **Resource-based policies for cross-account read access**: Only some AWS services support resource-based policies for cross-account access (S3, KMS, Lambda, SQS, SNS). GuardDuty findings and VPC Flow Logs do NOT have resource-based policies that allow cross-account access. CloudTrail logs stored in S3 could have bucket policies, but GuardDuty and Flow Logs require in-account IAM roles to access.
- **Single role = blast radius**: If the single role is compromised, the attacker has access to all 50 accounts' data simultaneously. The requirement limits a compromised credential to 3 accounts.
- This architecture is technically infeasible for most AWS security services.

### D. AWS RAM for sharing — ❌ Incorrect

- **RAM-supported resources**: RAM supports sharing specific resource types (VPC subnets, Transit Gateways, Route 53 Resolver rules, License Manager configs, etc.). **CloudTrail logs, GuardDuty findings, and VPC Flow Logs cannot be shared via RAM.**
- GuardDuty has its own multi-account model (delegated administrator), and CloudTrail can be configured as an Organization trail — but these are service-specific features, not RAM.
- Even if sharing were possible, it wouldn't address authentication, session limits, or blast radius control requirements.

## Recommendations

- **STS AssumeRole** is the standard for cross-account access — never create IAM users in target accounts.
- **Max session duration**: Set to the minimum necessary (1 hour for interactive use, 15 minutes for automated tools). Shorter sessions reduce the window of exposure for compromised credentials.
- **Use IAM Identity Center** (AWS SSO) to centralize authentication and map analysts to account/role combinations via permission sets.
- **Tag-based access control**: Use `aws:PrincipalTag` conditions in trust policies and `aws:ResourceAccount` conditions in IAM policies to limit which analysts can access which accounts.
- **CloudTrail Organization trail**: Enable an Organization trail in the management account to capture all `AssumeRole` events across all accounts in a single, centralized location.
- **ExternalId for third-party access**: When granting cross-account access to external parties (SaaS vendors, consultants), always require `sts:ExternalId` in the trust policy to prevent the "confused deputy" attack.

## Relevant Links

- [Cross-Account IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
- [STS AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)
- [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [SecurityAudit Managed Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html#jf_security-auditor)
- [Confused Deputy Problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)
