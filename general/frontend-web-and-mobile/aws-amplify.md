Overview
Framework and hosting for web and mobile applications with authentication, APIs and hosting.

Tasks
- Initialize Amplify projects, configure hosting, auth, APIs and storage.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom CI/CD + hosting | Full control | More setup and maintenance |
| Vercel / Netlify | Simpler hosting | Less AWS-native integration |

Limitations
- Opinionated structure and abstractions; may be constraining for complex backends.

Price info
- Hosting + build + backend service charges apply depending on resources used.

Network & multi-region
- Host in-region; leverage CloudFront for global distribution where needed.

Popular use cases
- Rapid prototypes, mobile backends, static sites with serverless APIs.

When Not to Use
- Large enterprises needing custom infra templates—use bespoke CI/CD.

DR strategy
- Store config in IaC, backup app builds and ensure hosting export options.
