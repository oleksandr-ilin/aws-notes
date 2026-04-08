Overview
Service that recommends rightsizing for EC2, EBS, Lambda and Auto Scaling based on usage patterns.

Tasks
- Enable Compute Optimizer, review recommendations, and apply rightsizing or purchase changes.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Manual sizing reviews | Control | Time-consuming

Limitations
- Accuracy depends on historical usage; not all workloads lend themselves to automatic rightsizing.

Price info
- Basic recommendations free; advanced features may be billed.

Network & multi-region
- Regional; enable per-region and aggregate recommendations centrally.

Popular use cases
- Cost optimization and instance family recommendations.

When Not to Use
- Short-lived or highly variable workloads where historical data is not predictive.

DR strategy
- Archive recommendations and use IaC to apply known-good instance types.
