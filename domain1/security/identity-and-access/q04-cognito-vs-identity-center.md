# Q04: Amazon Cognito vs IAM Identity Center for Third-Party IdP Integration

## Question

A company is building two applications that need integration with third-party identity providers:

**Application 1**: A customer-facing mobile app used by 500,000 external users. Users sign up with email/password or through their existing Google, Facebook, or Apple accounts. The app calls Amazon API Gateway endpoints protected by JWT tokens.

**Application 2**: An internal operations portal used by 200 employees. The company uses Okta as its enterprise identity provider. Employees need SSO access to the portal AND to multiple AWS accounts (12 accounts) via the AWS Management Console and CLI.

Which identity solution should the solutions architect implement for each application?

## Options

- **A.** Use Amazon Cognito User Pools for Application 1 — configure social identity providers (Google, Facebook, Apple) as federated providers, use the Cognito hosted UI for sign-up/sign-in, and issue JWT tokens for API Gateway authorization. Use AWS IAM Identity Center for Application 2 — configure Okta as an external SAML 2.0 IdP with SCIM provisioning, create permission sets mapped to IAM roles, and assign users to the 12 AWS accounts.
- **B.** Use IAM Identity Center for both applications. Configure social providers as SAML 2.0 providers in Identity Center. Create permission sets for mobile app users with scoped API access. Configure Okta as a second provider for the operations portal.
- **C.** Use Amazon Cognito User Pools for both applications. Configure Okta as a SAML IdP in a Cognito User Pool for the operations portal. Use Cognito Identity Pools to vend temporary AWS credentials for Console and CLI access to the 12 accounts.
- **D.** Use custom IAM federation for both. Create individual IAM roles in each account with SAML trust policies pointing to both Okta and social IdPs. Build a custom token-exchange service to issue temporary credentials for the mobile app and Console federation URLs for the portal.

## Answers

### A. Cognito for customer-facing + IAM Identity Center for enterprise — ✅ Correct

This matches each tool to its design purpose:

**Cognito for Application 1**:
- **User Pools** are purpose-built for customer identity (CIAM): sign-up/sign-in flows, social federation (Google, Facebook, Apple via OIDC), and hosted UI.
- Cognito issues JWTs (ID token + access token) natively — API Gateway has a built-in Cognito authorizer that validates these tokens with zero custom code.
- Scales to millions of users with per-user pricing suited to consumer apps.
- Provides built-in MFA, email/SMS verification, password policies, and adaptive authentication.

**IAM Identity Center for Application 2**:
- Designed for **workforce identity**: SSO into AWS accounts and SAML-enabled business applications.
- Okta integration via SAML 2.0 + SCIM enables automatic user/group provisioning and deprovisioning.
- **Permission sets** map to IAM roles in each account — centralized assignment of access across 12 accounts from a single pane.
- Provides a user portal for Console access and CLI credential generation (via `aws configure sso`).

### B. IAM Identity Center for both — ❌ Incorrect

IAM Identity Center does not support social identity providers (Google, Facebook, Apple) as federated providers — it supports SAML 2.0 and SCIM for enterprise IdPs. It's designed for workforce (employees), not for millions of customer-facing users. It has no built-in sign-up flow, hosted UI, or consumer-grade features like email verification and adaptive authentication. Per-account pricing and management model doesn't suit 500K external users.

### C. Cognito for both — ❌ Partially incorrect

Using Cognito for the operations portal is possible (Okta → SAML → Cognito User Pool), but providing multi-account AWS Console and CLI access through Cognito Identity Pools is complex and not well-suited:
- **Cognito Identity Pools** can vend temporary credentials, but mapping users to different IAM roles across 12 accounts requires custom logic and role trust policies per account.
- There's no built-in permission set concept — you'd need to manually maintain role assignments.
- No SSO portal for AWS Console access — you'd need to build custom Console federation URLs.
- IAM Identity Center provides this out of the box with managed Console/CLI SSO and centralized permission management.

### D. Custom IAM federation — ❌ Incorrect

Building custom SAML trust per account creates significant operational burden: 12 IAM roles × multiple IdPs = many trust relationships to maintain. A custom token-exchange service introduces code to maintain, security surface to protect, and a single point of failure. Console federation URL generation requires custom code that IAM Identity Center provides natively. This approach has no advantage over the managed services and significantly increases complexity and security risk.

## Recommendations

- **Cognito User Pools** = customer identity (CIAM): external users, social login, consumer apps, JWT-based API auth.
- **IAM Identity Center** = workforce identity: employees, enterprise IdP (Okta/Azure AD/ADFS), multi-account Console/CLI SSO.
- **Cognito Identity Pools** (federated identities) bridge Cognito-authenticated users to temporary AWS credentials — useful for mobile apps needing direct S3/DynamoDB access, less suitable for multi-account Console SSO.
- For the operations portal app itself (not AWS Console), you can use IAM Identity Center's SAML integration to SSO into the custom app as well, keeping a single authentication experience.
- Cognito supports **SAML 2.0** and **OIDC** federation in User Pools for enterprise integration — but for AWS account access, IAM Identity Center is always the better choice.

## Relevant Links

- [Amazon Cognito User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html)
- [Cognito Social Identity Providers](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-social-idp.html)
- [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [IAM Identity Center External IdP](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
- [SCIM Provisioning with Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html)
