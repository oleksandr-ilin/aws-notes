# Q02: Direct Connect vs. VPN Cost Optimization

## Question

A manufacturing company is migrating to AWS and needs to connect its on-premises data center to 3 VPCs in us-east-1. The initial data migration involves transferring 50 TB of data, followed by a steady-state workload that transfers approximately 10 TB/month outbound from AWS. The company currently uses AWS Site-to-Site VPN for the connection but is concerned about the monthly data transfer costs and bandwidth limitations. The company's data center is 30 miles from an AWS Direct Connect location.

Which approach should the solutions architect recommend to optimize long-term costs?

## Options

- **A.** Continue using AWS Site-to-Site VPN. Add additional VPN tunnels to increase bandwidth. Use VPC endpoints for AWS service traffic to reduce data transfer through the VPN.
- **B.** Provision a 10 Gbps AWS Direct Connect dedicated connection. Create private virtual interfaces for each VPC. Use AWS Direct Connect data transfer pricing, which is lower than internet-based transfer pricing.
- **C.** Provision a 1 Gbps AWS Direct Connect dedicated connection for steady-state traffic. Use AWS Site-to-Site VPN as a backup connection. For the initial 50 TB migration, use AWS Snowball Edge devices.
- **D.** Use AWS Direct Connect via a hosted connection from a partner for 500 Mbps. Route only the 50 TB migration through the connection, then switch back to VPN for steady-state traffic.

## Answers

### C. 1 Gbps Direct Connect + VPN backup + Snowball for initial transfer — ✅ Correct

This approach optimizes cost at every phase:
- **Initial migration (50 TB)**: Snowball Edge avoids saturating the network for days/weeks. At 1 Gbps, transferring 50 TB would take ~4.6 days continuously — Snowball Edge is faster and avoids data transfer charges.
- **Steady-state (10 TB/month)**: Direct Connect data transfer rates are significantly lower than internet-based rates ($0.02/GB vs. $0.09/GB for the first 10 TB). At 10 TB/month, this saves ~$700/month ($8,400/year).
- **1 Gbps vs. 10 Gbps**: 10 TB/month averages ~30 Mbps — a 1 Gbps connection provides ample headroom at a fraction of the 10 Gbps port cost.
- **VPN backup**: Provides resilient connectivity if the Direct Connect link fails, at minimal additional cost.

### A. Continue VPN + VPC endpoints — ❌ Incorrect

VPN data transfer costs are internet-based rates ($0.09/GB). At 10 TB/month, that's ~$900/month in data transfer alone. Each VPN tunnel is also limited to 1.25 Gbps. VPC endpoints only help for AWS service traffic (S3, DynamoDB), not for data transfer back to on-premises. Direct Connect would save significantly on ongoing costs.

### B. 10 Gbps Direct Connect — ❌ Incorrect

A 10 Gbps connection provides massive bandwidth but the steady-state workload only averages ~30 Mbps. The port-hour cost for a 10 Gbps connection is substantially higher than 1 Gbps ($1.638/hr vs. $0.210/hr in us-east-1). Over-provisioning the connection wastes money on port charges, even though data transfer rates are the same as 1 Gbps.

### D. Temporary Direct Connect — ❌ Incorrect

Direct Connect has a cross-connect setup lead time (weeks to months) and minimum commitment terms. Using it only for the initial 50 TB migration and then switching back to VPN would forfeit the ongoing data transfer savings, which are the primary cost benefit of Direct Connect for this workload.

## Recommendations

- **Break-even analysis**: Direct Connect port charges + lower data transfer rates vs. VPN + higher transfer rates. Typically, Direct Connect becomes cost-effective at ~5+ TB/month of outbound data.
- Use **Direct Connect** for high-volume, steady-state data transfer; use **VPN** for low-volume or backup connectivity.
- Use **AWS Snowball** for bulk initial migrations to avoid network saturation and transfer costs.
- Consider **Direct Connect Gateway** to reach VPCs across multiple Regions through a single Direct Connect connection.
- Always pair Direct Connect with a **VPN backup** for resilience (different physical path).

## Relevant Links

- [AWS Direct Connect Pricing](https://aws.amazon.com/directconnect/pricing/)
- [AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)
- [AWS Snowball Edge](https://docs.aws.amazon.com/snowball/latest/developer-guide/whatisedge.html)
- [Data Transfer Pricing](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer)
