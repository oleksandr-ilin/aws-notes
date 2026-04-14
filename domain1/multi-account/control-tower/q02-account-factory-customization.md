# Q02: Account Factory Customization

## Question

A regulated financial institution uses AWS Control Tower to manage 100+ accounts. The compliance team requires that every new account must be provisioned with: a specific VPC CIDR range from a centralized IPAM (IP Address Manager), an AWS Config conformance pack for PCI-DSS compliance checks, a hardened AMI pipeline that publishes golden AMIs from a shared services account, and automated enrollment in Amazon GuardDuty and AWS Security Hub. Currently, account creation takes 3 days because engineers manually configure these items after Account Factory creates the account.

Which approach should the solutions architect recommend to automate the post-creation customization?

## Options

- **A.** Use Customizations for AWS Control Tower (CfCT) to deploy additional CloudFormation StackSets that configure the VPC with IPAM-allocated CIDRs, deploy Config conformance packs, configure AMI sharing from the shared services account, and enable GuardDuty and Security Hub enrollment — all triggered automatically when Account Factory creates a new account.
- **B.** Create an AWS Lambda function triggered by the `CreateManagedAccount` CloudTrail event. The Lambda function calls AWS APIs to configure the VPC, deploy Config packs, set up AMI access, and enable security services in the new account.
- **C.** Modify the Account Factory product in AWS Service Catalog to include all customizations in a single CloudFormation template. Deploy the VPC, Config rules, AMI pipeline, and security services as part of the Account Factory product.
- **D.** Create a manual runbook documenting all post-creation steps. Train the cloud operations team to follow the runbook within 24 hours of account creation to ensure compliance.

## Answers

### A. Customizations for Control Tower (CfCT) — ✅ Correct

CfCT is the AWS-recommended approach for extending Account Factory with custom configurations:
- Deploys additional **CloudFormation StackSets** automatically when accounts are created or updated.
- Integrates with **Service Control Policies** for additional guardrails.
- Uses a **manifest file** to define which customizations apply to which OUs or accounts.
- Runs as a **pipeline** (CodePipeline + CodeBuild) triggered by Control Tower lifecycle events.
- Supports **VPC with IPAM integration** for centralized IP management.
- This is a managed, auditable, and repeatable solution — no custom Lambda code to maintain.

### B. Lambda triggered by CloudTrail — ❌ Incorrect

While this would work technically, it requires maintaining custom Lambda code, error handling, retry logic, and monitoring. If the Lambda fails, the account may be partially configured without clear remediation. CfCT provides a managed pipeline with built-in error handling, drift detection, and rollback capabilities.

### C. Modify Account Factory product — ❌ Incorrect

Account Factory's Service Catalog product has limitations on what can be customized directly. VPC settings in Account Factory are limited to predefined options (up to 2 subnets per VPC). Complex configurations like IPAM integration, Config conformance packs, and cross-account AMI sharing require additional StackSets beyond what the native Account Factory product supports.

### D. Manual runbook — ❌ Incorrect

Manual configuration introduces human error, inconsistency, and delays. A 3-day provisioning time with manual steps violates the principle of automation. For 100+ accounts in a regulated environment, manual configuration is an audit risk.

## Recommendations

- **CfCT** uses a code-based manifest (`manifest.yaml`) stored in CodeCommit — treat it as infrastructure-as-code.
- For Terraform-based organizations, consider **Account Factory for Terraform (AFT)** as an alternative to CfCT.
- Use **Control Tower lifecycle events** (CloudTrail events) for any automation that CfCT doesn't cover.
- Enable **delegated administrator** for GuardDuty and Security Hub in the security account to avoid using the management account.
- Use **AWS Service Catalog** constraints to restrict which Account Factory options are available to different teams.

## Relevant Links

- [Customizations for AWS Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/customize-landing-zone.html)
- [Account Factory for Terraform](https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html)
- [Control Tower Lifecycle Events](https://docs.aws.amazon.com/controltower/latest/userguide/lifecycle-events.html)
- [AWS VPC IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html)
