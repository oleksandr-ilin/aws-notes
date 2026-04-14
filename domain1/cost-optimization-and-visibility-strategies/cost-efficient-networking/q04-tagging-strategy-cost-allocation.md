# Q04: Tagging Strategy for Cost Allocation

## Question

A company with 60 AWS accounts under AWS Organizations needs to implement a cost allocation strategy. The CFO requires monthly reports that break down AWS costs by: business unit (5 units), environment (production, staging, development), project code, and cost center. Some teams have been inconsistently tagging resources, and 30% of the monthly spend cannot be attributed to any business unit. The cloud governance team needs a solution that enforces tagging compliance and provides accurate cost allocation reports.

Which combination of actions should the solutions architect recommend? **(Select THREE.)**

## Options

- **A.** Define mandatory cost allocation tags (BusinessUnit, Environment, ProjectCode, CostCenter) in the management account. Activate them as user-defined cost allocation tags in the Billing console.
- **B.** Use AWS Organizations Tag Policies to enforce the mandatory tags across all accounts. Configure tag policies with allowed values for BusinessUnit and Environment to prevent inconsistent tagging.
- **C.** Deploy AWS Config rules (e.g., `required-tags`) across all accounts using AWS Config conformance packs. Configure automatic remediation using AWS Systems Manager Automation to tag non-compliant resources.
- **D.** Use AWS Service Catalog to provision resources. Create Service Catalog products with mandatory tag parameters that users must fill in before provisioning.
- **E.** Create IAM policies in each account that deny resource creation unless all four mandatory tags are included in the API call. Attach these policies to all IAM roles and users.
- **F.** Use Amazon Inspector to scan for untagged resources and generate compliance reports monthly.

## Answers

### A. Activate cost allocation tags — ✅ Correct

Activating user-defined cost allocation tags in the management account's Billing console is a prerequisite for tags to appear in Cost Explorer and Cost and Usage Reports. Without activation, even if resources are tagged, the tags won't be available for cost analysis. This is the first step in any cost allocation strategy.

### B. Tag Policies via AWS Organizations — ✅ Correct

Tag Policies enforce consistent tagging across all accounts in the organization. You can define allowed tag key names (preventing "business_unit" vs. "BusinessUnit" vs. "BU" inconsistencies), specify allowed values (e.g., Environment must be "production", "staging", or "development"), and enforce compliance. Tag Policies achieve standardization at the governance level.

### C. Config rules with auto-remediation — ✅ Correct

AWS Config rules detect resources that are missing required tags after creation. Combined with automatic remediation via Systems Manager Automation, non-compliant resources get tagged automatically. This catches resources created through automation, APIs, or CLI that might bypass other controls, and addresses the existing 30% untagged spend by retroactively tagging resources.

### D. Service Catalog — ❌ Incorrect

Service Catalog can enforce tags at provisioning time, but it only works if ALL resource provisioning goes through Service Catalog. Teams using CloudFormation, Terraform, CLI, or the console directly would bypass Service Catalog entirely. It's a useful additional control but cannot be the primary enforcement mechanism.

### E. IAM deny policies — ❌ Incorrect

While IAM condition keys can enforce tags at creation time (e.g., `aws:RequestTag`), this approach is brittle: it requires updating IAM policies across all accounts and roles, may break automation and CI/CD pipelines that don't include tags, and doesn't retroactively tag existing resources. It also requires each service to support tag-on-create, which not all AWS services do.

### F. Amazon Inspector — ❌ Incorrect

Amazon Inspector is a vulnerability scanning service for EC2 instances, Lambda functions, and container images. It does not scan for tagging compliance. AWS Config is the correct service for evaluating resource configuration compliance, including tag requirements.

## Recommendations

- **Layer defenses**: Tag Policies (prevention) + Config rules (detection) + remediation (correction) provides comprehensive coverage.
- Activate cost allocation tags **early** — tags only appear in billing data from the activation date forward, not retroactively.
- Use **AWS-generated cost allocation tags** (e.g., `aws:createdBy`) alongside user-defined tags for additional attribution.
- Consider **SCP-based tag enforcement** as a stronger preventive control for critical resources.
- Regularly run **untagged resource reports** using AWS Config aggregators to track and improve tagging compliance percentages.

## Relevant Links

- [Cost Allocation Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
- [AWS Organizations Tag Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies.html)
- [AWS Config Required Tags Rule](https://docs.aws.amazon.com/config/latest/developerguide/required-tags.html)
- [Tagging Best Practices](https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html)
