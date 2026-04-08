Overview
Centralized identity and access management for AWS resources.

Tasks
- Manage users, roles, policies, and federation for AWS access.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| External IdP (Okta, Azure AD) | Single sign-on across apps | External dependency & cost |
| Long‑lived keys | Simplicity | Security risk |

Limitations
- Policy complexity can lead to misconfigurations; principle of least privilege is manual.

Price info
- IAM itself is free; costs come from using other services and identity providers.

Network & multi-region
- Global service (account-level); policies are effective across regions.

Popular use cases
- Cross-account roles, service principals, programmatic access with fine-grained policies.

When Not to Use
- For app-level user profiles; use Cognito or external IdPs for end-user auth.

DR strategy
- Backup critical policies and roles (IaC), and test account recovery procedures.
