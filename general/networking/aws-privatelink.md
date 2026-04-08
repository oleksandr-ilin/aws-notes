# AWS PrivateLink

## Service Overview
AWS PrivateLink provides private connectivity to AWS services and customer/partner services via interface VPC endpoints (ENIs) without using public IPs.

## Tasks this service can solve
- Secure private access to AWS services from within VPCs
- Expose services to partner VPCs privately (service consumers/providers)

## Alternatives
| Name | Type | Short description | Pro vs PrivateLink | Cons vs PrivateLink | Price comparison |
|---|---|---:|---|---|---|
| VPC Peering | AWS | Direct VPC-to-VPC networking | Simpler for small numbers of VPCs | Not scalable for many VPCs | Peering cheaper but less scalable
| Transit Gateway | AWS | Hub-and-spoke VPC connectivity | Scales across many VPCs | Different security boundary | Transit Gateway costs can be higher for heavy traffic

## Limitations
- Endpoint quota limits and per-ENI cost; not ideal for high-scale cross-account connectivity without planning.

## Price info
- Charged per endpoint hour and data processed through endpoints.

## Network & Multi-region considerations
- PrivateLink is regional; for multi-region access, deploy endpoints per region or use inter-region networking solutions.

## When Not to Use This Service
- Not needed for simple VPC-to-VPC low-scale setups—use peering or Transit Gateway as appropriate.

## DR strategy
- Maintain endpoint configurations in IaC and use multi-AZ ENIs; replicate endpoint-service in other regions if required.

## Popular use cases
- Private access to managed services (e.g., Kinesis, ECR), partner SaaS integrations, and cross-account private APIs.
