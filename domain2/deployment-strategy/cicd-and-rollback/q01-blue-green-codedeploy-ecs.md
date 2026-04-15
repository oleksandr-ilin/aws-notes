# Q01: Blue/Green Deployment with CodeDeploy and Automatic Rollback

## Question

A company runs a critical e-commerce application on Amazon ECS with an Application Load Balancer. They need to deploy a new application version with these requirements:
- Zero downtime during deployment
- The ability to test the new version with a small percentage of production traffic before full cutover
- Automatic rollback if the error rate exceeds 2% or p99 latency exceeds 500 ms during the deployment
- The old version must remain available for at least 15 minutes after cutover for emergency manual rollback

Which deployment strategy should the solutions architect configure?

## Options

- **A.** Configure CodeDeploy with an ECS blue/green deployment. Create two target groups on the ALB. Define a CodeDeploy deployment group with a `Linear10PercentEvery5Minutes` traffic shifting configuration. Configure `DeploymentReadyOption` with a 15-minute wait before terminating the original task set. Add CloudWatch alarms for HTTPCode_Target_5XX_Count and target response time p99. Enable automatic rollback on alarm threshold breach and deployment failure.
- **B.** Use ECS rolling update deployment type (default). Set `minimumHealthyPercent` to 50% and `maximumPercent` to 200%. Monitor CloudWatch metrics manually and run `aws ecs update-service` to revert if issues are detected.
- **C.** Deploy the new version as a separate ECS service behind the same ALB. Use Route 53 weighted routing to shift 10% of traffic to the new service. Monitor metrics and manually update Route 53 weights over time. Delete the old service after validation.
- **D.** Use AWS Elastic Beanstalk with immutable deployment type. Let Beanstalk create a new Auto Scaling group, run health checks, and swap. Enable Beanstalk enhanced health reporting for rollback decisions.

## Answers

### A. CodeDeploy ECS blue/green with traffic shifting + alarm-based rollback — ✅ Correct

This is the correct approach:
- **ECS blue/green via CodeDeploy**: Creates a replacement task set (green) alongside the original (blue). Both run simultaneously with separate target groups on the ALB.
- **Linear traffic shifting**: `Linear10PercentEvery5Minutes` gradually moves production traffic — 10% at a time — to the green target group. This provides canary-like validation with real production traffic before full cutover.
- **CloudWatch alarm-based rollback**: CodeDeploy monitors specified alarms during deployment. If the 5XX rate or p99 latency alarm triggers, CodeDeploy automatically reroutes 100% of traffic back to the blue target group. No manual intervention needed.
- **15-minute termination wait**: `DeploymentReadyOption` waits before terminating old tasks — the blue task set stays running and can receive traffic via manual reroute during this window.
- **Automatic rollback on failure**: CodeDeploy natively supports rollback on deployment failure (health check failures) and alarm breach.

### B. ECS rolling update — ❌ Incorrect

Rolling updates replace tasks incrementally but **cannot shift a percentage of traffic** — the ALB routes to all healthy tasks equally. There's no staged traffic shifting or alarm-based automatic rollback. If a bad version starts receiving traffic, you must manually detect and revert. `minimumHealthyPercent` at 50% means up to half the tasks could be the new version before any validation. This doesn't meet the controlled traffic shifting or automatic rollback requirements.

### C. Separate ECS service + Route 53 weighted routing — ❌ Incorrect

Route 53 weighted routing operates at DNS level with TTL caching — traffic shifting is imprecise and slow (DNS TTL of 60s minimum, client caching varies). This doesn't provide the ALB-level, request-by-request traffic control that CodeDeploy offers. Manual weight updates and service deletion are operationally risky. No integrated automatic rollback on CloudWatch alarms. Two separate ECS services doubles the management surface.

### D. Elastic Beanstalk immutable — ❌ Incorrect

Beanstalk immutable deployment is an all-or-nothing swap — it does not support gradual traffic shifting to test with a percentage of production traffic. Enhanced health reporting provides health data but not alarm-integrated automatic rollback mid-deployment. The application runs on ECS (not Beanstalk), so switching platforms would require migration effort. Beanstalk's deployment model is less flexible than CodeDeploy for ECS-native workloads.

## Recommendations

- CodeDeploy supports three traffic shifting options for ECS: **Canary** (shift X%, wait, then 100%), **Linear** (shift X% every Y minutes), and **All-at-once** (immediate full shift).
- **Canary10Percent5Minutes** is best for quick validation; **Linear10PercentEvery5Minutes** gives more gradual confidence.
- Always define meaningful **CloudWatch alarms** before deployment — error rate, latency, and custom business metrics.
- Use **deployment hooks** (BeforeAllowTraffic, AfterAllowTraffic) for integration test Lambda functions that validate the new version before more traffic shifts.
- Include **rollback alarms** that cover a post-deployment bake time — not just during shifting.
- Integrate CodeDeploy deployments into **CodePipeline** for end-to-end CI/CD with approval gates.

## Relevant Links

- [CodeDeploy ECS Blue/Green](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-groups-create-ecs.html)
- [ECS Traffic Shifting](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html#appspec-hooks-ecs)
- [CodeDeploy Automatic Rollback](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-rollback-and-redeploy.html)
- [CodeDeploy Deployment Configurations](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html)
