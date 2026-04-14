# Q01: Transit Gateway for Multi-VPC Multi-Region Connectivity

## Question

A company operates 25 VPCs across us-east-1, eu-west-1, and ap-southeast-1. Each Region has VPCs for production, staging, and shared services. The requirements are:
- Production VPCs in each Region must communicate with each other (cross-Region)
- Staging VPCs must NOT communicate with production VPCs in any Region
- All VPCs must access shared services (DNS, logging) in their local Region
- On-premises data center must connect to all production VPCs via Direct Connect in us-east-1

Which networking architecture should the solutions architect implement?

## Options

- **A.** Deploy a Transit Gateway in each Region. Attach all VPCs to their Regional TGW. Create TGW route tables for network segmentation: a Production route table and a Staging route table. Use Transit Gateway peering between Regional TGWs for cross-Region production connectivity. Connect the on-premises data center via Direct Connect to the us-east-1 TGW with a Transit Virtual Interface (Transit VIF).
- **B.** Create full-mesh VPC Peering connections between all production VPCs within and across Regions. Create separate mesh for staging VPCs. Attach the on-premises network via VPN to each production VPC individually.
- **C.** Deploy a single Transit Gateway in us-east-1 and attach all 25 VPCs from all Regions using VPN connections. Route all traffic through the central TGW.
- **D.** Use AWS PrivateLink to connect all VPCs. Create VPC endpoint services in each production VPC and VPC endpoints in the consuming VPCs. Connect on-premises via Direct Connect to a single VPC.

## Answers

### A. Regional TGWs with route table segmentation + TGW peering — ✅ Correct

This architecture correctly addresses all requirements:
- **Regional TGWs**: Each Region gets its own TGW — VPCs attach locally for optimal latency and cost.
- **Route table segmentation**: TGW route tables isolate production and staging traffic. Production route table has routes to production VPCs and shared services. Staging route table has routes to staging VPCs and shared services only. No route between production and staging route tables ensures isolation.
- **TGW peering**: Connects Regional TGWs for cross-Region production traffic. Static routes on peered TGWs control which traffic flows cross-Region.
- **Transit VIF on Direct Connect**: A Transit VIF connects Direct Connect to TGW (unlike a Private VIF which connects to a single VPC). This gives on-premises access to all VPCs attached to the TGW, controlled by route tables.

### B. Full-mesh VPC Peering — ❌ Incorrect

With 25 VPCs, full-mesh peering creates hundreds of connections — unmanageable operationally. Cross-Region VPC Peering works but each pair needs a separate connection. VPN per production VPC from on-premises is complex and doesn't scale. VPC Peering also doesn't support transitive routing — traffic from on-premises cannot transit through one peered VPC to reach another.

### C. Single central TGW — ❌ Incorrect

TGW attachments are Regional — you cannot attach a VPC in eu-west-1 directly to a TGW in us-east-1. VPN connections from other Regions to a central TGW add latency, cost, and bandwidth limitations (1.25 Gbps per VPN tunnel). This forces all inter-VPC traffic from other Regions through us-east-1, even for local intra-Region communication.

### D. PrivateLink for all connectivity — ❌ Incorrect

PrivateLink is designed for specific service endpoints (e.g., exposing an API), not for general network routing between VPCs. It requires a Network Load Balancer in every service-providing VPC and explicit endpoint configuration per consumer VPC. This doesn't scale for general inter-VPC connectivity and doesn't support on-premises integration the way TGW + Direct Connect does.

## Recommendations

- **TGW route tables** are the primary mechanism for network segmentation — create separate route tables for each security domain (prod, staging, shared).
- Use **TGW peering** (not VPN) for cross-Region TGW connectivity — it's encrypted, uses AWS backbone, and supports higher bandwidth.
- **Transit VIF** vs **Private VIF**: Transit VIF connects DX to TGW (for many VPCs); Private VIF connects DX to a single VPC (or Direct Connect Gateway for multiple VPCs without TGW).
- Enable **TGW Network Manager** for centralized visibility of your global network topology.
- Consider **TGW Connect** attachments for SD-WAN integration with third-party appliances.

## Relevant Links

- [Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)
- [Transit Gateway Route Tables](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
- [Transit Gateway Peering](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html)
- [Transit Virtual Interface](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)
