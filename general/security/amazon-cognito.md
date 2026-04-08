Overview
Managed user authentication and federation for web and mobile applications.

Tasks
- Configure user pools, identity pools, federated IdPs, and custom auth flows.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Auth0 / external IdP | Rich features & integrations | Cost & external dependency |
| Custom auth service | Full control | Security and maintenance burden |

Limitations
- Some advanced enterprise SSO features are limited; user management UI is basic.

Price info
- Charged per MAU (monthly active users) for some features; free tier for low usage.

Network & multi-region
- Regional service; design for global user bases with identity federation.

Popular use cases
- App sign-up/sign-in, social login, and temporary AWS credentials for clients.

When Not to Use
- Full enterprise SSO where integrated SSO providers may be preferable.

DR strategy
- Backup Cognito configuration as IaC and ensure fallback authentication paths for emergency access.
