# Q03: Data Transfer Costs — VPN, Direct Connect, Transit Gateway, and Cross-AZ Optimization

## Question

A company operates a hybrid architecture connecting an on-premises data center to AWS. Current networking setup:

**Connection 1 — Primary: AWS Direct Connect**
- 1 Gbps dedicated connection in us-east-1
- Private VIF connected to a Direct Connect Gateway → 3 VPCs (Production, Staging, Shared Services)
- Monthly data transfer:
  - On-premises → AWS: 8 TB/month (database replication, application deployments)
  - AWS → on-premises: 15 TB/month (reports, backups, API responses)

**Connection 2 — Backup: Site-to-Site VPN**
- Two VPN tunnels over the internet for redundancy
- Active only when Direct Connect fails (used 2 hours last month during a DX maintenance window)
- Monthly data transfer when active: negligible

**Connection 3 — Transit Gateway**
- AWS Transit Gateway connecting all 3 VPCs + Direct Connect Gateway + VPN
- Cross-VPC traffic: Production VPC → Shared Services VPC: 5 TB/month (centralized logging, monitoring)
- Cross-VPC traffic: Staging VPC → Shared Services VPC: 2 TB/month

**Internal Architecture (Production VPC):**
- 20 EC2 instances across 3 AZs communicate with each other at 3 TB/month cross-AZ
- EC2 instances access S3 via NAT Gateway: 10 TB/month
- EC2 instances access DynamoDB via NAT Gateway: 2 TB/month
- RDS Multi-AZ in Production VPC: synchronization traffic (no charge for Multi-AZ sync)

Current monthly networking cost: $4,200. The CFO wants to reduce it to under $2,500.

Which optimizations achieve the target?

## Options

- **A.** (1) Replace NAT Gateway S3/DynamoDB access with **S3 Gateway VPC endpoint** (free) and **DynamoDB Gateway VPC endpoint** (free) — eliminates NAT Gateway data processing charges ($0.045/GB × 12 TB = $540/month) AND NAT Gateway hourly charges if no other traffic needs it. (2) Evaluate cross-AZ EC2 traffic — move communicating instances to the **same AZ** where possible, or use **private IP addresses** (cross-AZ traffic: $0.01/GB each direction = $0.02/GB round trip × 3 TB = $60/month — small but reducible). (3) For Direct Connect data transfer: outbound (AWS → on-premises) is $0.02/GB for the first 10 TB = 15 TB × ~$0.02/GB = $300/month. Inbound (on-premises → AWS) is **free** via Direct Connect. (4) Transit Gateway data processing: $0.02/GB for each GB processed by TGW. 7 TB/month × $0.02/GB = $140/month. Evaluate whether VPC Peering ($0/GB data transfer) can replace TGW for the Production → Shared Services route.
- **B.** Replace Direct Connect with Site-to-Site VPN for all traffic. VPN has no data transfer charges. Remove Transit Gateway and use VPN connections directly to each VPC.
- **C.** Compress all data before transfer to reduce volume. Add a second Direct Connect connection for redundancy and remove VPN. No architecture changes needed.
- **D.** Move all workloads to a single AZ to eliminate cross-AZ charges. Remove Transit Gateway and use direct VPC peering for all connections. Cancel Direct Connect and use VPN only.

## Answers

### A. Gateway endpoints + cross-AZ optimization + TGW-to-peering evaluation — ✅ Correct

This answer identifies the highest-impact cost optimizations:

**Optimization 1: S3 and DynamoDB Gateway VPC Endpoints (highest impact)**

- **Current cost**: EC2 instances access S3 (10 TB) and DynamoDB (2 TB) through a NAT Gateway. NAT Gateway charges:
  - **Data processing**: $0.045/GB × 12,000 GB = **$540/month**
  - **Hourly charge**: $0.045/hour × 730 hours = **$32.85/month**
  - Total NAT Gateway cost for S3 + DynamoDB traffic: **$572.85/month**

- **With Gateway VPC endpoints**:
  - S3 Gateway endpoint: **$0 data transfer, $0 hourly charge**. Traffic goes directly from EC2 to S3 over the AWS network via the gateway endpoint — never touches the NAT Gateway.
  - DynamoDB Gateway endpoint: Same — **$0 data transfer, $0 hourly charge**.
  - Gateway endpoints are FREE — no per-GB charge, no hourly charge. They add a route table entry that directs S3/DynamoDB traffic through the endpoint instead of the NAT Gateway.

- **Savings**: $572.85/month immediately. If no other traffic requires the NAT Gateway (all internet-bound traffic goes through other means), the NAT Gateway can be removed — saving the $32.85/month hourly charge plus any remaining data processing.

- **Implementation**: Create a Gateway VPC endpoint for S3 and one for DynamoDB. Associate them with the route tables of the subnets where EC2 instances run. No application code changes — the SDK automatically uses the endpoint.

- **Gateway vs. Interface endpoints**:

  | Feature | Gateway Endpoint | Interface Endpoint |
  |---|---|---|
  | Services | S3, DynamoDB only | 100+ services (Lambda, ECR, STS, etc.) |
  | Cost | **Free** (no hourly, no per-GB) | $0.01/GB processed + $0.01/hour per AZ |
  | Mechanism | Route table entry | ENI in your subnet with private IP |
  | Cross-Region | Same Region only | Same Region only |

**Optimization 2: Direct Connect Data Transfer Understanding**

- **Inbound (on-premises → AWS via DX)**: **FREE**. AWS does not charge for data transferred into AWS over Direct Connect.
  - The 8 TB/month inbound: $0

- **Outbound (AWS → on-premises via DX)**:
  - $0.02/GB for the first 10 TB/month (us-east-1 pricing)
  - $0.0125/GB for the next 40 TB/month
  - 15 TB: (10 TB × $0.02) + (5 TB × $0.0125) = $200 + $62.50 = **$262.50/month**

- **Direct Connect port charge**: $0.30/hour for 1 Gbps dedicated = $219/month (fixed — can't reduce without changing connection)

- **Key exam fact**: Direct Connect data transfer rates are LOWER than internet egress rates ($0.09/GB for internet vs. $0.02/GB for DX). This is one of the cost justifications for Direct Connect.

**Optimization 3: Transit Gateway vs. VPC Peering**

- **Transit Gateway data processing**: $0.02/GB for each GB that traverses TGW attachments.
  - Production → Shared Services: 5 TB × $0.02/GB = $100/month
  - Staging → Shared Services: 2 TB × $0.02/GB = $40/month
  - Total TGW data processing: **$140/month**
  - TGW attachment charges: $0.05/hour per attachment × 3 VPCs = $109.50/month
  - Total TGW cost: **$249.50/month**

- **VPC Peering alternative**:
  - VPC Peering: **$0 data transfer** (same-Region peering has no data transfer charge for traffic within the same AZ; cross-AZ peering traffic IS charged at $0.01/GB).
  - No hourly charge for VPC peering connections.
  - If Production and Shared Services are in the same AZ: $0.
  - If cross-AZ: $0.01/GB × 5 TB = $50/month (still cheaper than TGW $100/month + attachment fees).

- **When to keep TGW vs. switch to peering**:
  - 3 VPCs → 3 peering connections (mesh). Manageable.
  - 10+ VPCs → 45+ peering connections. TGW simplifies routing and is worth the cost.
  - For this scenario (3 VPCs), VPC Peering is cheaper. Replace TGW with peering for the Production ↔ Shared Services route (highest traffic). Keep TGW for the DX/VPN connection if needed, or connect DX to Shared Services VPC and peer from there.

**Optimization 4: Cross-AZ Traffic Reduction**

- **Cross-AZ charges**: $0.01/GB in each direction. 3 TB/month round trip = $60/month.
- **Reduction strategies**:
  - Co-locate frequently communicating services in the same AZ (use AZ-aware placement)
  - Use private IPs (not public/Elastic IPs) for same-AZ communication — same-AZ with private IPs is **free**
  - Note: cross-AZ is intentional for HA — don't sacrifice availability for small savings. Focus on reducing unnecessary cross-AZ traffic (e.g., a logging agent sending to a centralized service in a different AZ).

- **RDS Multi-AZ sync**: Explicitly **free** — AWS does not charge for data replication between RDS primary and standby in Multi-AZ deployments. Same for Aurora Multi-AZ replication.

**Total savings summary**:
- Gateway endpoints (eliminate NAT GW for S3/DynamoDB): **-$573/month**
- TGW → VPC Peering (Production ↔ Shared Services): **-$200/month** (estimated)
- Cross-AZ optimization: **-$30/month** (partial reduction)
- Total reduction: ~$800/month → from $4,200 to ~$3,400

- **Additional optimizations to reach $2,500**: Evaluate if the NAT Gateway is still needed (if all internet access is through VPC endpoints or Direct Connect, remove it entirely: -$32.85/month + remaining data processing). Review Direct Connect utilization — if 1 Gbps is underutilized, consider 500 Mbps hosted connection (-$109.50/month). Compress outbound DX data. Evaluate whether CloudFront can serve some of the 15 TB egress traffic at lower rates.

### B. Replace Direct Connect with VPN — ❌ Incorrect

- **VPN data transfer costs**: VPN data transfer is NOT free. Outbound data transfer over VPN is charged at standard internet egress rates ($0.09/GB for first 10 TB). For 15 TB outbound: (10 TB × $0.09) + (5 TB × $0.085) = $900 + $425 = **$1,325/month** — far more than DX outbound ($262.50/month).
- **VPN hourly charges**: $0.05/hour per VPN tunnel × 2 tunnels = $73/month per VPN connection.
- **Bandwidth**: VPN over internet is limited to ~1.25 Gbps per tunnel (and real-world throughput is often lower due to encryption overhead and internet variability). 8 TB/month inbound + 15 TB/month outbound requires sustained throughput — VPN may not deliver reliably.
- **Direct Connect justifies its cost**: DX port ($219/month) + DX data transfer ($262.50/month) = $481.50/month. VPN data transfer alone: $1,325/month. DX saves $843.50/month in data transfer for this traffic volume.

### C. Compress data + add second DX — ❌ Incorrect

- **Compression**: Valid technique that can reduce data transfer volume 50-80% (depending on data type). However:
  - Not all data is compressible (encrypted data, compressed media, binary formats).
  - Compression adds CPU overhead on both sides.
  - Compression alone doesn't address the highest-cost item: NAT Gateway charges for S3/DynamoDB traffic ($573/month).
- **Second Direct Connect**: A second DX connection adds $219/month in port charges + additional data transfer costs. This increases cost — the opposite of the goal. Redundancy is valid for reliability, but the question focuses on cost reduction.
- **Site-to-Site VPN provides backup**: VPN is already the backup for DX ($73/month, rarely used). Adding a second DX for redundancy while keeping VPN creates unnecessary redundancy at higher cost.

### D. Single AZ + remove TGW + VPN only — ❌ Incorrect

- **Single AZ = no availability**: Moving all workloads to one AZ saves cross-AZ charges ($60/month) but eliminates fault tolerance. If that AZ fails, all workloads are unavailable. For production systems, this is unacceptable — AZ failures, while rare, do occur.
  - RDS Multi-AZ specifically requires 2 AZs. Removing Multi-AZ saves cross-AZ replication (already free) but eliminates database HA.
- **VPN only**: As analyzed in Option B, VPN data transfer costs ($1,325/month outbound) far exceed Direct Connect ($262.50/month). Canceling DX increases costs by ~$843/month.
- **VPC Peering for all connections**: VPC Peering can replace TGW for VPC-to-VPC traffic (correct). But VPC Peering cannot connect to Direct Connect or VPN — you need TGW or a Virtual Private Gateway for hybrid connectivity.

## Recommendations

- **Always use Gateway VPC endpoints for S3 and DynamoDB**: They're free and eliminate NAT Gateway costs for these services. There is zero reason to route S3/DynamoDB traffic through a NAT Gateway.
- **NAT Gateway audit**: Check what traffic actually traverses the NAT Gateway. If only S3/DynamoDB, removing the NAT Gateway after adding Gateway endpoints saves the hourly charge. If other internet-bound traffic exists, keep the NAT Gateway but at a smaller volume.
- **Direct Connect cost tiers**: DX outbound pricing decreases with volume. At 100+ TB/month, DX is significantly cheaper per GB than internet. For smaller volumes (< 5 TB/month outbound), VPN may be cheaper if bandwidth needs are met.
- **Data transfer cost cheat sheet**:
  - Inbound (internet/DX → AWS): **Free**
  - Same AZ, private IP: **Free**
  - Same AZ, public/Elastic IP: **$0.01/GB each direction**
  - Cross-AZ: **$0.01/GB each direction**
  - Cross-Region: **$0.02/GB** (varies by pair)
  - Internet egress: **$0.09/GB** (first 10 TB)
  - DX egress: **$0.02/GB** (first 10 TB)
  - VPC Gateway endpoint: **Free**
  - VPC Interface endpoint: **$0.01/GB** + hourly
  - NAT Gateway: **$0.045/GB** + $0.045/hour
  - TGW: **$0.02/GB** + $0.05/hour per attachment
  - VPC Peering (same Region, same AZ): **Free**
  - VPC Peering (same Region, cross-AZ): **$0.01/GB**
- **Transit Gateway vs. VPC Peering**: Use TGW when VPC count > 5-10 (mesh complexity). Use Peering when VPC count is small and traffic patterns are simple.

## Relevant Links

- [S3 Gateway VPC Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
- [VPC Endpoint Pricing](https://aws.amazon.com/privatelink/pricing/)
- [Direct Connect Pricing](https://aws.amazon.com/directconnect/pricing/)
- [Data Transfer Pricing](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer)
- [Transit Gateway Pricing](https://aws.amazon.com/transit-gateway/pricing/)
- [VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [NAT Gateway Pricing](https://aws.amazon.com/vpc/pricing/)
