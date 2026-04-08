# Amazon EC2

## Service Overview
Amazon EC2 provides resizable compute capacity in the cloud — virtual machines with full OS control, networking, and storage options.

## Tasks this service can solve
- Long-running servers and custom OS-level workloads
- Stateful applications requiring local disk or specialized drivers
- Legacy lift-and-shift applications
- High-performance computing with specific instance types

## Alternatives
| Name | Type | Short description | Pro vs EC2 | Cons vs EC2 | Price comparison |
|---|---|---:|---|---|---|
| AWS Fargate | AWS | Serverless containers | Less ops, faster scaling | Less host-level control | Often cheaper for ephemeral container workloads; EC2 better for steady-state large fleets |
| AWS Lambda | AWS | Serverless functions | No servers to manage for short jobs | Not suitable for long-running processes | Lambda cheaper for short, bursty tasks |
| Google Compute Engine | GCP | VMs on GCP | Multi-cloud parity | Different ecosystem/integrations | Comparable; varies by instance family |
| Bare metal / On-prem | Commercial | Local hardware | Full control and compliance | Hardware lifecycle & ops | Capital and operating expenses apply |

## Limitations
- Requires management for OS patching, scaling, and security hardening.
- Single-instance availability depends on AZ placement; use Auto Scaling and multi-AZ for HA.

## Price info
- Pricing models: On-demand, Reserved Instances, Savings Plans, Spot Instances. Costs depend on instance type, region, storage, and data transfer.

## Network & Multi-region considerations
- EC2 instances are AZ-scoped. Architect for multi-AZ by deploying across multiple subnets/AZs and using load balancers (ELB). For multi-region, deploy duplicated stacks and use Route 53 or global load balancing.
- Integrate with VPC, NAT gateways, and Transit Gateway for complex networks.

## Popular use cases
- Databases requiring specific OS-level tuning
- Custom middleware or agents that require kernel modules
- Applications migrated from on-premises VMs

## When Not to Use This Service
- Not ideal for short-lived, highly event-driven workloads where serverless (Lambda/Fargate) is cheaper and simpler. For containerized microservices without OS-level needs, consider Fargate or ECS.

## DR strategy
- Use multi-AZ deployments for high availability; replicate across regions by deploying duplicate stacks and using Route 53 for failover. Snapshot critical EBS volumes and automate AMI creation and cross-region replication for recovery.
