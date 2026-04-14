# Q03: Least Privilege — AdministratorAccess vs. PowerUserAccess

## Question

A company has three IAM role types for its development teams:
- **Platform Engineers**: Need full access to all AWS services including IAM to manage infrastructure and create roles for applications
- **Senior Developers**: Need full access to most AWS services for building and deploying applications but should NOT be able to create IAM users, manage IAM policies, or modify Organizations settings
- **Junior Developers**: Need access only to specific services (Lambda, DynamoDB, S3, CloudWatch) in the development account

Which IAM configuration follows the principle of least privilege correctly?

## Options

- **A.** Platform Engineers: `AdministratorAccess` managed policy. Senior Developers: `PowerUserAccess` managed policy. Junior Developers: Custom policy with explicit Allow for Lambda, DynamoDB, S3, and CloudWatch.
- **B.** All three roles: `AdministratorAccess` managed policy with different permissions boundaries to restrict access.
- **C.** Platform Engineers: `PowerUserAccess` managed policy. Senior Developers: Custom policy with all services except IAM. Junior Developers: `ReadOnlyAccess` managed policy.
- **D.** All three roles: Custom inline policies with explicitly enumerated actions for every service they need.

## Answers

### A. AdministratorAccess + PowerUserAccess + Custom — ✅ Correct

This correctly applies least privilege at each level:

**Platform Engineers — `AdministratorAccess`**:
- Grants `*:*` (all actions on all resources). Appropriate for platform engineers who manage infrastructure, IAM roles, and account settings.
- This is the only role that should have IAM management capabilities.

**Senior Developers — `PowerUserAccess`**:
- Grants full access to all AWS services **EXCEPT** IAM user/group management and Organizations. Specifically, `PowerUserAccess` includes:
  - `Allow: *:*` for all services
  - `Deny: iam:CreateUser, iam:CreateGroup, iam:AttachUserPolicy, etc.` (IAM user/group write operations)
  - But **allows** `iam:CreateServiceLinkedRole`, `iam:GetRole`, `iam:PassRole` — necessary for services like Lambda and EC2 that need to assume roles.
- This matches the requirement: full service access, no IAM user management.

**Junior Developers — Custom policy**:
- Explicitly allows only the 4 required services. Any service not explicitly allowed is implicitly denied.
- Narrowest access scope — appropriate for limited roles.

### B. AdminAccess + permissions boundaries — ❌ Incorrect

Using `AdministratorAccess` for all roles and relying on permissions boundaries to restrict access is backwards. Permissions boundaries set the **maximum** permissions that an IAM entity can have, but the entity still needs identity-based policies to actually grant permissions. Starting with full access and restricting down is less transparent and harder to audit than starting with the right level of access.

### C. PowerUserAccess for Platform Engineers — ❌ Incorrect

Platform Engineers need IAM management capabilities (creating roles, policies, service-linked roles). `PowerUserAccess` explicitly restricts IAM user/group management, making it insufficient for platform engineering work like creating application roles and configuring service permissions.

### D. Full custom inline policies — ❌ Incorrect

While technically most precise, custom inline policies for every action across all services would contain thousands of action entries, be extremely difficult to maintain, and break whenever AWS adds new service actions. AWS managed policies like `PowerUserAccess` are maintained by AWS and updated as services evolve. Use managed policies where they match the need, and custom policies for fine-grained requirements.

## Recommendations

- **AdministratorAccess** should be assigned to the minimum number of roles — typically only platform/infra engineers and break-glass accounts.
- **PowerUserAccess** is the right level for developers who need broad service access but shouldn't manage IAM identities. Key distinction:
  - PowerUserAccess **allows**: `iam:PassRole`, `iam:CreateServiceLinkedRole`, `iam:GetRole` (needed for services)
  - PowerUserAccess **denies**: `iam:CreateUser`, `iam:CreateGroup`, `iam:AttachUserPolicy` (identity management)
- Use **permissions boundaries** as a safety net on top of identity-based policies — not as the primary access control mechanism.
- For junior/limited roles, start with **AWS managed policies for specific services** (e.g., `AmazonDynamoDBFullAccess`) and customize from there.
- Regularly review IAM roles using **IAM Access Analyzer** to identify unused permissions.

## Relevant Links

- [AdministratorAccess Policy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AdministratorAccess.html)
- [PowerUserAccess Policy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/PowerUserAccess.html)
- [Permissions Boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html)
- [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
