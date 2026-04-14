# Q02: Budget Actions for Automated Cost Control

## Question

A media company runs development and testing workloads across 15 AWS accounts. The CTO wants to enforce strict spending limits — if any account exceeds its monthly budget, EC2 instances tagged as "Environment: Dev" should be automatically stopped, and an IAM policy should be applied to deny further `ec2:RunInstances` actions in that account. The solution must work without manual intervention.

Which approach should the solutions architect recommend?

## Options

- **A.** Create an AWS Budget for each account with actual cost thresholds. Configure AWS Budget Actions to apply a custom IAM policy that denies `ec2:RunInstances` and to execute an AWS Systems Manager Automation document to stop EC2 instances tagged "Environment: Dev" when the budget threshold is exceeded.
- **B.** Enable AWS Cost Anomaly Detection to identify unexpected cost increases. Use an Amazon EventBridge rule to trigger an AWS Lambda function that stops EC2 instances and applies a deny IAM policy when an anomaly is detected.
- **C.** Create Amazon CloudWatch billing alarms for each account. Configure the alarm actions to invoke an AWS Lambda function that stops instances and modifies IAM policies when the alarm transitions to ALARM state.
- **D.** Use AWS Organizations Service Control Policies (SCPs) to set spending limits on each account. When the limit is reached, the SCP automatically prevents new resource provisioning.

## Answers

### A. AWS Budget Actions — ✅ Correct

AWS Budget Actions allow you to define automated responses when a budget threshold is reached. You can configure actions to apply an IAM policy (e.g., deny `ec2:RunInstances`) and run Systems Manager Automation documents (e.g., to stop instances by tag). This is the native, fully-automated approach that directly meets all requirements — no custom Lambda code required.

### B. Cost Anomaly Detection + EventBridge + Lambda — ❌ Incorrect

Cost Anomaly Detection identifies unusual spending patterns, not strict budget thresholds. It may trigger on unexpected spikes that are still within budget, or fail to trigger when steady spending crosses a fixed threshold. It also requires custom Lambda code for the stop/deny actions, adding unnecessary complexity when Budget Actions provides this natively.

### C. CloudWatch billing alarms + Lambda — ❌ Incorrect

CloudWatch billing alarms can trigger notifications but have limited native action capabilities. While you could use Lambda as a target, this requires writing and maintaining custom code to stop instances and modify IAM policies. Budget Actions provides this capability out of the box without custom code.

### D. SCPs for spending limits — ❌ Incorrect

Service Control Policies do not support cost-based conditions. SCPs define maximum available permissions for accounts — they cannot be triggered by spending thresholds. There is no "spending limit" attribute in SCP condition keys.

## Recommendations

- **AWS Budget Actions** supports three action types: applying IAM policies, applying SCPs, and running SSM Automation runbooks — use them for automated cost control.
- Budget Actions can run in "auto" mode (execute immediately) or "approval" mode (require manual confirmation) — choose based on risk tolerance.
- Combine Budget Actions with **AWS Budgets alerts** via SNS to ensure account owners are also notified, not just auto-remediated.
- For accounts that should never exceed a hard limit, consider Budget Actions in the management account with cross-account roles.

## Relevant Links

- [AWS Budget Actions](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html)
- [Configuring AWS Budgets Actions](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-create.html#budgets-create-actions)
- [AWS Systems Manager Automation](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html)
- [AWS Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/manage-ad.html)
