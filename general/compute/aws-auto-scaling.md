Overview
AWS Auto Scaling automatically adjusts resource capacity to maintain desired performance and cost targets.

Tasks
- Define scaling policies and target tracking for EC2, ECS, and other scalable resources.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Manual scaling | Predictability | Human intervention required

Limitations
- Configuration complexity across services; cooldown and policy tuning needed.

Price info
- No direct service charge; you pay for the scaled resources.

Network & multi-region
- Configured per-region for resources; coordinate scaling across AZs for HA.

Popular use cases
- Auto-scaling EC2 fleets, scaling ECS tasks, and application-layer scaling.

When Not to Use
- Single-instance or static workloads.

DR strategy
- Ensure scaling policies are part of runbooks and IaC for recovery scenarios.
