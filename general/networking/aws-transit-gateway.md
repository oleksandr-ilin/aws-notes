# AWS Transit Gateway

## Service Overview
Transit Gateway is a hub that connects VPCs and on-prem networks, simplifying large-scale network topologies.

## Tasks this service can solve
- Centralized connectivity for many VPCs and VPNs
- Simplify routing and segmentation across accounts

## Alternatives
| Name | Type | Short description | Pro vs Transit Gateway | Cons vs Transit Gateway | Price comparison |
|---|---|---:|---|---|---|
| VPC Peering | AWS | Peer VPCs directly | Simpler for few VPCs | Not scalable for many VPCs | Peering cheaper for small scale
| Third-party SD-WAN | Commercial | Managed WAN orchestration | Service provider features & optimizations | Vendor cost and complexity | Varies; transit gateway is AWS-native

## Limitations
- Attachment limits and per-GB processing costs; design topology to avoid hairpinning.

## Price info
- Charged for attachments and data processing per GB; inter-region peering may add costs.

## Network & Multi-region considerations
- Supports inter-region peering; plan for region-level hubs and edge routing for global topologies.

## When Not to Use This Service
- Not necessary for small numbers of VPCs; VPC peering or simple VPNs may suffice.

## DR strategy
- Use redundant Transit Gateway attachments and define failover routing; keep IaC and route tables versioned for rebuild.

## Popular use cases
- Enterprise multi-VPC networks, hub-and-spoke architectures, and centralized egress or security inspection.
