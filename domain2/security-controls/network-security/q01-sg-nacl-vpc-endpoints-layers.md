# Q01: Network Security Layers — Security Groups, NACLs, and VPC Endpoints

## Question

A company is deploying a three-tier application (web → application → database) in a single VPC. Requirements:
- The web tier (ALB) must accept HTTPS traffic from the internet only
- The application tier (ECS Fargate tasks) must accept traffic only from the ALB, not directly from the internet
- The database tier (RDS Aurora) must accept connections only from the application tier on port 5432
- The application tier must call the S3 API and DynamoDB API without traffic leaving the VPC
- All tiers must block SSH/RDP access from any source
- An additional outbound restriction: the database subnet must not be able to initiate any outbound internet traffic

Which network security configuration should the solutions architect implement?

## Options

- **A.** Create 3 subnet tiers (public, private-app, private-db). Configure the ALB security group to allow inbound 443 from 0.0.0.0/0. Configure the app SG to allow inbound traffic from the ALB SG only (SG reference). Configure the DB SG to allow inbound 5432 from the app SG only. Add a NACL on the DB subnet denying all outbound to 0.0.0.0/0 (except ephemeral port responses). Create a Gateway VPC Endpoint for S3 and an Interface VPC Endpoint for DynamoDB. Add route table entries for the S3 gateway endpoint in the app subnet's route table. Do not add a NAT Gateway route to the DB subnet route table.
- **B.** Put all resources in a single public subnet. Use security groups to restrict traffic between tiers. Use IAM policies on the ECS task role to restrict S3 and DynamoDB access. Use a NAT Gateway for all outbound traffic.
- **C.** Create 3 subnets but use only NACLs for all access control (no security groups). Configure NACL rules for each subnet to allow/deny specific port ranges. Use S3 bucket policies to restrict access to the VPC CIDR range.
- **D.** Create 3 subnets with security groups referencing each other. Use AWS Network Firewall in the VPC to inspect all traffic between tiers. Create PrivateLink endpoints for both S3 and DynamoDB. Block all outbound with a default-deny NACL on all subnets.

## Answers

### A. Tiered subnets + SG chaining + NACL outbound deny + VPC endpoints — ✅ Correct

This implements defense in depth correctly:
- **Subnet tiers**: Public subnet (ALB), private-app (Fargate), private-db (Aurora). Only the public subnet has an Internet Gateway route.
- **Security group chaining**: App SG allows inbound from ALB SG (not CIDR — SG references automatically track ALB IPs). DB SG allows inbound 5432 from app SG only. This ensures only the correct tier can communicate with the next. No SSH/RDP rules are added, so those ports are denied by default (SGs are deny-by-default).
- **NACL on DB subnet**: NACLs are stateless — an explicit deny outbound to 0.0.0.0/0 blocks the database from initiating internet connections. Allow outbound to VPC CIDR (for responses to app tier) and ephemeral ports. This is a belt-and-suspenders control alongside the route table having no NAT/IGW route.
- **VPC endpoints**: Gateway endpoint for S3 (free, route-table-based), Interface endpoint for DynamoDB (PrivateLink-based, per-hour + per-GB). Traffic to S3 and DynamoDB stays within the VPC — no internet traversal.
- **No NAT Gateway route in DB subnet**: Even without the NACL, the DB subnet can't reach the internet because there's no route. The NACL adds explicit enforcement.

### B. Single public subnet — ❌ Incorrect

Placing all resources in a public subnet means every resource has a potential internet-reachable path. Security groups alone provide instance-level protection, but any misconfiguration exposes the database directly. This violates network segmentation best practices. NAT Gateway for outbound means even the DB can reach the internet. IAM policies control API authorization, not network-level traffic flow.

### C. NACLs only (no security groups) — ❌ Incorrect

NACLs are stateless and rule-order-dependent — managing complex traffic flows with NACLs alone is error-prone. NACLs cannot reference security groups (only CIDRs), so dynamic scaling (new Fargate task IPs) requires constant NACL updates. Security groups are stateful and automatically handle return traffic — relying only on NACLs means explicitly managing ephemeral port ranges for every flow. S3 bucket policies restricting by VPC CIDR don't prevent traffic from traversing the internet — VPC endpoints are needed for that.

### D. Network Firewall + default-deny all NACLs — ❌ Over-engineered

AWS Network Firewall for inter-tier inspection is overkill for standard three-tier traffic control — SGs handle this natively. Default-deny NACLs on ALL subnets (including the web tier) would block ALB internet traffic unless explicitly permitted, adding significant NACL rule complexity. PrivateLink (Interface Endpoints) for both S3 and DynamoDB works but is more expensive than using the free Gateway Endpoint for S3. The added cost and complexity aren't justified by the requirements.

## Recommendations

- **Security groups** are the primary network control — use SG-to-SG references (not CIDR) to automatically handle IP changes from scaling.
- **NACLs** are a secondary layer — use them for broad subnet-level deny rules (e.g., block outbound internet from DB subnet), not for fine-grained application controls.
- **Gateway VPC Endpoints** (S3, DynamoDB) are free and should be used whenever services access these. Interface endpoints cost money but keep traffic private.
- **Default deny**: SGs deny all inbound by default. NACLs default-allow unless changed. Explicitly deny internet outbound on sensitive subnets via NACL and route table (no NAT route).
- **Never add SSH/RDP security group rules** — use Systems Manager Session Manager for administrative access instead.

## Relevant Links

- [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [Gateway Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
