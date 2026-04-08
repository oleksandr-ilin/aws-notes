# Amazon VPC

## Service Overview
Amazon VPC provides isolated virtual networks in AWS, defining subnets, route tables, security groups, and network ACLs.

## Tasks this service can solve
- Network isolation and segmentation for AWS resources
- Define AZ/subnet placement, NAT, and routing for workloads

## Alternatives
| Name | Type | Short description | Pro vs VPC | Cons vs VPC | Price comparison |
|---|---|---:|---|---|---|
| On-prem networks | OSS/Commercial | Local networking infrastructure | Full control and compliance | Ops overhead and hardware costs | Higher capital and ops costs

## Limitations
- IP address planning and CIDR planning are required; VPC limits (ENIs, subnets) must be managed.

## Price info
- No direct charge for VPC itself; charges apply for NAT gateways, VPN, endpoints, and data transfer.

## Network & Multi-region considerations
- VPCs are regional; use Transit Gateway, VPC peering, or PrivateLink for cross-VPC or cross-region connectivity.

## When Not to Use This Service
- N/A — VPC is fundamental to AWS networking; choose designs carefully but VPC is required for AWS resources.

## DR strategy
- Keep network templates and IaC in source control; maintain documented IP plans and recovery steps to recreate VPCs and routing in another region.

## Popular use cases
- Standard networking foundation for all EC2, ECS/EKS, RDS, and other VPC-backed services.
