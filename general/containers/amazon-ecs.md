# Amazon ECS

## Service Overview
Amazon Elastic Container Service (ECS) is a fully managed container orchestration service that runs Docker containers at scale on EC2 or Fargate.

## Tasks this service can solve
- Run containerized microservices with tight AWS integration
- Host long-running services or batch jobs with scheduled tasks
- Integrate with AWS IAM, ELB, and CloudWatch for operations

## Alternatives
| Name | Type | Short description | Pro vs ECS | Cons vs ECS | Price comparison |
|---|---|---:|---|---|---|
| Amazon EKS | AWS | Managed Kubernetes control plane | Portability and k8s ecosystem | Operational complexity | EKS control-plane fee + worker costs; ECS can be cheaper for simple setups
| Kubernetes on EC2 | OSS | Self-hosted Kubernetes | Full control & portability | Ops overhead | Higher ops cost typically
| Fargate (ECS/EKS) | AWS | Serverless containers | No servers to manage | Less host-level control | Fargate pricing per vCPU and memory; simpler ops

## Limitations
- Less native portability than Kubernetes; ECS has different API semantics.

## Price info
- No control-plane fee for ECS; pay for underlying EC2 or Fargate resources, storage, and data transfer.

## Network & Multi-region considerations
- Clusters are regional and AZ-scoped; distribute tasks across AZs and use multi-region deployments with separate clusters for global resilience. Use VPC and Service Discovery for networking.

## When Not to Use This Service
- If your team requires Kubernetes ecosystem features or portability across clouds, consider EKS or self-hosted k8s instead.

## DR strategy
- Recreate clusters via infrastructure-as-code, store task definitions and service configurations in source control, and use multi-region cluster deployments to fail over.

## Popular use cases
- Microservices, background workers, scheduled jobs, and tightly integrated AWS service workloads.
