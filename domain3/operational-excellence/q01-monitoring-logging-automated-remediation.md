# Q01: Improving Monitoring, Logging, and Automated Remediation

## Question

A company operates a three-tier application (ALB → ECS Fargate → Aurora) across two Regions. After a recent 4-hour outage, the post-mortem revealed multiple operational gaps:

1. **Detection was slow**: The on-call engineer was alerted 45 minutes after the issue started. CloudWatch alarms existed but only monitored CPU and memory — the actual failure was Aurora connection exhaustion (max connections reached), which had no alarm
2. **Logs were fragmented**: ECS container logs went to CloudWatch Logs in each Region. Aurora slow query logs were disabled. ALB access logs were in S3 but nobody analyzed them. No centralized view — the engineer checked 4 consoles to correlate events
3. **Root cause took hours**: The engineer manually correlated timestamps across ALB access logs (increased 5xx), ECS task logs (connection pool errors), and Aurora metrics (connections at max). No automated correlation existed
4. **Remediation was manual**: After identifying connection exhaustion, the engineer manually increased Aurora `max_connections` parameter and restarted the cluster. No runbook or automation existed for this known failure mode

The CTO wants: (a) detection within 5 minutes, (b) centralized log analysis, (c) automated root cause correlation, and (d) automated remediation for known failure patterns.

Which improvement strategy addresses all four gaps?

## Options

- **A.** (a) Create **CloudWatch composite alarm**: sub-alarms on Aurora `DatabaseConnections` > 80% of `max_connections` (breaching = connection exhaustion risk) + ECS `HTTPCode_Target_5XX_Count` > threshold + ALB `TargetResponseTime` P99 > 5 seconds. Composite alarm triggers when 2+ sub-alarms are in ALARM state simultaneously — sends SNS alert within 5 minutes. (b) Deploy **CloudWatch Logs cross-account/cross-Region aggregation** — stream all ECS logs, Aurora slow query logs (enable `slow_query_log`), and ALB access logs to a centralized **OpenSearch** domain via Kinesis Data Firehose. OpenSearch dashboards provide a single-pane view. (c) Enable **CloudWatch Application Insights** on the application group — it uses ML to detect anomalies and correlates related metrics and logs into a single "problem" view. Enable **X-Ray tracing** for request-level correlation. (d) Create a **Systems Manager Automation** runbook for connection exhaustion: triggered by the composite alarm via EventBridge → SSM Automation document that: modifies the Aurora parameter group (`max_connections`), applies changes, scales ECS tasks down then up (to reset connection pools), and sends notification. Test the runbook with **Fault Injection Simulator** regularly.
- **B.** Install Datadog agents on all instances. Set up PagerDuty for alerting. Use Datadog APM for tracing. Manual remediation with documented runbooks.
- **C.** Create individual CloudWatch alarms for every Aurora metric. Send all logs to S3 and query with Athena. Use CloudWatch Contributor Insights for root cause. Manual remediation with AWS CLI scripts.
- **D.** Enable AWS Health Dashboard notifications. Use CloudTrail for all logging. Enable RDS Performance Insights for database analysis. Create Lambda functions triggered by CloudWatch alarms for remediation.

## Answers

### A. Composite alarms + centralized logging (OpenSearch) + Application Insights + SSM Automation — ✅ Correct

**Gap (a) — Fast Detection with Composite Alarms:**

- **Why the original alarms failed**: CPU and memory alarms don't detect connection exhaustion. Aurora can run at 30% CPU with 100% connections consumed — CPU alarms never fire while the database is completely saturated.

- **Composite alarm design**:
  - Sub-alarm 1: `DatabaseConnections / max_connections > 0.8` for 2 consecutive 1-minute periods. This catches connection exhaustion before it causes failures.
  - Sub-alarm 2: `HTTPCode_Target_5XX_Count > 50` for 1 minute. The application is returning errors.
  - Sub-alarm 3: `TargetResponseTime P99 > 5s` for 2 minutes. User experience is degrading.
  - Composite: `ALARM(connections) AND (ALARM(5xx) OR ALARM(latency))` — requires the connection issue PLUS user-visible impact. This prevents false alerts when connections are high but performance is fine.
  - **Detection time**: Health check interval (1 min) × evaluation periods (2) = **2 minutes**. SNS notification adds < 30 seconds. Total: < 3 minutes.

- **Missing metric — `max_connections`**: Aurora doesn't publish `max_connections` as a CloudWatch metric. You must: (1) query the parameter group value via API, (2) publish it as a custom CloudWatch metric via Lambda on a schedule, or (3) hard-code the threshold based on the known `max_connections` value for the instance class (calculated as `{DBInstanceClassMemory / 1048576 * 45}`).

**Gap (b) — Centralized Logging:**

- **Current state**: Logs scattered across 4 sources in 2 Regions — impossible to correlate during an incident.

- **Architecture**: 
  - ECS container logs → CloudWatch Logs (already there) → **Subscription filter** → Kinesis Data Firehose → OpenSearch
  - Aurora slow query logs → CloudWatch Logs (enable `slow_query_log` and `log_output = FILE` in the parameter group) → Subscription filter → Firehose → OpenSearch
  - ALB access logs → S3 → Lambda (S3 event trigger) → OpenSearch
  - Both Regions stream to a centralized OpenSearch domain in the primary Region

- **OpenSearch dashboards**: Create dashboards showing: ALB 5xx rate over time, top slow queries, ECS error log patterns, correlation by timestamp. During an incident, one dashboard shows the complete picture.

- **Alternative — CloudWatch Logs Insights**: For teams not wanting to manage OpenSearch, CloudWatch Logs Insights can query across log groups with SQL-like queries. Less powerful than OpenSearch for dashboards but simpler and serverless.

**Gap (c) — Automated Root Cause Correlation:**

- **CloudWatch Application Insights**:
  - Groups related resources (ECS cluster, Aurora, ALB) into an "application" and monitors them collectively.
  - Uses ML to detect anomalies across the group — if Aurora connections spike AND ECS errors increase simultaneously, Application Insights correlates them into a single "problem" with a timeline showing the sequence of events.
  - Significantly reduces MTTR (Mean Time To Resolve) by eliminating manual correlation work.

- **X-Ray tracing**: Traces individual requests showing: ALB (2 ms) → ECS (50 ms) → Aurora query (timeout after 30s). The trace immediately shows the Aurora connection timeout as the bottleneck. Without tracing, you only know "the request was slow" — not where.

**Gap (d) — Automated Remediation with SSM:**

- **SSM Automation runbook**:
  - Triggered by EventBridge rule matching the composite alarm state change → `ALARM`
  - Runbook steps:
    1. `aws:executeAwsApi` — Modify Aurora parameter group: increase `max_connections` by 20%
    2. `aws:executeAwsApi` — Reboot Aurora reader instances (applies parameter change)
    3. `aws:executeAwsApi` — Update ECS service desired count (force new task deployment to reset connection pools)
    4. `aws:executeAwsApi` — SNS publish notification with actions taken
  - Each step has error handling (`onFailure: Abort`) and timeout
  - The runbook is **idempotent** — running it twice doesn't cause issues (parameter modification to the same value is a no-op)

- **FIS testing**: Use Fault Injection Simulator to inject an Aurora connection spike (via a Lambda that opens hundreds of connections). Verify the composite alarm fires → EventBridge triggers → SSM runbook executes → connections are relieved → alarm returns to OK. Run monthly.

### B. Third-party tooling (Datadog + PagerDuty) — ❌ Incorrect

- Third-party tools (Datadog, PagerDuty) can solve these problems, but the question asks for AWS solutions specifically. From an exam perspective, AWS-native solutions are preferred unless the question explicitly mentions existing third-party tools.
- Datadog requires agents on all hosts — Fargate doesn't have traditional hosts (sidecar container needed). Aurora monitoring requires Datadog's RDS integration (separate setup).
- "Manual remediation with documented runbooks" doesn't meet the "automated remediation" requirement. Documented runbooks help humans act faster but don't eliminate human intervention.

### C. Individual alarms + S3/Athena + Contributor Insights — ❌ Incorrect

- **Individual alarms for every Aurora metric**: Creates alarm fatigue — dozens of alarms firing independently without correlation. A composite alarm consolidates signals into one meaningful alert. Individual alarms generate noise: "connections high" + "CPU elevated" + "IOPS increased" as three separate notifications instead of one correlated problem.
- **S3 + Athena for log analysis**: Athena is excellent for ad-hoc analysis but provides no real-time dashboard. Query latency is seconds to minutes (Athena scans S3 data). During an incident, you need real-time log tailing — OpenSearch or CloudWatch Logs Insights provides this. Athena is for post-incident forensics, not real-time incident response.
- **Contributor Insights**: Identifies top contributors to a metric (e.g., "which IP addresses generate the most 5xx errors"). Useful for specific analyses but doesn't provide holistic root cause correlation across multiple services. Application Insights provides the multi-service correlation.
- **AWS CLI scripts for remediation**: Scripts require human execution (SSH, run script, verify output). Not automated. SSM Automation runs unattended with error handling, audit trail, and rollback.

### D. Health Dashboard + CloudTrail + Performance Insights + Lambda — ❌ Incorrect

- **AWS Health Dashboard**: Shows AWS service-level issues (e.g., "EC2 degraded in us-east-1") — not application-level issues. Connection exhaustion in YOUR application doesn't appear in the Health Dashboard. It might show if Aurora itself has an issue, but not your parameter misconfiguration.
- **CloudTrail for logging**: CloudTrail logs API calls (who did what on AWS resources), not application logs or database query logs. CloudTrail shows "User X modified parameter group" — it doesn't show "Aurora query took 30 seconds because connections were exhausted." Wrong logging tool for this use case.
- **RDS Performance Insights**: Excellent tool for database performance analysis — shows top SQL queries, wait events, and database load. But it's a diagnostic tool for database-specific issues, not a cross-service correlation tool. It doesn't correlate ALB errors with ECS container failures with database issues.
- **Lambda for remediation**: Lambda can execute remediation actions, but raw Lambda functions lack: built-in step orchestration, error handling across steps, approval workflows, audit trails, and parameter management. SSM Automation provides all of these out of the box. Lambda is appropriate as a step within an SSM Automation document, not as the orchestrator.

## Recommendations

- **Alarm strategy**: Create composite alarms for user-impacting failure scenarios. Each scenario (connection exhaustion, disk full, memory exhaustion) gets its own composite alarm combining cause metrics with impact metrics.
- **Enable ALL database logs**: Slow query logs, general logs (in dev), error logs, audit logs. The storage cost is minimal; the debugging value during incidents is immeasurable.
- **Centralized logging hierarchy**: CloudWatch Logs (collection) → Firehose (streaming) → OpenSearch (analysis/dashboards). For simpler setups: CloudWatch Logs + Logs Insights is sufficient.
- **SSM Automation runbook library**: Build a library of remediation runbooks for known failure patterns. Each runbook should be tested monthly with FIS. Common runbooks: connection exhaustion, disk full, memory exhaustion, certificate expiration, unhealthy targets.
- **Post-incident automation**: After every incident, ask "could this remediation be automated?" If yes, create an SSM Automation runbook before closing the incident.

## Relevant Links

- [CloudWatch Composite Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html)
- [CloudWatch Application Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-application-insights.html)
- [SSM Automation](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html)
- [CloudWatch Logs Cross-Account](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CrossAccountSubscriptions.html)
- [Fault Injection Simulator](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
