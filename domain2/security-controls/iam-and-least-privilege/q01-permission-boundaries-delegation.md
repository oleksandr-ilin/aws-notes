# Q01: Least Privilege IAM Design with Permission Boundaries

## Question

A company has 50 development teams, each operating in their own AWS account within an Organization. A central platform team creates IAM roles for developers in each account. Requirements:
- Developers can create and manage Lambda functions, S3 buckets, and DynamoDB tables in their account
- Developers must NOT be able to create IAM users, escalate their own privileges, or modify account-level settings (billing, Organizations)
- Developers must be able to create IAM roles for Lambda execution, but those roles must be constrained to the same permissions the developer has
- The platform team must enforce these boundaries even if a developer writes a permissive IAM policy for their Lambda role

Which IAM strategy should the solutions architect implement?

## Options

- **A.** Create a developer IAM role with an identity-based policy allowing Lambda, S3, and DynamoDB actions. Attach a permission boundary (`DeveloperBoundary`) that denies IAM user creation, privilege escalation, and account settings modification. Require that any IAM role the developer creates must have the same `DeveloperBoundary` attached — enforce this with a condition in the identity policy: `iam:PermissionsBoundary` must equal the boundary ARN. Use SCPs at the OU level to deny `iam:CreateUser`, `account:*`, and `organizations:*` as a second layer.
- **B.** Give developers `PowerUserAccess` managed policy (full access except IAM). Add an SCP at the OU level to deny `iam:*` actions. Allow developers to create Lambda functions that use pre-created execution roles only.
- **C.** Create a custom IAM policy for developers allowing Lambda, S3, DynamoDB, and full IAM access. Trust the developers to follow a policy document describing which IAM actions are acceptable. Audit quarterly with IAM Access Analyzer.
- **D.** Use AWS Service Catalog to provide pre-approved CloudFormation templates for Lambda, S3, and DynamoDB. Developers can only launch products from the catalog. No direct API/console access to create resources.

## Answers

### A. Identity policy + permission boundary + SCP layered enforcement — ✅ Correct

This is the defense-in-depth approach:
- **Identity-based policy**: Grants the actions developers need — Lambda (create/invoke/manage), S3 (CRUD), DynamoDB (CRUD), and limited IAM (CreateRole, AttachRolePolicy, PassRole).
- **Permission boundary**: Sets the *maximum* permissions any principal can have. Even if a developer writes a `*:*` policy and attaches it to a Lambda role, the role is constrained by the boundary. The boundary denies IAM user creation, privilege escalation actions (`iam:CreateUser`, `iam:AttachUserPolicy`, `iam:PutUserPolicy`), and account-level actions.
- **`iam:PermissionsBoundary` condition**: Forces developers to attach the same boundary to any role they create. Without this condition, a developer could create an unbounded role and escalate privileges via Lambda. The condition key ensures the platform team's boundary follows all derived roles.
- **SCP backup layer**: Even if the identity policy or boundary has a gap, the SCP at the OU level provides a hard guardrail. SCPs cannot be bypassed by anyone in the member account.

### B. PowerUserAccess + SCP deny IAM — ❌ Incorrect

- **PowerUserAccess** grants access to all AWS services except IAM — but it includes actions the developers don't need (EC2, RDS, etc.), violating least privilege.
- **SCP denying all `iam:*`** prevents developers from creating Lambda execution roles — Lambda functions cannot run without a role, and if developers can't create roles, they depend on the platform team for every new function.
- Pre-created roles limit flexibility: each Lambda function may need different S3 buckets or DynamoDB tables in its policy. This creates a bottleneck on the platform team.

### C. Full IAM access + trust-based governance — ❌ Incorrect

- Giving developers full IAM access and trusting them to self-limit is a privilege escalation risk. Any developer can create an admin role, assume it, and bypass all restrictions.
- Quarterly audits detect violations after the fact — potentially months after a privilege escalation occurred.
- IAM Access Analyzer identifies public/cross-account access and unused permissions, but it doesn't prevent privilege escalation in real-time.
- This violates the principle of least privilege and provides no preventive controls.

### D. Service Catalog only — ❌ Incorrect

- Service Catalog restricts developers to pre-approved templates — but this prevents ad-hoc experimentation and slows development velocity.
- Developers cannot create custom Lambda functions with unique configurations; every permutation requires a catalog product.
- No direct API access means no CLI, no SDK — developers can't test or iterate locally.
- This approach is appropriate for production governance but is too restrictive for development teams.

## Recommendations

- **Permission boundaries** are the key mechanism for safe IAM delegation — they let users create roles while constraining what those roles can do.
- Always enforce `iam:PermissionsBoundary` conditions when allowing `iam:CreateRole` — without it, permission boundaries are bypassable.
- **Layer defenses**: identity policy (what you can do) + permission boundary (maximum possible permissions) + SCP (organizational backstop).
- Use **IAM Access Analyzer** for continuous monitoring: policy validation, external access detection, and unused permission analysis.
- Consider **AWS SSO permission sets** with session duration limits rather than long-lived IAM roles for developer access.

## Relevant Links

- [Permission Boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html)
- [Delegating IAM with Boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html#access_policies_boundaries-delegate)
- [IAM Condition Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_iam-condition-keys.html)
- [Preventing Privilege Escalation](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html)
