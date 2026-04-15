# Q01: CloudFormation StackSets for Multi-Account Multi-Region Deployment

## Question

A company uses AWS Organizations with 30 accounts across 4 Regions. The platform team must deploy a standardized security baseline (CloudTrail, Config, GuardDuty, and a logging S3 bucket) to every account and Region. Requirements:
- New accounts added to the Organization must automatically receive the baseline
- Updates to the baseline template must propagate to all accounts without manual intervention
- Deployment failures in one account must not block deployment to other accounts
- The platform team must track deployment status and detect drift from the baseline

Which deployment strategy should the solutions architect implement?

## Options

- **A.** Create a CloudFormation StackSet with service-managed permissions in the management account. Target the entire Organization (or specific OUs). Enable automatic deployment for new accounts. Configure the StackSet with `REGIONAL` concurrency mode, `failure tolerance` of 10%, and enable drift detection on a schedule. Use CloudFormation change sets to preview updates before propagation.
- **B.** Create individual CloudFormation stacks in each account using a cross-account IAM role. Write a Lambda function triggered by the Organizations `CreateAccountResult` event to deploy stacks in new accounts. Monitor each stack individually with CloudWatch alarms.
- **C.** Use AWS Service Catalog with a portfolio shared across the Organization. Require each account administrator to launch the security baseline product manually. Use Service Catalog constraints to enforce approved configurations.
- **D.** Use Terraform with a remote S3 backend and DynamoDB lock table. Create a CI/CD pipeline that iterates over all accounts and runs `terraform apply` in each. Store state files per account in the management account's S3 bucket.

## Answers

### A. CloudFormation StackSets with service-managed permissions — ✅ Correct

This is the purpose-built solution:
- **Service-managed permissions**: StackSets integrates directly with Organizations — no need to create IAM roles in each account manually. AWS automatically creates the `AWSCloudFormationStackSetExecutionRole` in member accounts.
- **Automatic deployment**: When enabled, any new account added to the targeted OU automatically receives the stack instances. No custom Lambda required.
- **Failure tolerance**: `MaxConcurrentPercentage` and `FailureTolerancePercentage` parameters control how StackSets handles failures — a failure in one account doesn't block others.
- **Drift detection**: StackSets supports drift detection across all stack instances, letting the team identify accounts that have deviated from the baseline.
- **Change sets**: Preview changes before applying updates across all accounts — critical for change management.

### B. Individual stacks + Lambda — ❌ Incorrect

This works but is a custom-built solution replicating what StackSets provides natively. Managing individual stacks across 30 accounts × 4 Regions = 120 stacks is operationally expensive. The Lambda function for new account provisioning requires building, testing, and maintaining custom code. No consolidated drift detection or coordinated update mechanism — each stack is independent.

### C. Service Catalog — ❌ Incorrect

Service Catalog requires account administrators to *launch* products — there's no automatic deployment to new accounts. This violates the requirement that new accounts automatically receive the baseline. Service Catalog is better suited for self-service provisioning of optional infrastructure, not mandatory organizational baselines. Tracking compliance (did every account launch the product?) requires additional tooling.

### D. Terraform with CI/CD — ❌ Incorrect

While Terraform is a valid IaC tool, this approach requires: managing state files per account, handling cross-account authentication in the pipeline, iterating over all accounts in code, and building custom automation for new account detection. StackSets provides all of this natively. Additionally, storing all state files in the management account creates a large blast radius if compromised, and DynamoDB lock contention can slow parallel deployments.

## Recommendations

- Use **service-managed permissions** (not self-managed) for Organization-wide StackSets — it handles IAM role creation automatically.
- Set **failure tolerance > 0** for large deployments so one account failure doesn't halt everything.
- Enable **automatic deployment** at the OU level — this ensures governance compliance for new accounts.
- Schedule **drift detection** weekly or daily for security-critical baselines.
- Use **StackSet change sets** in a CI/CD pipeline (CodePipeline → CloudFormation StackSet action) for automated yet safe rollouts.
- Consider **delegated administrator** for StackSets — lets a non-management account manage StackSets to reduce management account usage.

## Relevant Links

- [CloudFormation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)
- [Service-Managed Permissions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-manage-auto-provision.html)
- [StackSet Drift Detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-drift.html)
- [StackSet Operations](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-update.html)
