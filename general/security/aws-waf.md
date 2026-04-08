Overview
Web Application Firewall to protect HTTP/S endpoints from common web exploits.

Tasks
- Define rules, managed rule groups, and attach to CloudFront or ALB/API Gateway.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Third‑party WAF (Cloudflare, Akamai) | Additional global features | Vendor lock-in and cost |
| App-level input validation | No infra cost | Less comprehensive protection |

Limitations
- Rule capacity and managed rule costs; testing rules can cause false positives.

Price info
- Per-rule or per-request charges depending on deployment and managed rules.

Network & multi-region
- Attach at edge (CloudFront) or regional endpoints; ensure global protection via CloudFront.

Popular use cases
- Protect web apps from OWASP top 10, bot mitigation, and common exploits.

When Not to Use
- Internal-only APIs where network controls are sufficient.

DR strategy
- Export rules as IaC and version; test rule changes in staging before production rollouts.
