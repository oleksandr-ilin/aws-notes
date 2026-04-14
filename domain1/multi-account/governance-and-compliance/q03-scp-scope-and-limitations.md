# Q03: Service Control Policies — Scope and Limitations

## Question

An organization uses AWS Organizations with SCPs to enforce governance. A developer in a member account holds the `AdministratorAccess` IAM policy. The management account has applied an SCP to the developer's OU that denies `iam:CreateUser` and restricts API calls to us-east-1 only. The developer discovers they can still create IAM users.

What is the MOST LIKELY explanation for this behavior?

## Options

- **A.** The SCP is applied to the OU but the developer's account was moved to a different OU that doesn't have the SCP attached.
- **B.** The `AdministratorAccess` IAM policy grants `iam:*` permissions that override the SCP deny effect.
- **C.** The SCP has a syntax error in the condition key for Region restriction, causing the entire SCP to be invalid and unenforced.
- **D.** The developer is performing actions using the account root user, which is not restricted by SCPs except for certain AWS Organizations actions.
- **E.** SCPs only affect the management account, not member accounts.

## Answers

### D. Using root user — ✅ Correct

AWS account root user credentials are **not restricted by SCPs** for most actions. SCPs affect IAM users and roles in member accounts, but the root user bypasses SCP restrictions. The only SCP-enforceable actions on root users are specific service-related actions. If the developer has access to the root user credentials, they can bypass the SCP.

**Important nuance**: AWS recently changed root user behavior with **centralized root access** in AWS Organizations. With this feature enabled, member account root users can be further restricted. However, by default (without centralized root access), root users bypass SCPs.

### A. Account moved to different OU — ❌ Incorrect

While this is possible, the question states the SCP is applied to "the developer's OU" — implying the account is currently in that OU. This would be a configuration error, not a fundamental explanation. However, in a real scenario, you should verify OU membership.

### B. IAM overrides SCP — ❌ Incorrect

This is a fundamental misunderstanding. SCPs set the **maximum available permissions** — they act as a permissions boundary at the account level. Even if an IAM policy grants `iam:*`, the SCP deny takes precedence. IAM policies cannot override SCP denials. The effective permissions are the intersection of SCP and IAM policies.

### C. Invalid SCP syntax — ❌ Incorrect

If an SCP has a syntax error, AWS Organizations will reject it when you try to attach it. SCPs are validated at attachment time. A successfully attached SCP is syntactically valid. It may have logical errors (e.g., wrong condition key), but the question states the Region restriction also isn't working, suggesting a broader bypass (root user) rather than a syntax issue.

### E. SCPs only affect management account — ❌ Incorrect

This is backwards. SCPs do **not** affect the management account — they only affect member accounts. The management account is always exempt from SCPs, which is why the management account should have minimal resources and strictly controlled access.

## Recommendations

- **SCPs + IAM = effective permissions**. SCPs set the ceiling; IAM policies determine what principals can do within that ceiling.
- **Root user bypass**: Secure root user credentials with MFA, disable root access keys, and use **centralized root access** (Organizations feature) to prevent root user API access in member accounts.
- **Management account** is exempt from SCPs — never use it for workloads or grant developer access.
- SCP deny statements are absolute — even `AdministratorAccess` IAM policy cannot override them (for IAM principals, not root).
- Test SCPs in a **sandbox OU** before applying to production OUs to avoid unintended lockouts.

## Relevant Links

- [SCP Effects on Permissions](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html#scp-effects-on-permissions)
- [Tasks Requiring Root User Credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-tasks.html)
- [Centralized Root Access](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html)
- [SCP Syntax](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_syntax.html)
