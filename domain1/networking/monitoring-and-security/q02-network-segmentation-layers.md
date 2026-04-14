# Q02: Network Segmentation and Security Layers

## Question

A company is designing a 3-tier web application architecture in a VPC. The tiers are: public-facing web servers (ALB + EC2), application servers (EC2), and database servers (RDS). The security team requires:
- Web servers accept HTTPS traffic only from the internet
- Application servers accept traffic only from web servers on port 8443
- Database servers accept traffic only from application servers on port 3306
- No tier should be able to initiate outbound connections to the internet except the web tier
- All inter-tier communication must be logged

Which combination of network controls should the solutions architect implement? **(Select THREE.)**

## Options

- **A.** Place each tier in a separate subnet tier: public subnets for web (with ALB), private subnets for application, and isolated private subnets for database. Use NAT Gateway in public subnets only for web tier outbound internet access.
- **B.** Configure security groups for each tier: Web SG allows HTTPS (443) from 0.0.0.0/0. App SG allows port 8443 from Web SG only. DB SG allows port 3306 from App SG only. No outbound rules beyond the required communication.
- **C.** Configure Network ACLs on each subnet tier as an additional layer: Web subnet NACL explicitly allows inbound 443 and denies all other inbound. App subnet NACL allows inbound 8443 only from web subnet CIDR. DB subnet NACL allows inbound 3306 only from app subnet CIDR.
- **D.** Enable VPC Flow Logs on all subnets, publishing to CloudWatch Logs and S3 for inter-tier communication logging and forensic analysis.
- **E.** Deploy all three tiers in the same subnet and use security groups alone for isolation.
- **F.** Use AWS Network Firewall between each tier for stateful packet inspection and logging.

## Answers

### A. Subnet segmentation — ✅ Correct

Subnet placement is the foundation of network segmentation:
- **Public subnets** (web/ALB): Route table with internet gateway route. Only the ALB needs to be internet-facing.
- **Private subnets** (application): No internet gateway route. Cannot be reached directly from the internet.
- **Isolated private subnets** (database): No NAT Gateway, no internet route. Completely isolated from outbound internet access.
- NAT Gateway in public subnets allows the web/app tier outbound access if needed. The database tier's subnet route table has **no NAT Gateway route**, preventing any outbound internet connectivity.

### B. Security groups — ✅ Correct

Security groups provide stateful, instance-level firewall rules:
- **SG referencing**: App SG referencing Web SG ensures only EC2 instances with the Web SG can reach port 8443 — this is more secure and dynamic than CIDR-based rules.
- **Implicit deny**: Security groups deny all inbound traffic by default. Only explicitly allowed ports/sources are permitted.
- **Stateful**: Return traffic is automatically allowed, simplifying rule management.
- Outbound rules can be restricted to only the required destinations (app tier → DB port 3306 only).

### D. VPC Flow Logs — ✅ Correct

VPC Flow Logs capture all network traffic metadata (accepted and rejected) for compliance and troubleshooting:
- Enable on all subnets to log inter-tier communication.
- Publish to both CloudWatch Logs (for real-time alerts) and S3 (for long-term forensic storage).
- Flow logs capture source/destination IPs, ports, protocols, byte counts, and accept/reject actions.

### C. Network ACLs — ❌ Incorrect

While NACLs provide an additional layer of defense, they are **stateless** (require explicit inbound AND outbound rules), use CIDR-based rules (not security group references), and are harder to manage than security groups for tier-based isolation. For this scenario, subnets + security groups provide sufficient segmentation. NACLs are useful as a broad subnet-level backstop but are not one of the three most important controls here.

### E. Same subnet — ❌ Incorrect

Placing all tiers in the same subnet eliminates subnet-based segmentation, making it impossible to apply different route tables (e.g., internet access for web only). All tiers would share the same NACL rules, and network isolation would depend entirely on security groups — a single layer of defense.

### F. Network Firewall between tiers — ❌ Incorrect

AWS Network Firewall provides deep packet inspection and is valuable at the VPC perimeter, but deploying it between every tier adds significant cost and complexity. For standard 3-tier applications, subnets, security groups, and NACLs provide sufficient isolation. Network Firewall is more appropriate for VPC ingress/egress inspection, not inter-tier traffic.

## Recommendations

- **Defense in depth**: Layered security — subnets (network segmentation) + security groups (instance-level) + NACLs (subnet-level) + flow logs (visibility).
- **Security group chaining**: Use SG-to-SG references (not CIDRs) for inter-tier rules — this automatically adapts when instances are added/removed.
- **Minimize outbound**: Restrict security group outbound rules to only required destinations and ports, not the default "allow all outbound."
- **RDS in isolated subnets**: RDS subnet groups should use subnets with no internet route for maximum database isolation.
- **Flow log costs**: CloudWatch Logs ingestion is more expensive than S3. For high-volume logging, prefer S3 with Athena queries.

## Relevant Links

- [VPC Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
