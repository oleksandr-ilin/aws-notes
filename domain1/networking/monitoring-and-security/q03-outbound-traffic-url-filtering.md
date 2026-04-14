# Q03: Outbound Traffic Control with Forward Proxy

## Question

A regulated healthcare company runs EC2 instances in private subnets that need to download software updates from specific third-party vendor URLs. The security team requires that outbound internet access be restricted to an **allowlist of specific URLs** (not IP addresses, since vendors use CDNs with dynamic IPs). All other outbound internet traffic must be blocked. The solution must log all outbound web requests for compliance auditing.

Which approach should the solutions architect recommend?

## Options

- **A.** Configure security groups on the EC2 instances to allow outbound HTTPS (port 443) only to the vendors' IP addresses. Block all other outbound traffic.
- **B.** Deploy a forward web proxy (e.g., Squid) on EC2 instances in a public subnet. Route all outbound HTTP/HTTPS traffic from the private subnet through the proxy. Configure the proxy with URL-based allowlists and enable access logging.
- **C.** Use an AWS Network Firewall with domain-based filtering rules. Create stateful rules that allow outbound HTTPS traffic only to the specified vendor FQDNs. Enable firewall logging to S3 for compliance.
- **D.** Configure the NAT Gateway's security group to allow outbound traffic only to the vendor URLs.

## Answers

### C. AWS Network Firewall with domain filtering — ✅ Correct

AWS Network Firewall provides managed, scalable URL/domain-based filtering:
- **Domain-based stateful rules**: Allow outbound HTTPS to specific FQDNs (e.g., `updates.vendor.com`) while denying all other outbound traffic. Works with dynamic IPs because filtering is based on the domain in the TLS SNI (Server Name Indication) field.
- **Managed service**: No EC2 instances to manage, patch, or scale. Network Firewall scales automatically with traffic.
- **Centralized logging**: Flow logs and alert logs can be published to S3, CloudWatch Logs, or Kinesis Data Firehose for compliance.
- **Architecture**: Network Firewall is deployed in its own subnet. Route tables direct traffic from private subnets → firewall subnet → NAT Gateway → internet.

### B. Forward web proxy (Squid) — ❌ Incorrect

A Squid forward proxy provides URL-based filtering and logging, but it requires managing EC2 instances: provisioning, patching, high availability configuration (across AZs), scaling, and monitoring. Before AWS Network Firewall existed, this was the standard approach. Now, Network Firewall provides the same domain-filtering capability as a managed service. Use a proxy server only when you need features beyond Network Firewall (e.g., request/response body modification, caching).

### A. Security group IP allowlisting — ❌ Incorrect

Security groups work with **IP addresses and CIDR ranges**, not domain names/URLs. Vendors using CDNs (CloudFront, Akamai) have thousands of IP addresses that change frequently. Maintaining an IP allowlist would be operationally impossible and would break when vendors change their CDN configurations.

### D. NAT Gateway security group — ❌ Incorrect

NAT Gateways do **not** have security groups. NAT Gateways are managed resources that pass traffic without application-layer inspection. They cannot filter by URL or domain name. Traffic control must be applied before traffic reaches the NAT Gateway (using Network Firewall, proxy, or NACLs).

## Recommendations

- **AWS Network Firewall** is the managed solution for domain-based egress filtering — preferred over self-managed proxies.
- Network Firewall evaluates the **TLS SNI field** for HTTPS domain filtering — it doesn't decrypt traffic, maintaining end-to-end encryption.
- **Route table architecture**: Private subnet → Firewall subnet → NAT Gateway subnet → IGW. The firewall inspects traffic before it reaches the NAT Gateway.
- For simple use cases (allowing specific AWS service endpoints), **VPC endpoints** are simpler and cheaper than Network Firewall.
- Consider **AWS Firewall Manager** to deploy Network Firewall policies consistently across multiple accounts and VPCs.

## Relevant Links

- [AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html)
- [Stateful Domain Filtering](https://docs.aws.amazon.com/network-firewall/latest/developerguide/stateful-rule-groups-domain-names.html)
- [Network Firewall Logging](https://docs.aws.amazon.com/network-firewall/latest/developerguide/firewall-logging.html)
- [Deployment Architectures](https://docs.aws.amazon.com/network-firewall/latest/developerguide/architectures.html)
