# Q03: Network Firewall vs. Security Groups vs. NACLs

## Question

A company needs to implement network security controls at different layers. They need to understand which AWS service is appropriate for each requirement:

1. Block all traffic from a specific list of known malicious IP addresses at the VPC perimeter
2. Allow only an application's EC2 instances to communicate with an RDS database on port 5432
3. Inspect outbound HTTPS traffic and block connections to known command-and-control (C2) domains
4. Provide stateless deny rules at the subnet level as a broad safety net

Which mapping of requirements to AWS network security services is correct?

## Options

- **A.** 1: Network ACLs. 2: Security Groups. 3: AWS Network Firewall. 4: Network ACLs.
- **B.** 1: Security Groups. 2: Security Groups. 3: Security Groups. 4: Network ACLs.
- **C.** 1: AWS Network Firewall. 2: Security Groups. 3: AWS Network Firewall. 4: Network ACLs.
- **D.** 1: AWS WAF. 2: Network ACLs. 3: AWS Network Firewall. 4: Security Groups.

## Answers

### C. Network Firewall (1,3) + Security Groups (2) + NACLs (4) — ✅ Correct

Each service operates at a different layer with different capabilities:

**Requirement 1 — AWS Network Firewall** (block malicious IPs at VPC perimeter):
Network Firewall can evaluate traffic at the VPC level using IP-based stateful rules with threat intelligence feeds. While NACLs can also block IPs, they have a 20-rule-per-direction limit (can be increased to 40), making them impractical for large malicious IP lists. Network Firewall supports thousands of rules via rule groups and integrates with managed threat intelligence.

**Requirement 2 — Security Groups** (application-to-database access):
Security groups provide stateful, instance-level access control. The RDS security group references the application's security group as the source, allowing only those specific instances to connect on port 5432. This is the correct tool for inter-tier access control.

**Requirement 3 — AWS Network Firewall** (HTTPS domain inspection):
Network Firewall can perform TLS SNI (Server Name Indication) inspection on outbound HTTPS traffic, detecting and blocking connections to specific domains (C2 servers) without decrypting the traffic. Neither security groups nor NACLs can inspect domain names — they only work with IPs and ports.

**Requirement 4 — Network ACLs** (stateless subnet-level deny rules):
NACLs provide stateless rules at the subnet level. They evaluate every inbound and outbound packet independently (no connection tracking). They're ideal for broad deny rules as a safety net, complementing the stateful rules of security groups.

### A. NACLs for malicious IPs — ❌ Incorrect

NACLs have a practical limit on the number of rules (20 default, up to 40). For large malicious IP lists (hundreds or thousands of IPs), NACLs are insufficient. Network Firewall can handle thousands of IP-based rules efficiently.

### B. Security Groups for everything — ❌ Incorrect

Security groups cannot inspect domain names (requirement 3) and cannot be applied at the subnet level (requirement 4). They are instance-level, stateful firewalls — excellent for inter-resource access control but not designed for VPC perimeter security or deep packet inspection.

### D. WAF for IP blocking — ❌ Incorrect

WAF operates at the application layer (Layer 7) and only protects HTTP/HTTPS resources (ALB, CloudFront, API Gateway). It cannot block non-HTTP traffic or protect at the VPC level. For arbitrary IP blocking at the VPC level, Network Firewall is the correct choice.

## Recommendations

- **Defense in depth layers**: Network Firewall (VPC perimeter) → NACLs (subnet level) → Security Groups (instance level).
- **Security Groups**: Stateful, referencing other SGs by ID. Best for: inter-resource access control within VPC.
- **NACLs**: Stateless, CIDR-based. Best for: broad deny rules, subnet isolation as a safety net.
- **Network Firewall**: Stateful, supports domain filtering, IDS/IPS (Suricata rules), and IP threat intelligence. Best for: VPC ingress/egress inspection, advanced threat protection.
- When in doubt between NACLs and Network Firewall for IP blocking: NACLs for small, static lists (<20 IPs); Network Firewall for large, dynamic lists.

## Relevant Links

- [AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/what-is-aws-network-firewall.html)
- [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [Network Firewall Rule Groups](https://docs.aws.amazon.com/network-firewall/latest/developerguide/rule-groups.html)
