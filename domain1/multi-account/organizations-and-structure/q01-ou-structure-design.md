# Q01: AWS Organizations OU Structure Design

## Question

A global enterprise is migrating to AWS and needs to design an organizational unit (OU) structure for AWS Organizations. The company has the following requirements:
- 4 business divisions, each with production and non-production workloads
- A shared services team providing centralized logging, DNS, and identity
- A security team that manages GuardDuty, Security Hub, and CloudTrail across all accounts
- Sandbox accounts for developer experimentation with strict spending limits
- All production accounts must enforce encryption at rest and deny public S3 access
- Non-production accounts need more permissive policies for testing

Which OU structure should the solutions architect recommend?

## Options

- **A.** Create one OU per business division (4 OUs). Place production and non-production accounts for each division in the same OU. Create separate accounts for shared services and security within the management account.
- **B.** Create OUs by environment type: Production OU, Non-Production OU, Shared Services OU, Security OU, and Sandbox OU. Place each business division's accounts in the appropriate environment OU. Apply SCPs at the OU level.
- **C.** Create a flat structure with all accounts directly under the root. Use resource tags to differentiate production from non-production. Apply IAM policies per account for governance.
- **D.** Create nested OUs: a top-level OU per business division, with child OUs for Production and Non-Production under each. Add Shared Services OU, Security OU, and Sandbox OU at the top level. Apply SCPs at appropriate levels.

## Answers

### B. Environment-based OUs — ✅ Correct

Organizing by environment type aligns SCPs with security requirements:
- **Production OU**: Apply strict SCPs (enforce encryption, deny public S3, restrict Region usage). All business divisions' production accounts inherit the same baseline security.
- **Non-Production OU**: Apply permissive SCPs allowing testing flexibility.
- **Security OU**: Dedicated OU for centralized security services (GuardDuty delegated admin, Security Hub aggregation, CloudTrail organization trail).
- **Shared Services OU**: Centralized logging, DNS (Route 53), Active Directory, identity federation.
- **Sandbox OU**: Apply SCPs with spending limits and restricted services.

This structure is the **AWS recommended approach** and aligns with the AWS multi-account strategy whitepaper. SCPs applied at the OU level automatically apply to all accounts within, making governance scalable.

### D. Nested OUs (division > environment) — ❌ Incorrect

Nested OUs add complexity without clear benefit here. The key governance boundary is environment type (prod vs. non-prod), not business division. With nested OUs, you'd need to duplicate SCPs across 4 Production child OUs and 4 Non-Production child OUs, or manage SCP inheritance carefully. This adds management overhead. Nested OUs are useful only when business divisions need fundamentally different security policies.

### A. Division-based OUs — ❌ Incorrect

Placing production and non-production accounts in the same OU means you can't differentiate SCPs by environment. You'd need account-level SCPs or IAM policies instead, which don't scale. The shared services and security workloads should never run in the management account — the management account should have minimal resources.

### C. Flat structure with tags — ❌ Incorrect

A flat structure provides no SCP hierarchy, meaning every policy must be applied account-by-account. Resource tags are useful for cost allocation but cannot enforce governance policies. IAM policies are account-scoped and can be modified by account administrators, unlike SCPs which are controlled by the organization management.

## Recommendations

- Follow the **AWS recommended OU structure**: Security, Infrastructure/Shared Services, Sandbox, Workloads (with Prod/Non-Prod sub-OUs if needed).
- Keep the **management account** minimal — no application workloads. Use it only for Organizations management and billing.
- Use **delegated administrator** for services like GuardDuty, Security Hub, and CloudFormation StackSets to avoid using the management account.
- Apply **restrictive SCPs at the root** (e.g., deny unused Regions) and **environment-specific SCPs at the OU level**.
- Consider an **Exceptions OU** for accounts that need temporary SCP exemptions during migrations.

## Relevant Links

- [AWS Organizations Best Practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html)
- [Organizing Your AWS Environment Using Multiple Accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)
- [Recommended OUs](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/recommended-ous.html)
- [SCPs Inheritance](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_inheritance_auth.html)
