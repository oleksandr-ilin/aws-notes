# Q03: IAM Access Analyzer for Unused Permissions and External Access

## Question

A company completed an audit that revealed two critical issues:
1. Multiple S3 buckets and IAM roles are shared with external AWS accounts, but the sharing was configured months ago and the business justification is unclear
2. Many IAM roles have significantly more permissions than they actually use — some roles with `AdministratorAccess` only ever call S3 and DynamoDB APIs

The security team needs to: identify all resources shared with external entities, generate least-privilege IAM policies based on actual usage, and validate that IAM policies don't grant unintended access.

Which service addresses all three requirements?

## Options

- **A.** Use AWS IAM Access Analyzer with three capabilities: External Access Analyzer to find resources shared outside the account, Unused Access Analyzer to identify unused permissions and roles, and Policy Validation to check IAM policies against best practices.
- **B.** Use AWS CloudTrail to review API call history. Manually identify external access by filtering CloudTrail events for cross-account principals. Manually compare IAM policy permissions against CloudTrail usage data.
- **C.** Use AWS Config rules to detect S3 buckets with public or cross-account policies. Use Trusted Advisor to identify IAM best practice violations. Manually review IAM policies for least privilege.
- **D.** Use Amazon Detective to investigate sharing patterns. Detective analyzes VPC Flow Logs and CloudTrail to map resource access patterns.

## Answers

### A. IAM Access Analyzer — ✅ Correct

IAM Access Analyzer provides three distinct capabilities that address all requirements:

**1. External Access Analyzer** (finding external sharing):
- Analyzes resource-based policies to identify resources shared with external principals (other AWS accounts, federated users, public access).
- Supported resource types: S3 buckets, IAM roles, KMS keys, Lambda functions, SQS queues, Secrets Manager secrets.
- Generates findings for each externally-accessible resource with details about the sharing configuration.
- Can be enabled at the organization level to monitor all accounts.

**2. Unused Access Analyzer**:
- Identifies **unused IAM roles** (roles that haven't been assumed in X days).
- Identifies **unused permissions** within active roles (permissions granted but never exercised).
- Uses CloudTrail activity data to determine actual usage vs. granted permissions.
- Can generate **policy recommendations** based on actual usage — effectively creating least-privilege policies from observed behavior.

**3. Policy Validation**:
- Checks IAM policies against AWS best practices and security warnings.
- Identifies: overly permissive statements, missing condition keys, resource wildcards, and deprecated actions.
- Can be integrated into CI/CD pipelines to validate policies before deployment.

### B. Manual CloudTrail analysis — ❌ Incorrect

While CloudTrail contains the raw data, manually analyzing API calls across hundreds of roles and correlating with IAM policies is extremely time-consuming. Access Analyzer automates this analysis using CloudTrail data under the hood. Manual analysis also doesn't identify resources with external access based on policy analysis — it only shows historical access patterns.

### C. Config + Trusted Advisor — ❌ Incorrect

Config rules can detect some policy violations (e.g., public S3 buckets), but they don't analyze IAM permissions for unused access or generate least-privilege recommendations. Trusted Advisor's IAM checks are basic (MFA, root access keys). Neither tool provides the policy generation capability based on actual usage that Access Analyzer offers.

### D. Amazon Detective — ❌ Incorrect

Detective is an investigation service that helps analyze security findings. It builds behavior graphs from CloudTrail and VPC Flow Logs, but it doesn't analyze IAM policies, generate least-privilege recommendations, or identify resources shared with external entities based on policy analysis.

## Recommendations

- Enable **External Access Analyzer** at the **Organization level** to detect cross-account and public resource sharing across all accounts.
- Run **Unused Access Analyzer** regularly to identify and remediate over-provisioned roles. Start with roles that have `AdministratorAccess` or `PowerUserAccess`.
- Use **policy generation** (based on CloudTrail activity) to create least-privilege replacement policies for over-provisioned roles. Review generated policies before applying.
- Integrate **Policy Validation** into CI/CD pipelines (CloudFormation, Terraform) to catch policy issues before deployment.
- Access Analyzer External findings can be **archived** with a reason (approved sharing) or **resolved** (sharing removed) — use this for audit documentation.

## Relevant Links

- [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
- [External Access Findings](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-findings.html)
- [Unused Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-unused-access.html)
- [Policy Generation](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-generation.html)
- [Policy Validation](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-validation.html)
