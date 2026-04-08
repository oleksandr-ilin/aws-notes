Overview
AWS Elastic Beanstalk provides a PaaS-like experience to deploy and manage applications with minimal infrastructure management.

Tasks
- Upload application bundle, choose platform, configure environment and monitor application health.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| ECS/EKS | Container portability | More setup and infra management

Limitations
- Less flexibility for deep infra customizations; opinionated platform choices.

Price info
- Pay for underlying resources (EC2, RDS, load balancers) provisioned by Beanstalk.

Network & multi-region
- Environments are regional; deploy multiple environments for HA and blue/green.

Popular use cases
- Rapid deployment for web apps and prototypes.

When Not to Use
- When full control of the infrastructure or network is necessary.

DR strategy
- Use configuration templates and snapshots; deploy environments in multiple regions as needed.
