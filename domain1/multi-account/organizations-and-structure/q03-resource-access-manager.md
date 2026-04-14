# Q03: AWS Resource Access Manager Cross-Account Sharing

## Question

A company has a shared services account that manages a centralized VPC with a Transit Gateway, AWS Directory Service (Microsoft AD), and AWS License Manager configurations. The company needs 20 workload accounts to: use subnets from the shared VPC (without VPC peering), join instances to the shared Active Directory, and use shared license configurations for Microsoft SQL Server. All accounts are in the same AWS Organization.

Which approach should the solutions architect recommend?

## Options

- **A.** Use AWS Resource Access Manager (RAM) to share the VPC subnets, Transit Gateway, Directory Service directory, and License Manager configurations from the shared services account to the workload accounts. Enable RAM sharing with AWS Organizations for automatic trust.
- **B.** Deploy a VPN connection from each workload account's VPC to the shared services VPC. Create a trust relationship with the shared Active Directory from each workload account. Manually duplicate License Manager configurations in each account.
- **C.** Use VPC Peering to connect each workload account's VPC to the shared services VPC. Install Active Directory connectors in each workload account. Export and import License Manager configurations.
- **D.** Consolidate all workloads into the shared services account and use IAM roles to separate access between teams. This eliminates the need for cross-account resource sharing.

## Answers

### A. AWS Resource Access Manager — ✅ Correct

AWS RAM is purpose-built for sharing AWS resources across accounts within an Organization:
- **VPC Subnets**: RAM shares subnets to participant accounts. Workload accounts can launch resources directly into shared subnets without creating VPC Peering connections or managing separate VPCs. The subnet appears in the participant's VPC console.
- **Transit Gateway**: RAM shares the TGW so workload accounts can create attachments.
- **Directory Service**: RAM shares Managed Microsoft AD directories for cross-account domain joining.
- **License Manager**: RAM shares license configurations for centralized license tracking.
- When **RAM is integrated with Organizations**, shares are automatically accepted without manual invitation handling.

### B. VPN + manual duplication — ❌ Incorrect

Creating VPN connections to 20 accounts adds complexity, cost ($0.05/hr per VPN connection × 20), and latency. Trust relationships don't replace the need for directory sharing. Duplicating License Manager configurations defeats the purpose of centralized license management and leads to configuration drift.

### C. VPC Peering + AD connectors — ❌ Incorrect

VPC Peering creates 20 point-to-point connections (compared to RAM subnet sharing, which requires zero). AD Connectors redirect authentication requests but require separate setup per account. License Manager configurations cannot be exported/imported between accounts — RAM is the correct sharing mechanism.

### D. Single account consolidation — ❌ Incorrect

Consolidating all workloads into one account eliminates the security benefits of multi-account isolation (blast radius containment, separate IAM boundaries, independent billing). This contradicts AWS best practices for multi-account architecture. IAM roles within a single account don't provide the same isolation as separate accounts.

## Recommendations

- **RAM-shareable resources** include: VPC subnets, Transit Gateways, Route 53 Resolver rules, License Manager configs, Aurora DB clusters, CodeBuild projects, and more.
- Enable **RAM integration with Organizations** in the management account — this allows shares to be automatically accepted.
- Shared VPC subnets via RAM allow **participant accounts to launch resources** (EC2, RDS, Lambda) into the shared subnet, but they **cannot modify** the shared networking resources (route tables, NACLs, etc.).
- Use RAM organizational sharing when possible — it's simpler and more secure than account-ID-based sharing.

## Relevant Links

- [AWS Resource Access Manager](https://docs.aws.amazon.com/ram/latest/userguide/what-is.html)
- [Shareable Resources](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html)
- [VPC Subnet Sharing](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html)
- [Sharing Directory Service](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_directory_sharing.html)
