Overview
AWS Fargate is a serverless compute engine for containers that removes the need to manage servers.

Tasks
- Define task definitions, configure service, set CPU/memory and networking, monitor tasks.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| EC2-backed ECS | More control | Manage EC2 instances

Limitations
- No host-level access; some features like daemon tasks require workarounds.

Price info
- Charged per vCPU and memory per second.

Network & multi-region
- Tasks are regional; integrate with VPC, subnets and security groups per task.

Popular use cases
- Microservices and short-lived containers where ops overhead should be minimal.

When Not to Use
- Workloads needing privileged host access or kernel modules.

DR strategy
- Keep task defs and service configs in IaC; run parallel services in another region if required.
