# AWS Direct Connect

## Service Overview
AWS Direct Connect provides dedicated network connections from on-premises to AWS for predictable bandwidth and lower latency.

## Tasks this service can solve
- High-throughput private connectivity for data migration and hybrid networks
- Consistent network performance for latency-sensitive apps

## Alternatives
| Name | Type | Short description | Pro vs Direct Connect | Cons vs Direct Connect | Price comparison |
|---|---|---:|---|---|---|
| VPN over Internet | AWS/OSS | Encrypted tunnels over internet | Easier to set up and cheaper | Less predictable performance | VPN cheaper but less consistent
| Partner hosted connections | Commercial | Provider-managed connectivity | Faster provisioning via partners | Additional vendor costs | Varies by provider

## Limitations
- Provisioning time and colocation requirements; not available everywhere.

## Price info
- Port-hours and data transfer pricing; pricing varies by port speed and location.

## Network & Multi-region considerations
- Combine with redundant VPN or multi-Direct Connect links for resiliency; use Direct Connect Gateway to reach multiple VPCs and regions.

## When Not to Use This Service
- Not suitable for small-scale or transient connectivity needs where VPN is sufficient.

## DR strategy
- Use redundant Direct Connect links and failover to VPN or other Direct Connect locations; maintain failover routing and BGP configurations in IaC.

## Popular use cases
- Bulk data transfer, SAN/NFS over AWS, low-latency connections to databases or control planes.
