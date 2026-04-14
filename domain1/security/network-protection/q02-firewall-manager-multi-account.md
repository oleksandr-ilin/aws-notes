# Q02: AWS Firewall Manager for Multi-Account Security

## Question

A company with 60 AWS accounts needs to enforce consistent firewall policies across all accounts. The requirements are:
- All VPCs must have AWS Network Firewall deployed with standardized egress filtering rules
- All public-facing ALBs must have AWS WAF with the same managed rule groups applied
- All security groups across the organization must be audited for overly permissive rules (e.g., 0.0.0.0/0 on SSH)
- New accounts added to the Organization must automatically receive all firewall policies

Which service should the solutions architect use to manage these requirements centrally?

## Options

- **A.** Deploy AWS Firewall Manager as the delegated administrator in the security account. Create Firewall Manager policies for: Network Firewall (egress), WAF (ALB protection), and Security Group Audit (compliance checking). Associate policies with the Organization root or specific OUs.
- **B.** Create CloudFormation StackSets in the management account to deploy Network Firewall, WAF rules, and Config rules for security group auditing in all accounts. Use CloudFormation drift detection to monitor compliance.
- **C.** Deploy Terraform modules via a CI/CD pipeline to provision Network Firewall, WAF, and security group monitoring in each account individually. Use Terraform state to track deployments.
- **D.** Manually deploy Network Firewall and WAF in each account using the AWS Console. Use AWS Config rules to audit security groups. Run monthly compliance reviews.

## Answers

### A. AWS Firewall Manager — ✅ Correct

Firewall Manager is purpose-built for centralized network security management across Organizations:

- **Network Firewall policy**: Deploys AWS Network Firewall with standardized rule groups across all VPCs in member accounts. Firewall Manager creates the firewall, firewall policy, and updates route tables automatically.
- **WAF policy**: Deploys WAF web ACLs with managed rule groups to all ALBs/CloudFront distributions across accounts. New ALBs automatically receive the WAF policy.
- **Security Group audit policy**: Identifies non-compliant security groups (e.g., unrestricted SSH) and optionally auto-remediates by removing the offending rules.
- **Automatic enrollment**: New accounts added to the Organization (or moved into an OU with a policy) automatically receive all applicable Firewall Manager policies.
- **Centralized management**: All policies managed from a single delegated admin account.

### B. CloudFormation StackSets — ❌ Incorrect

StackSets can deploy resources across accounts but don't provide the automatic policy enforcement that Firewall Manager does. StackSets won't automatically detect new ALBs and apply WAF rules, won't audit existing security groups, and require manual drift detection. StackSets deploy infrastructure; Firewall Manager enforces ongoing compliance.

### C. Terraform pipeline — ❌ Incorrect

Terraform provides infrastructure deployment but requires significant custom development for: discovering new resources (ALBs), detecting configuration drift (security group changes), and enrolling new accounts. Firewall Manager provides these capabilities natively. Terraform is better suited for provisioning infrastructure within accounts, not for cross-account security policy enforcement.

### D. Manual deployment — ❌ Incorrect

Manual deployment across 60 accounts is error-prone, time-consuming, and doesn't scale. Missing one account or ALB creates a security gap. It doesn't address the requirement for automatic enrollment of new accounts.

## Recommendations

- **Firewall Manager** requires: AWS Organizations with all features enabled, AWS Config enabled in all accounts, and a delegated administrator account configured.
- Firewall Manager supports **four policy types**: WAF, Shield Advanced, Network Firewall, and Security Group.
- **Security group policies** can operate in two modes: audit (identify non-compliance) and usage audit (find unused security groups).
- Use the **security account** as the Firewall Manager delegated administrator — not the management account.
- Firewall Manager integrates with **AWS Config** for compliance tracking and **CloudWatch** for policy violation metrics.
- Consider enabling **auto-remediation** carefully for security group policies — aggressive remediation could disrupt application connectivity.

## Relevant Links

- [AWS Firewall Manager](https://docs.aws.amazon.com/waf/latest/developerguide/fms-chapter.html)
- [Network Firewall Policies](https://docs.aws.amazon.com/waf/latest/developerguide/network-firewall-policies.html)
- [Security Group Policies](https://docs.aws.amazon.com/waf/latest/developerguide/security-group-policies.html)
- [WAF Policies](https://docs.aws.amazon.com/waf/latest/developerguide/waf-policies.html)
- [Firewall Manager Prerequisites](https://docs.aws.amazon.com/waf/latest/developerguide/fms-prereq.html)
