# Q01: Centralized Monitoring for Proactive Recovery

## Question

A SaaS company runs microservices across 3 Regions. Each Region has an ALB, ECS Fargate cluster, Aurora database, and ElastiCache Redis. The operations team needs:
- A single dashboard showing health status of all Regions
- Automatic detection when a Region becomes degraded (not fully down) — e.g., error rate rising above 1% or database replication lag exceeding 5 seconds
- Automatic execution of a predefined runbook when degradation is detected, including scaling up the healthy Region and notifying the on-call engineer via PagerDuty
- Weekly failure injection tests to validate the monitoring and recovery pipeline

Which monitoring and recovery architecture should the solutions architect implement?

## Options

- **A.** Create CloudWatch cross-account/cross-Region dashboards in a central monitoring account. Build composite alarms that combine per-Region metrics: ALB 5XX rate > 1%, Aurora replica lag > 5s, and ECS task health < 90%. Route composite alarm state changes to EventBridge, which triggers an SSM Automation runbook to scale up the healthy Region's ASG and send a PagerDuty notification via SNS. Schedule weekly FIS experiments that inject Aurora failover and ECS task failures to validate the pipeline.
- **B.** Install a third-party monitoring agent (Datadog/New Relic) on all ECS tasks. Configure third-party alerting rules and dashboards. Use the third-party's webhook integration to trigger a Lambda function for scaling. Manually run game days quarterly for DR testing.
- **C.** Create per-Region CloudWatch dashboards. Set individual CloudWatch alarms per metric per Region. Send all alarms to a shared SNS topic with email notifications. The on-call engineer reviews the dashboard and manually decides whether to scale up another Region. Skip failure injection testing to avoid production risk.
- **D.** Use AWS Health Dashboard to monitor Region-wide outages. Configure Personal Health Dashboard notifications via EventBridge to trigger the runbook. Use Trusted Advisor to check for performance degradation. Rely on AWS-detected outages as the primary degradation signal.

## Answers

### A. Cross-account dashboards + composite alarms + EventBridge + SSM + FIS — ✅ Correct

This covers proactive detection, automated response, and validated testing:
- **Cross-account/cross-Region dashboards**: CloudWatch supports cross-account observability — a central monitoring account can display metrics, logs, and alarms from all 3 Regions in one view. This gives the single-pane-of-glass visibility required.
- **Composite alarms**: Combine multiple metric conditions into a single alarm state. Example: the "Region-degraded" composite alarm triggers when ANY of (5XX > 1% OR replica lag > 5s OR task health < 90%) is true. This detects degradation patterns before a full outage.
- **EventBridge → SSM Automation**: Alarm state change events route to EventBridge, which triggers SSM Automation runbooks. Runbooks can: increase desired count on healthy Region ECS services, update Route 53 routing weights, and notify PagerDuty via SNS with HTTPS endpoint.
- **FIS weekly experiments**: Fault Injection Simulator can inject: Aurora cluster failover, ECS task stop, network disruption, and CPU stress. This validates that alarms fire, EventBridge routes correctly, and the runbook executes as expected — all without manual coordination.

### B. Third-party monitoring + manual game days — ❌ Incorrect

- Third-party tools add cost and require agent management on serverless workloads (Fargate sidecars add complexity).
- Lambda-based scaling triggered by webhooks introduces a custom code path that must be maintained and tested.
- Quarterly game days are insufficient — weekly testing builds confidence and catches drift in runbooks.
- Third-party monitoring doesn't natively integrate with SSM Automation for AWS-specific runbooks.
- This approach works but is more expensive and complex than native AWS tooling for this use case.

### C. Per-Region dashboards + manual response — ❌ Incorrect

- **Per-Region dashboards** don't provide a single consolidated view — the engineer must switch between 3 dashboards.
- **Individual alarms with email notification** create alert fatigue and require manual triage. No composite alarm means the engineer must correlate multiple alerts mentally.
- **Manual scaling decision** violates the automation requirement and adds human response time (15-30+ minutes) to the recovery.
- **Skipping failure injection** means the monitoring and recovery pipeline is never validated — when a real incident occurs, untested runbooks are likely to fail.

### D. AWS Health Dashboard + Trusted Advisor — ❌ Incorrect

- **AWS Health Dashboard** reports AWS-detected outages (typically announced after they've been confirmed) — it does not detect application-level degradation (your 5XX rate or DB lag).
- By the time AWS announces a Region issue, your customers have already been impacted for minutes to hours.
- **Trusted Advisor** provides best-practice recommendations (cost, security, performance) — it does not provide real-time performance degradation signals.
- This approach detects only AWS infrastructure outages, not application-level degradation, and cannot be proactive.

## Recommendations

- **CloudWatch cross-account observability**: Set up a central monitoring account with `OrganizationAccountAccessRole`-based sharing. Use `aws:cloudwatch:cross-account` to aggregate metrics.
- **Composite alarms** are critical for reducing noise — instead of 9 individual alarms per Region (3 metrics × 3 Regions), create 3 composite alarms (one per Region) that encapsulate the degradation definition.
- **SSM Automation runbooks** should be idempotent — safe to run multiple times without side effects. Include verification steps (check current state before acting).
- **FIS experiment templates** can be version-controlled and integrated into CI/CD — treat chaos engineering as part of the release process.
- Consider **Amazon Managed Service for Prometheus** + **Amazon Managed Grafana** for more advanced cross-account/cross-Region dashboards with PromQL queries.

## Relevant Links

- [CloudWatch Cross-Account Observability](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Unified-Cross-Account.html)
- [Composite Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html)
- [SSM Automation Runbooks](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-documents.html)
- [AWS Fault Injection Simulator](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
- [EventBridge Integration with SSM](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-ssm.html)
