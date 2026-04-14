# Q02: Federated Access with IAM Identity Center

## Question

A company with 2,000 employees uses Microsoft Azure AD (now Entra ID) as its corporate identity provider. The company has 40 AWS accounts under AWS Organizations. They need to:
- Allow employees to sign in once with their corporate credentials and access any authorized AWS account
- Assign different permission sets based on job role (developers get PowerUserAccess, admins get AdministratorAccess, auditors get ReadOnlyAccess)
- Automatically provision and deprovision AWS access when employees join or leave the company
- Provide a single portal for employees to see all their authorized AWS accounts

Which approach should the solutions architect recommend?

## Options

- **A.** Configure AWS IAM Identity Center (formerly SSO) with Azure AD as the external identity provider via SAML 2.0. Enable automatic provisioning (SCIM) to sync users and groups. Create permission sets mapped to Azure AD groups. Users access the AWS Access Portal for single sign-on.
- **B.** Configure SAML 2.0 federation directly in IAM for each of the 40 accounts. Create IAM roles in each account that trust the Azure AD SAML provider. Users assume roles via the SAML assertion.
- **C.** Use Amazon Cognito User Pools with Azure AD as a federated identity provider. Create Cognito groups mapped to IAM roles. Distribute Cognito app client credentials to employees.
- **D.** Sync all Azure AD users to AWS IAM as IAM users using a Lambda function. Assign IAM policies based on group membership. Disable Azure AD credentials and use IAM-only authentication.

## Answers

### A. IAM Identity Center with SAML 2.0 + SCIM — ✅ Correct

AWS IAM Identity Center is the recommended service for enterprise SSO:
- **SAML 2.0 integration with Azure AD**: Employees authenticate with their existing corporate credentials — single sign-on.
- **SCIM automatic provisioning**: Users and groups are automatically synced from Azure AD to IAM Identity Center. When an employee is disabled in Azure AD, their AWS access is automatically revoked.
- **Permission sets**: Define once, assign to groups per account. A "Developer" permission set with PowerUserAccess can be assigned to the "Developers" Azure AD group across 10 accounts. Changes to the permission set propagate automatically.
- **AWS Access Portal**: Centralized web portal showing all accounts and roles the user is authorized for. One-click to sign into any account.
- **Organization-level management**: Works natively with AWS Organizations — manage access across all 40 accounts from one place.

### B. Per-account SAML federation — ❌ Incorrect

Configuring SAML 2.0 federation in each of the 40 accounts requires creating SAML identity providers and IAM roles in every account individually. This doesn't scale — adding a new permission set means updating 40 accounts. There's no centralized portal, and users must navigate to each account's sign-in URL. IAM Identity Center eliminates this per-account management.

### C. Amazon Cognito — ❌ Incorrect

Cognito is designed for customer-facing application authentication (web/mobile apps), not enterprise workforce SSO to AWS accounts. It doesn't provide an AWS Access Portal, permission sets, or native Organizations integration. Using Cognito for AWS console access is architecturally inappropriate.

### D. IAM user sync — ❌ Incorrect

Creating IAM users for 2,000 employees violates the principle of "federate, don't create IAM users." This creates credential management overhead, doesn't leverage existing corporate identity, requires password management in AWS, and creates orphaned accounts when the sync Lambda fails or has delays. It also exceeds the IAM users-per-account limit (5,000) in some scenarios.

## Recommendations

- **IAM Identity Center** is AWS's recommended approach for workforce identity federation. It replaces the older per-account SAML pattern.
- Enable **SCIM provisioning** for automated user lifecycle management. Manual user management in IAM Identity Center quickly becomes unmanageable.
- **Permission sets** support both AWS managed policies and custom inline policies — start with managed policies and customize as needed.
- IAM Identity Center supports **attribute-based access control (ABAC)** — pass Azure AD attributes as session tags for dynamic, attribute-based IAM policies.
- For **temporary elevated access** (break-glass), consider a separate IAM Identity Center permission set with MFA required and session duration limits.

## Relevant Links

- [AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [Connect to Azure AD](https://docs.aws.amazon.com/singlesignon/latest/userguide/gs-ad.html)
- [SCIM Provisioning](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html)
- [Permission Sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)
