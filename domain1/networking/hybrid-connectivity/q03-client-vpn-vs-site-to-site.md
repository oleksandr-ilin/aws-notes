# Q03: Client VPN vs. Site-to-Site VPN

## Question

A company has two connectivity requirements:
1. **Remote workers** (500 employees) need secure access to internal applications running in a VPC from their laptops over the internet. The solution must support MFA through the company's existing SAML 2.0 identity provider (Okta).
2. **On-premises data center** needs a persistent, always-on tunnel to the VPC for application-to-application communication with consistent throughput.

Which combination of VPN solutions should the solutions architect recommend?

## Options

- **A.** Use AWS Client VPN for both requirements. Provision a Client VPN endpoint in the VPC for remote workers and configure a separate Client VPN endpoint for the data center connection.
- **B.** Use AWS Client VPN for remote worker access with SAML 2.0 authentication via Okta. Use AWS Site-to-Site VPN for the persistent data center connection with BGP routing.
- **C.** Use AWS Site-to-Site VPN for both requirements. Employees install VPN client software that connects to the VPN gateway, and the data center uses the same gateway with a separate tunnel.
- **D.** Deploy a third-party VPN appliance on an EC2 instance in the VPC. Configure OpenVPN for remote workers and IPSec for the data center connection.

## Answers

### B. Client VPN (remote workers) + Site-to-Site VPN (data center) — ✅ Correct

Each VPN type is designed for a different use case:

**AWS Client VPN** (for remote workers):
- Managed OpenVPN-based service for **user-to-VPC** connectivity.
- Supports **SAML 2.0 federation** (Okta, Azure AD) with MFA.
- Split tunnel or full tunnel options — users install an AWS or OpenVPN-compatible client.
- Scales to thousands of concurrent users automatically.
- Per-user authorization rules for fine-grained access control.

**AWS Site-to-Site VPN** (for data center):
- IPSec VPN for **network-to-VPC** connectivity.
- Persistent, always-on tunnels between the customer gateway (on-premises router) and AWS VPN gateway.
- **BGP routing** for dynamic route exchange and failover.
- Up to 1.25 Gbps per tunnel; use ECMP with Transit Gateway for higher aggregate throughput.

### A. Client VPN for both — ❌ Incorrect

Client VPN is designed for individual user connections, not site-to-site network connectivity. It would require running a VPN client on a router or server in the data center, which is not how Client VPN is designed. It also doesn't support BGP routing or persistent site-to-site tunnels.

### C. Site-to-Site VPN for both — ❌ Incorrect

Site-to-Site VPN requires a customer gateway device (hardware or software router) at each endpoint. It's not designed for individual laptop connections — employees would each need to be on a network behind a customer gateway. Site-to-Site VPN also doesn't natively support SAML-based authentication — it uses pre-shared keys or certificates for tunnel authentication.

### D. Third-party appliance — ❌ Incorrect

A self-managed VPN appliance introduces operational overhead: EC2 instance management, patching, high availability configuration, scaling, and licensing costs. AWS Client VPN and Site-to-Site VPN are managed services that handle availability, scaling, and maintenance. Only use third-party appliances when AWS-native solutions lack required features.

## Recommendations

- **Client VPN** for user-to-VPC (remote work, developer access, contractor access).
- **Site-to-Site VPN** for network-to-VPC (branch offices, data centers, partner networks).
- For Client VPN, enable **split tunneling** to route only VPC-destined traffic through the VPN, reducing bandwidth costs.
- For Site-to-Site VPN with higher throughput needs, use **ECMP** (Equal-Cost Multi-Path) with Transit Gateway — supports up to 50 tunnels for aggregate throughput.
- Enable **VPN CloudWatch metrics** to monitor tunnel status, data transferred, and connection count.

## Relevant Links

- [AWS Client VPN](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/what-is.html)
- [AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html)
- [Client VPN SAML Authentication](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/client-authentication.html#federated-authentication)
- [Site-to-Site VPN Tunnel Options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNTunnels.html)
