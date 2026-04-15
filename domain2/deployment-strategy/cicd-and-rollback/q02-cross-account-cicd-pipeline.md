# Q02: Cross-Account CI/CD Pipeline with CodePipeline

## Question

A financial services company has an AWS Organization with 4 accounts: Tooling (CI/CD), Dev, Staging, and Production. The security team requires:
- All CI/CD tooling (pipelines, build projects) must reside in the Tooling account
- Deployments to each environment account must use cross-account IAM roles — no long-lived credentials
- Artifacts must be encrypted at rest with a KMS key managed by the Tooling account
- Production deployments require manual approval from two separate approvers (four-eyes principle)
- Rollback must be automatic if CloudWatch alarm "5xxErrorRate > 5%" fires within 10 minutes post-deployment

Which pipeline architecture meets all requirements?

## Options

- **A.** CodePipeline in the Tooling account with: Source (CodeCommit) → Build (CodeBuild) → Deploy-Dev (CodeDeploy cross-account via assumed IAM role in Dev account) → Deploy-Staging (CodeDeploy cross-account) → Manual Approval action (SNS notification to approvers) → Deploy-Production (CodeDeploy cross-account with alarm-based rollback configured in the deployment group). Artifact store: S3 bucket in Tooling account encrypted with a CMK, with key policy granting `kms:Decrypt` to the cross-account deployment roles.
- **B.** Separate CodePipeline in each account (Dev, Staging, Production) triggered by S3 artifact upload from the Tooling account. Each pipeline uses local IAM roles. Artifacts copied to each account's S3 bucket via Lambda. Production pipeline includes a Lambda approval function that checks for two approvals in a DynamoDB table.
- **C.** CodePipeline in the Tooling account deploying to all environments via a single IAM role with permissions across all accounts. Artifacts stored in S3 with SSE-S3 encryption. Production deployment uses a scheduled maintenance window instead of manual approval.
- **D.** AWS Proton managing deployments across all accounts. Proton templates define environment and service configurations. Production approvals handled via Proton's built-in approval workflow. Rollback via Proton's managed rollback feature.

## Answers

### A. Single CodePipeline with cross-account roles + KMS CMK + manual approval + alarm rollback — ✅ Correct

This architecture satisfies every requirement:

- **Single pipeline in Tooling account**: All CI/CD tooling is centralized. CodePipeline orchestrates the flow: source → build → deploy-dev → deploy-staging → approve → deploy-prod. One pipeline provides full visibility and auditability.

- **Cross-account deployment via assumed IAM roles**: Each target account (Dev, Staging, Production) has a deployment role (e.g., `arn:aws:iam::PROD_ACCOUNT:role/CodeDeployRole`). CodePipeline assumes this role via STS AssumeRole when executing the deploy action. The trust policy on the target account role trusts the Tooling account's pipeline role. No credentials are stored — each deployment uses temporary STS tokens.

- **KMS CMK for artifact encryption**: The S3 artifact bucket in the Tooling account uses a customer-managed KMS key. The key policy grants `kms:Decrypt` and `kms:DescribeKey` to the cross-account deployment roles — this lets CodeDeploy in the target accounts decrypt artifacts. Without this key policy, cross-account deployments fail with "Access Denied" on artifact download.

- **Manual approval with SNS**: CodePipeline's Manual Approval action sends an SNS notification (email/Slack) and blocks the pipeline until approved. For four-eyes principle, configure TWO sequential Manual Approval actions — each requiring a different approver. Alternatively, use a Lambda-backed custom action that validates two distinct IAM identities approved via API.

- **Alarm-based automatic rollback**: CodeDeploy deployment groups support alarm-based rollback. If the CloudWatch alarm `5xxErrorRate > 5%` enters ALARM state during or within 10 minutes after deployment, CodeDeploy automatically rolls back to the previous revision. This is a native CodeDeploy feature — no custom automation needed.

### B. Separate pipelines per account + S3 artifact copy + DynamoDB approvals — ❌ Incorrect

- **Separate pipelines per account**: Violates the requirement that all CI/CD tooling resides in the Tooling account. Pipelines in Dev/Staging/Production mean CI/CD resources are distributed — harder to audit, secure, and maintain.
- **Lambda copying artifacts to each account's S3 bucket**: Adds operational complexity and creates multiple artifact copies. Cross-account CodePipeline can reference a single S3 artifact store with proper KMS permissions — no copying needed.
- **DynamoDB-based approval**: Custom approval workflow requires maintaining a Lambda function, DynamoDB table, and approval UI. CodePipeline's native Manual Approval action provides this out of the box with SNS integration.
- **No mention of alarm-based rollback**: Missing the automatic rollback requirement.

### C. Single IAM role for all accounts + SSE-S3 + scheduled window — ❌ Incorrect

- **Single IAM role with cross-account permissions**: A single role with access to Dev, Staging, AND Production violates least privilege. If the role is compromised, all environments are exposed. Separate per-account roles limit the blast radius — a compromised Dev role can't deploy to Production.
- **SSE-S3 encryption**: SSE-S3 uses Amazon-managed keys that cannot have custom key policies. Cross-account access to SSE-S3 encrypted objects requires bucket policies but doesn't support fine-grained key-level permissions. Customer-managed KMS keys provide explicit key policy control — you specify exactly which principals can decrypt.
- **Scheduled maintenance window instead of manual approval**: Doesn't meet the four-eyes principle requirement. Scheduled windows allow deployments at specific times but don't require human review/approval. Automated deployments during a window could push unreviewed changes to production.

### D. AWS Proton managed deployments — ❌ Incorrect

- **AWS Proton**: Proton is a managed service for platform teams to define, provision, and manage infrastructure templates. It's designed for standardizing environments and services — not for fine-grained CI/CD pipeline control with cross-account IAM roles, custom KMS encryption, and alarm-based rollback.
- **Proton approval workflow**: Proton doesn't have a native "four-eyes" approval mechanism equivalent to CodePipeline's Manual Approval actions.
- **Proton rollback**: Proton can roll back environment and service updates, but this is at the infrastructure template level — not at the application deployment level with CloudWatch alarm triggers.
- Proton is the right tool for platform-as-a-service abstractions, not for custom deployment pipelines with specific security controls.

## Recommendations

- **Cross-account CodePipeline pattern**: Tooling account owns the pipeline → target accounts provide deployment roles → KMS key policy bridges encryption. This is the AWS-recommended pattern for enterprise CI/CD.
- **KMS key policy is the #1 cross-account pipeline blocker** — always include `kms:Decrypt`, `kms:DescribeKey`, and `kms:GenerateDataKey*` for cross-account roles.
- **Separate deployment roles per account** with least privilege: Dev role can deploy anything, Staging role requires passing tests, Production role requires manual approval before assume.
- **Four-eyes principle**: Use two sequential Manual Approval stages, or a custom Lambda action validating two distinct approving identities.
- **CodeDeploy alarm-based rollback** monitors up to 10 CloudWatch alarms — add error rate, latency, and health check alarms for comprehensive deployment safety.

## Relevant Links

- [Cross-Account CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-create-cross-account.html)
- [CodePipeline Manual Approval](https://docs.aws.amazon.com/codepipeline/latest/userguide/approvals.html)
- [CodeDeploy Automatic Rollback](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-rollback-and-redeploy.html)
- [KMS Key Policies for Cross-Account](https://docs.aws.amazon.com/kms/latest/developerguide/key-policy-modifying-external-accounts.html)
