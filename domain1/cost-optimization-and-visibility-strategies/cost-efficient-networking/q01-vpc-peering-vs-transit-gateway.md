# Q01: VPC Peering vs. Transit Gateway Cost Tradeoff

## Question

A company has 30 VPCs across 3 AWS Regions (10 VPCs per Region). Currently, all VPCs within each Region communicate via full-mesh VPC Peering connections, resulting in 45 peering connections per Region (135 total). The company is adding 20 more VPCs over the next 12 months and needs cross-Region connectivity for 5 VPCs that run shared services. The networking team reports that managing peering connections is becoming operationally burdensome and new VPCs take days to fully interconnect.

Which networking architecture should the solutions architect recommend to optimize both cost and operational efficiency?

## Options

- **A.** Continue with full-mesh VPC Peering for all VPCs. Add new peering connections as new VPCs are created. Use VPC Peering across Regions for the 5 shared-services VPCs.
- **B.** Deploy an AWS Transit Gateway in each Region. Attach all VPCs to the Regional Transit Gateway. Use Transit Gateway peering for cross-Region connectivity between the 3 Transit Gateways. 
- **C.** Deploy a Transit Gateway in one Region and route all inter-VPC traffic through that single Transit Gateway using VPN connections from the other two Regions.
- **D.** Use AWS PrivateLink for all inter-VPC communication instead of peering or Transit Gateway. Create VPC endpoint services in each VPC for the applications that need to communicate.

## Answers

### B. Regional Transit Gateways with TGW peering — ✅ Correct

Transit Gateway simplifies multi-VPC networking to a hub-and-spoke model:
- Each VPC has **one attachment** to the Regional TGW instead of N-1 peering connections.
- Adding a new VPC requires only **one** TGW attachment, not 9+ peering connections.
- **Transit Gateway peering** provides native cross-Region connectivity for shared services.
- **Route tables on TGW** provide centralized routing control and network segmentation.

**Cost consideration**: TGW charges per attachment ($0.05/hr ≈ $36/month) and per GB of data processed ($0.02/GB). For 30+ VPCs, the operational savings (reduced peering management, faster VPC onboarding) typically outweigh the additional data processing cost compared to free VPC Peering data transfer.

### A. Continue full-mesh VPC Peering — ❌ Incorrect

Full-mesh peering scales as N×(N-1)/2. With 50 VPCs per Region, that would be 1,225 peering connections per Region — operationally unmanageable. VPC Peering has no data transfer cost (within same AZ) or lower cross-AZ cost, but the operational burden of managing thousands of connections, route tables, and security groups outweighs the savings. VPC Peering also doesn't support transitive routing.

### C. Single-Region TGW with VPN — ❌ Incorrect

Routing all traffic through a single Region adds latency for inter-VPC communication in the other two Regions. VPN connections to the central TGW have bandwidth limitations (1.25 Gbps per tunnel) and introduce encryption overhead. TGW peering is the correct approach for cross-Region connectivity.

### D. AWS PrivateLink for all communication — ❌ Incorrect

PrivateLink is designed for specific service-to-consumer connectivity (e.g., exposing a single application endpoint). It is not a replacement for general network routing between VPCs. Each PrivateLink endpoint would need to be configured per-service, per-VPC — far more complex than TGW for general inter-VPC connectivity.

## Recommendations

- **VPC Peering** is most cost-effective for small numbers of VPCs (< 10) with low data volume — no hourly charge, free same-AZ transfers.
- **Transit Gateway** becomes operationally superior at ~10+ VPCs due to simplified hub-and-spoke management.
- Factor in **operational cost** (engineer time managing peering connections) against TGW's data processing charges.
- Use **TGW route table segmentation** to control which VPCs can communicate — equivalent to multiple peering topologies but managed centrally.
- For cross-Region, use **TGW peering** (not VPN) for native, high-bandwidth, encrypted connectivity.

## Relevant Links

- [AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)
- [Transit Gateway Pricing](https://aws.amazon.com/transit-gateway/pricing/)
- [VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [Transit Gateway Peering](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html)
