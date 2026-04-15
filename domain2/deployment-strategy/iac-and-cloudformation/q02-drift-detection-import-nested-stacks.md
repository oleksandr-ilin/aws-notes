# Q02: CloudFormation Drift Detection, Imports, and Nested Stacks

## Question

A company manages 200+ CloudFormation stacks across 3 Regions. They face three challenges:

1. **Drift**: Operations teams manually modify security groups and IAM roles via the console during incidents, causing stacks to drift from their templates. They need to detect and remediate drift continuously.
2. **Legacy resources**: 50 EC2 instances, 30 RDS databases, and 100 S3 buckets were created manually before IaC adoption. These must be brought under CloudFormation management without downtime or recreation.
3. **Template sprawl**: 80 stacks share identical networking, monitoring, and logging configurations. Updates to shared components require editing 80 templates individually.

Which approach addresses all three challenges?

## Options

- **A.** (1) Enable AWS Config rule `cloudformation-stack-drift-detection-check` to continuously detect drift across all stacks, with auto-remediation via SSM Automation that runs `DetectStackDrift` and alerts on drifted resources. (2) Use CloudFormation `resource import` to bring existing resources into new or existing stacks by providing the resource identifier and matching template definition. (3) Refactor shared components into CloudFormation nested stacks — a parent stack references child templates (networking.yaml, monitoring.yaml, logging.yaml) stored in S3. Updates to a child template propagate to all parent stacks on the next update.
- **B.** (1) Write a Lambda function that calls `DetectStackDrift` on every stack hourly via EventBridge scheduled rule. (2) Delete and recreate all legacy resources using CloudFormation. (3) Use CloudFormation modules (registry) for shared components.
- **C.** (1) Use CloudFormation change sets to preview drift before every deployment. (2) Use AWS CDK `cdk import` to import legacy resources. (3) Use cross-stack references (Export/Import) for shared components.
- **D.** (1) Enable CloudTrail to log all console modifications and manually reconcile with templates monthly. (2) Use Terraform `terraform import` for legacy resources, then convert to CloudFormation. (3) Use AWS Service Catalog to standardize shared components.

## Answers

### A. Config drift rule + resource import + nested stacks — ✅ Correct

- **(1) AWS Config `cloudformation-stack-drift-detection-check`**:
  - This managed Config rule continuously evaluates whether CloudFormation stacks have drifted. It calls `DetectStackDrift` on a configurable schedule and marks stacks as NON_COMPLIANT when drift is detected.
  - **Auto-remediation via SSM Automation**: When drift is detected, an SSM Automation document can: (a) notify the team via SNS, (b) generate a drift report, (c) optionally run `UpdateStack` to revert to the template definition (after review).
  - This is the AWS-native, fully automated approach — no custom Lambda needed.
  - **Key nuance**: Not all resource properties support drift detection. Check the [resource type coverage](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html) list.

- **(2) CloudFormation resource import**:
  - `resource import` allows you to bring existing AWS resources into a CloudFormation stack WITHOUT recreating them. You provide: the resource's physical ID (e.g., instance ID, bucket name), and a template that includes a matching resource definition.
  - **Zero downtime**: The resource is adopted as-is. CloudFormation doesn't modify it — it just starts managing it. Future stack updates will manage the resource through CloudFormation.
  - **Process**: Create a change set of type `IMPORT`, specify the resource identifier mapping, review the change set, then execute. The resource appears in the stack's resource list.
  - **Must define the resource in the template**: The template must have a resource definition that matches the existing resource's configuration. If the template differs from reality, drift is immediately detected.

- **(3) Nested stacks for shared components**:
  - A parent stack uses `AWS::CloudFormation::Stack` to reference child templates stored in S3. The child template is parameterized (VPC CIDR, retention period, etc.).
  - **Update propagation**: Update the child template (e.g., `monitoring.yaml`) in S3, then update any parent stack. CloudFormation detects the child template change and applies updates during the parent stack update.
  - **80 stacks sharing 3 child templates**: Instead of 80 × 3 = 240 duplicate configuration blocks, you maintain 3 child templates. Changes are authored once and deployed to each parent stack on its next update cycle.
  - **Tradeoff**: Parent stacks must be explicitly updated to pick up child template changes (unlike modules, which are version-locked at the registry level). But nested stacks are simpler and widely understood.

### B. Custom Lambda drift + delete/recreate + modules — ❌ Incorrect

- **Custom Lambda for drift detection**: Functional but re-implements what AWS Config's managed rule already provides. The Lambda needs error handling, pagination (200+ stacks), rate limiting (API throttling on `DetectStackDrift`), and monitoring. AWS Config handles all of this natively.
- **Delete and recreate legacy resources**: This causes downtime. Deleting 50 EC2 instances means stopping running applications. Deleting 30 RDS databases means data loss (unless snapshots + restore, which still causes downtime). CloudFormation resource import avoids this entirely.
- **CloudFormation modules**: Modules are registered in the CloudFormation registry and used like custom resource types. They're suitable for reusable component patterns but require module development, versioning, and registry management — more complex than nested stacks for straightforward shared configurations. Modules are better for publishing reusable components across organizations, not for refactoring 80 existing stacks.

### C. Change sets for drift + CDK import + cross-stack refs — ❌ Incorrect

- **Change sets for drift preview**: Change sets show what CloudFormation WOULD change during a stack update — they don't detect drift (changes made outside CloudFormation). Drift detection is a separate API (`DetectStackDrift`). A change set might show no changes even when resources have drifted.
- **CDK `cdk import`**: While CDK does support importing resources, using CDK requires rewriting all 200+ CloudFormation templates in CDK (TypeScript/Python). This is a massive migration effort when the requirement is just to import 180 legacy resources. Native CloudFormation resource import works directly with existing templates.
- **Cross-stack references (Export/Import)**: Exports create tight coupling — an exporting stack cannot be updated if its exports are consumed by other stacks. With 80 stacks consuming shared exports, updating the shared stack becomes extremely difficult (all consumers must be updated first). This creates "export lock-in." Nested stacks avoid this by encapsulating the shared resources.

### D. CloudTrail manual reconciliation + Terraform convert + Service Catalog — ❌ Incorrect

- **CloudTrail + monthly manual reconciliation**: CloudTrail logs API calls but doesn't compare them against desired state. Manually reviewing audit logs for 200+ stacks monthly is operationally unsustainable and doesn't catch drift until the next review cycle (up to 30 days of undetected drift).
- **Terraform import → convert to CloudFormation**: Introduces a second IaC tool (Terraform), imports resources into Terraform state, then requires converting Terraform HCL to CloudFormation YAML/JSON. This adds unnecessary complexity and tooling risk. CloudFormation resource import does this directly in one step.
- **Service Catalog**: Standardizes how products are launched (approved templates, constraints, permissions) but doesn't solve drift detection, resource import, or template refactoring for existing stacks. Service Catalog is for governance of new deployments, not for managing existing infrastructure.

## Recommendations

- **AWS Config drift detection** is the recommended continuous drift monitoring approach — enable it for all CloudFormation-managed stacks.
- **Resource import** supports most resource types (EC2, RDS, S3, Lambda, IAM, VPC, etc.) — check the [supported resources list](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html) before attempting import.
- **DeletionPolicy: Retain** on imported resources prevents accidental deletion if the stack is deleted — always set this during the import process.
- **Nested stacks vs. Modules vs. Cross-stack refs**: Use nested stacks for simple shared configurations within an account. Use modules for cross-organization reusable components. Avoid cross-stack exports for tightly-coupled shared infrastructure.
- **Import workflow**: (1) Write the resource definition in the template, (2) create import change set, (3) map physical IDs, (4) execute, (5) run drift detection to verify alignment.

## Relevant Links

- [CloudFormation Drift Detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html)
- [CloudFormation Resource Import](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import.html)
- [Nested Stacks](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html)
- [AWS Config CloudFormation Drift Rule](https://docs.aws.amazon.com/config/latest/developerguide/cloudformation-stack-drift-detection-check.html)
