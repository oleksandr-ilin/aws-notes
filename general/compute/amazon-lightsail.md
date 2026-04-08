Overview
Amazon Lightsail is a simplified VPS-like service offering predictable monthly pricing for small applications.

Tasks
- Create an instance, attach storage and networking, deploy applications via SSH or blueprints.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| EC2 | Full AWS feature set | More configuration complexity

Limitations
- Limited advanced AWS feature integration and scaling compared to EC2/ECS.

Price info
- Bundled monthly pricing per instance plan.

Network & multi-region
- Instances are regional; use snapshot export to move between regions.

Popular use cases
- Small websites, test servers, single-tenant apps.

When Not to Use
- Large, highly scalable production workloads.

DR strategy
- Use snapshots and exports; maintain IaC for replacement in another region.
