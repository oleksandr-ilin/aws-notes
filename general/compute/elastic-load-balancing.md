Overview
Elastic Load Balancing (ALB/NLB/CLB) distributes incoming traffic across targets to provide high availability and fault tolerance.

Tasks
- Create load balancer, attach target groups, configure listeners and health checks.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Third-party LB (F5, NGINX) | Advanced features | More ops & licensing

Limitations
- Feature differences between ALB/NLB/CLB; per-connection and per-GB pricing.

Price info
- Per-hour + per-GB processed depending on LB type.

Network & multi-region
- Load balancers are regional per AZ; use Global Accelerator or Route53 for global failover.

Popular use cases
- HTTP routing, TLS termination, and TCP load balancing.

When Not to Use
- Extremely low-latency intra-cluster traffic where host-level routing is preferred.

DR strategy
- Use multi-AZ target groups and health checks; recreate LB via IaC in DR runs.
