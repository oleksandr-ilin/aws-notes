Overview
Managed API hosting with throttling, auth, and request transformation (REST + WebSocket + HTTP APIs).

Tasks
- Define APIs, stages, routes, integrations, and deploy stages with usage plans.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| ALB + Lambda | Simpler for many use cases | Less API-level features |
| Custom API servers on EC2/ECS | Full control | More ops overhead |

Limitations
- Payload size limits, cold starts on Lambda integrations, and per-request billing at scale.

Price info
- Per-request + data transfer and optional caching costs.

Network & multi-region
- Regional endpoints; for global low-latency use CloudFront or Global Accelerator in front.

Popular use cases
- Serverless APIs, partner APIs, and rate-limited public endpoints.

When Not to Use
- Extremely high RPS low-cost needs—consider ALB or self-hosted options.

DR strategy
- Keep API definitions in IaC and ensure stage deployment automation for recovery.
