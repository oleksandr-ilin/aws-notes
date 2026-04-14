# Q02: Automated Remediation with EventBridge and Config

## Question

A company's security team detects that developers occasionally create security groups with inbound rules allowing SSH (port 22) from `0.0.0.0/0` in their AWS accounts. The team wants a solution that: automatically detects when an overly permissive security group rule is created, removes the offending rule within minutes, notifies the security team via email, and logs the remediation action for audit purposes. The solution must work across all 40 accounts in the AWS Organization.

Which approach should the solutions architect recommend?

## Options

- **A.** Deploy an AWS Config rule (`restricted-ssh`) across all accounts using a conformance pack. Configure automatic remediation using Systems Manager Automation document `AWS-DisablePublicAccessForSecurityGroup`. Set up an Amazon SNS topic in the security account that receives notifications from EventBridge when the Config rule becomes NON_COMPLIANT.
- **B.** Create an Amazon EventBridge rule in each account that matches `AuthorizeSecurityGroupIngress` API calls from CloudTrail. Trigger an AWS Lambda function that checks for `0.0.0.0/0` on port 22, revokes the rule, and sends an SNS notification.
- **C.** Deploy AWS Firewall Manager across the organization. Create a security group policy that defines allowed inbound rules. Firewall Manager will automatically remediate non-compliant security groups.
- **D.** Apply a Service Control Policy (SCP) that denies `ec2:AuthorizeSecurityGroupIngress` when the CIDR is `0.0.0.0/0`. This prevents the overly permissive rule from being created.

## Answers

### A. Config rule + SSM auto-remediation + EventBridge alerts — ✅ Correct

This approach meets all requirements:
- **AWS Config rule** (`restricted-ssh`): Continuously evaluates security groups and detects non-compliant configurations within minutes of creation.
- **Auto-remediation via SSM**: The `AWS-DisablePublicAccessForSecurityGroup` is a pre-built automation document that removes the offending rule. No custom code required.
- **Conformance pack**: Deploys the Config rule consistently across all 40 accounts from the management account.
- **EventBridge notification**: When Config evaluates a resource as NON_COMPLIANT, it generates an event that can trigger an SNS notification to the security team.
- **Audit trail**: Config records the resource change, the compliance evaluation, and the remediation action — all logged and auditable.

### B. EventBridge + Lambda per account — ❌ Incorrect

While this would work, it requires custom Lambda code in each account to evaluate and remediate security group rules. The code must be maintained, tested, and deployed across 40 accounts. Config rules + SSM auto-remediation provides this capability without custom code using built-in AWS services.

### C. AWS Firewall Manager — ❌ Incorrect

Firewall Manager can enforce security group policies, but its remediation approach replaces the entire security group configuration rather than removing individual rules. It's better suited for enforcing baseline security group configurations on specific resources (e.g., all EC2 instances must have a specific security group attached), not for monitoring and remediating individual rule violations.

### D. SCP to deny — ❌ Incorrect

An SCP could deny `ec2:AuthorizeSecurityGroupIngress` with a condition on the CIDR block, but the SCP condition key for IP ranges in security group rules (`ec2:IpPermissions`) has limited granularity. More importantly, blanket denial may break legitimate automation and infrastructure-as-code that applies security groups. The detect-and-remediate approach is more flexible and provides the notification/audit trail required.

## Recommendations

- **AWS Config auto-remediation** supports both manual (approval required) and automatic execution — use automatic for critical security violations.
- Use **AWS-managed SSM Automation documents** where available to avoid maintaining custom Lambda code.
- Deploy Config rules **organization-wide** using conformance packs or Config organization rules from the management account.
- For the notification chain: Config evaluation → EventBridge rule → SNS topic → email/Slack/PagerDuty.
- Consider combining with an SCP that denies security group modifications in **production** accounts (preventive) while using Config + remediation in **development** accounts (detective).

## Relevant Links

- [AWS Config Managed Rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
- [Auto-Remediation with AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/remediation.html)
- [AWS Config Conformance Packs](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html)
- [SSM Automation Documents](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-documents.html)
