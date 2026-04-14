# Q01: Control Tower Landing Zone Setup

## Question

A company is starting its cloud journey and needs to set up a multi-account AWS environment for 8 development teams. The CTO requires: automated account provisioning with standardized configurations, guardrails that enforce encryption and prevent public S3 access, centralized CloudTrail logging, and a dashboard showing compliance status across all accounts. The cloud team has limited AWS experience and wants a managed solution that minimizes custom development.

Which approach should the solutions architect recommend?

## Options

- **A.** Set up AWS Control Tower to create a landing zone. Use Account Factory to provision new accounts with pre-configured settings. Enable mandatory and strongly recommended guardrails. Use the Control Tower dashboard for compliance monitoring.
- **B.** Manually create an AWS Organization with OUs. Write custom CloudFormation StackSets for account baseline configuration. Deploy AWS Config rules for compliance. Build a custom dashboard using Amazon QuickSight.
- **C.** Use AWS Service Catalog to create account vending products. Deploy infrastructure-as-code templates for each new account. Configure AWS CloudTrail and AWS Config independently in each account.
- **D.** Deploy a third-party cloud management platform (e.g., Terraform Cloud) to manage account provisioning and governance. Use Terraform modules for guardrails and compliance monitoring.

## Answers

### A. AWS Control Tower — ✅ Correct

AWS Control Tower is a managed service designed exactly for this scenario:
- **Landing Zone**: Automatically sets up a well-architected multi-account environment with best-practice OUs (Security, Sandbox), log archive account, and audit account.
- **Account Factory**: Self-service portal for provisioning new accounts with pre-approved configurations (VPC settings, Region restrictions, baseline guardrails).
- **Guardrails**: Pre-built preventive (SCP-based) and detective (AWS Config rule-based) controls. Mandatory guardrails are always on; strongly recommended and elective guardrails can be selectively enabled. Examples include "Disallow public read access to S3 buckets" and "Enable encryption for EBS volumes at rest."
- **Dashboard**: Built-in compliance dashboard showing all accounts, OUs, and guardrail compliance status.
- Minimal custom development required — ideal for a team with limited AWS experience.

### B. Manual Organization setup — ❌ Incorrect

This approach provides the same capabilities but requires significant custom development: writing CloudFormation templates for account baselines, deploying Config rules, building aggregation and dashboards, and maintaining all of it over time. Control Tower provides all of this out of the box. This approach is appropriate only when Control Tower's opinionated defaults don't fit the organization's needs.

### C. Service Catalog account vending — ❌ Incorrect

Service Catalog can automate account creation, but you'd still need to build the governance framework (guardrails, compliance monitoring, dashboards) from scratch. Control Tower's Account Factory is built on Service Catalog under the hood, but adds the governance layer. Using Service Catalog alone is reinventing what Control Tower already provides.

### D. Third-party management platform — ❌ Incorrect

Third-party platforms add licensing cost, complexity, and a dependency on external tools. For the described requirements (standard account provisioning, guardrails, compliance), AWS Control Tower provides native coverage without external dependencies. Third-party tools are more appropriate when the company needs multi-cloud management or has existing Terraform/IaC investments.

## Recommendations

- **Control Tower** is the recommended starting point for new AWS multi-account environments — it implements AWS best practices by default.
- Control Tower creates a **Log Archive account** (centralized CloudTrail and Config logs) and **Audit account** (security tooling) automatically.
- **Account Factory** uses AWS Service Catalog under the hood and can be customized with **Account Factory for Terraform (AFT)** for IaC-driven account provisioning.
- Guardrails are categorized as: **Mandatory** (always enabled), **Strongly Recommended**, and **Elective**. Enable strongly recommended guardrails for production OUs.
- Control Tower supports **customizations** — you can add your own SCPs, Config rules, and CloudFormation resources to the baseline using **Customizations for AWS Control Tower (CfCT)**.

## Relevant Links

- [AWS Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html)
- [Landing Zone](https://docs.aws.amazon.com/controltower/latest/userguide/lz-how-it-works.html)
- [Account Factory](https://docs.aws.amazon.com/controltower/latest/userguide/account-factory.html)
- [Guardrails in Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/guardrails.html)
- [Customizations for Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/customize-landing-zone.html)
