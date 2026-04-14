# Q01: Direct Connect with VPN Backup

## Question

A financial services company requires a dedicated, private connection from its on-premises data center to AWS with the following requirements: minimum 1 Gbps sustained bandwidth, encryption in transit for compliance, connection to VPCs in 3 AWS Regions (us-east-1, eu-west-1, ap-southeast-1), maximum 4 hours RTO if the primary connection fails, and the backup connection must be available immediately without requiring new hardware installation.

Which hybrid connectivity architecture should the solutions architect recommend?

## Options

- **A.** Provision a 1 Gbps AWS Direct Connect dedicated connection at a DX location near the data center. Create a Direct Connect Gateway associated with virtual private gateways in all 3 Regions. Enable MACsec encryption on the Direct Connect connection. Provision an AWS Site-to-Site VPN connection over the internet as a backup path to the same virtual private gateways.
- **B.** Provision two 1 Gbps Direct Connect connections at the same DX location for redundancy. Create private virtual interfaces for each VPC in each Region. Use no encryption as Direct Connect is inherently private.
- **C.** Configure AWS Site-to-Site VPN connections to VPCs in all 3 Regions over the public internet. Enable VPN encryption for compliance. Provision a backup VPN connection on a separate internet circuit.
- **D.** Provision a 1 Gbps Direct Connect connection. Create a Site-to-Site VPN over the Direct Connect connection (public VIF) for encryption. Use the same VPN path for all 3 Regions via a Transit Gateway.

## Answers

### A. Direct Connect + DX Gateway + MACsec + VPN backup — ✅ Correct

This architecture addresses all requirements:
- **1 Gbps DX dedicated connection**: Provides sustained, predictable bandwidth with low latency.
- **Direct Connect Gateway**: A single DX Gateway connects the Direct Connect connection to virtual private gateways (or Transit Gateways) in **multiple Regions** without needing separate DX connections per Region.
- **MACsec encryption**: Layer 2 encryption on the DX connection provides in-transit encryption without VPN overhead. MACsec is available on 10 Gbps and 100 Gbps connections (note: not available on 1 Gbps, so a VPN over DX public VIF may be needed instead — see recommendations).
- **VPN backup over internet**: Available immediately (no hardware provisioning), provides encrypted connectivity, and can failover using BGP routing. Meets the 4-hour RTO with automated BGP failover.

### B. Dual Direct Connect, no encryption — ❌ Incorrect

Direct Connect is private but **not encrypted by default** — data crosses the physical connection in the clear. For a financial services company's compliance requirements, encryption in transit is mandatory. Also, two connections at the same DX location don't protect against a location-level failure (e.g., fiber cut to the facility).

### C. VPN only — ❌ Incorrect

Site-to-Site VPN provides encryption but is limited to **1.25 Gbps per tunnel** and Subject to internet path variability (jitter, latency spikes). It cannot reliably provide the 1 Gbps sustained bandwidth requirement. For financial services with strict latency and bandwidth needs, a VPN alone is insufficient.

### D. VPN over DX only — ❌ Incorrect

Running a VPN over Direct Connect (using a public VIF) provides encryption and private bandwidth, but using a **single DX connection with no backup path** fails the resilience requirement. If the DX connection fails, there's no backup and the 4-hour RTO cannot be met.

## Recommendations

- **MACsec** is only available on 10+ Gbps connections. For 1 Gbps DX, use a **Site-to-Site VPN over DX** (using a public VIF) for encryption instead.
- For maximum DX resilience, provision connections at **two different DX locations** — this protects against location-level failures.
- **DX Gateway** supports up to 10 virtual private gateways across Regions (or Transit Gateways) — use it for multi-Region connectivity.
- VPN backup should use a **separate internet path** (different ISP) from the DX location for true path diversity.
- Consider **DX SiteLink** for site-to-site traffic that routes through the AWS network without entering a VPC.

## Relevant Links

- [AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)
- [Direct Connect Gateway](https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-gateways.html)
- [Direct Connect Resilience](https://docs.aws.amazon.com/directconnect/latest/UserGuide/resilency_toolkit.html)
- [MACsec on Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/MACsec.html)
- [VPN as Backup to Direct Connect](https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/aws-direct-connect-vpn.html)
