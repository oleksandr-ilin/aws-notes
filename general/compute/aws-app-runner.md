Overview
AWS App Runner is a fully managed service for deploying containerized web applications without managing infrastructure.

Tasks
- Push container image, configure service with concurrency and autoscaling settings, monitor health.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| ECS/Fargate | More control | More infrastructure configuration

Limitations
- Limited runtime customization; opinionated service model.

Price info
- Pay for compute & requests; per-vCPU + memory pricing.

Network & multi-region
- Regional service; use ALB/CloudFront for global delivery.

Popular use cases
- Simple web apps and APIs where developer productivity is prioritized.

When Not to Use
- Complex networking or host-level customizations are required.

DR strategy
- Use IaC to recreate services and multi-region deployment patterns for resilience.
