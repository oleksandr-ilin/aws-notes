Overview
Amazon EC2 Auto Scaling manages groups of EC2 instances, automatically scaling based on policies and health checks.

Tasks
- Create Auto Scaling groups, define launch templates/configs, attach health checks and scaling policies.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Manual scaling | Predictability | Manual effort and delays

Limitations
- Requires careful lifecycle and policy tuning; instance warm-up and termination behaviors to manage.

Price info
- No service fee; pay for EC2 instances launched by scaling policies.

Network & multi-region
- Group operates within a region across AZs; coordinate scaling with load balancers and target groups.

Popular use cases
- Automatic scaling of web servers, worker fleets, and stateful workloads with sticky sessions.

When Not to Use
- Very small single-instance setups.

DR strategy
- Keep launch templates and lifecycle hooks in IaC; automate cross-region AMI replication for failover.
