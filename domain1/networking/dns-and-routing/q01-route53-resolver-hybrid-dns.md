# Q01: Route 53 Resolver for Hybrid DNS

## Question

A company has a hybrid environment with an on-premises Active Directory DNS server (`corp.example.com`) and AWS VPCs with Route 53 private hosted zones (`aws.example.com`). The requirements are:
- EC2 instances in AWS must resolve on-premises DNS names (e.g., `ldap.corp.example.com`)
- On-premises servers must resolve AWS private hosted zone names (e.g., `api.aws.example.com`)
- DNS queries must travel over the Direct Connect connection, not the internet
- The solution must work across 3 AWS Regions

Which DNS architecture should the solutions architect implement?

## Options

- **A.** Create Route 53 Resolver inbound endpoints in each VPC for on-premises-to-AWS DNS resolution. Create Route 53 Resolver outbound endpoints with forwarding rules for `corp.example.com` to forward queries to on-premises DNS servers. Share forwarding rules across accounts using AWS RAM.
- **B.** Deploy BIND DNS servers on EC2 instances in each VPC. Configure them as forwarders to both the on-premises DNS server and Route 53 Resolver. Point on-premises DNS to the BIND servers for AWS zone resolution.
- **C.** Configure the on-premises DNS server as a secondary zone for `aws.example.com`. Configure Route 53 to allow zone transfers to the on-premises DNS server. Set up conditional forwarding on Route 53 for `corp.example.com`.
- **D.** Use Route 53 public hosted zones instead of private hosted zones. On-premises servers can resolve public DNS names over the internet. EC2 instances use the on-premises DNS server as a custom DNS resolver.

## Answers

### A. Route 53 Resolver inbound + outbound endpoints — ✅ Correct

Route 53 Resolver endpoints provide native hybrid DNS resolution:

**Outbound endpoints** (AWS→on-premises):
- Create an outbound endpoint in the VPC with ENIs in at least 2 AZs.
- Create **forwarding rules** that say "for `corp.example.com`, forward queries to 10.0.1.53" (on-premises DNS IP).
- EC2 instances in the VPC automatically use the outbound endpoint for matching domains.
- Forwarding rules can be **shared via AWS RAM** to other accounts and VPCs.

**Inbound endpoints** (on-premises→AWS):
- Create an inbound endpoint in the VPC with ENIs that have private IP addresses.
- Configure on-premises DNS to forward `aws.example.com` queries to the inbound endpoint IPs (reachable over Direct Connect).
- Route 53 resolves the private hosted zone and returns the answer.

All DNS traffic flows over Direct Connect (private network), meeting the security requirement.

### B. BIND DNS servers on EC2 — ❌ Incorrect

Self-managed DNS servers require: EC2 instance management, patching, high availability configuration, scaling, and DNS software maintenance. Route 53 Resolver endpoints are a fully managed service that provides the same functionality without operational overhead. BIND servers were the pre-Resolver approach and are no longer recommended.

### C. Zone transfers — ❌ Incorrect

Route 53 private hosted zones do **not support zone transfers (AXFR/IXFR)**. This architecture fundamentally won't work. Even if it could, zone transfers would require replication scheduling, which adds latency to DNS record changes and complexity to the architecture.

### D. Public hosted zones — ❌ Incorrect

Using public hosted zones would expose internal DNS records to the internet — a security risk for internal APIs and services. On-premises resolution over the internet also adds latency and doesn't meet the "over Direct Connect" requirement.

## Recommendations

- **Outbound endpoints**: Use forwarding rules for on-premises domains. Rules are processed in the VPC they're associated with.
- **Inbound endpoints**: Provide IP addresses that on-premises DNS can forward to. Place in subnets reachable from on-premises via DX/VPN.
- **RAM sharing**: Share forwarding rules across accounts so all VPCs use the same outbound endpoint and rules — avoids duplicating endpoints per VPC.
- Each endpoint requires **ENIs in at least 2 AZs** for high availability.
- For multi-Region, create **separate Resolver endpoints per Region** but share forwarding rules organization-wide.

## Relevant Links

- [Route 53 Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)
- [Inbound Endpoints](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-inbound-queries.html)
- [Outbound Endpoints](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html)
- [Sharing Forwarding Rules with RAM](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-rules-managing.html#resolver-rules-managing-sharing)
