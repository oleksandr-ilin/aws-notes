# Q02: EventBridge Automated DR Response with Lambda and Route 53

## Question

A company runs a three-tier application in us-east-1 (Route 53 → ALB → ECS → Aurora). They have a warm standby in eu-west-1 (scaled-down ECS cluster, Aurora Global Database secondary). Currently, failover is manual — an operator must:

1. Detect the failure (check monitoring dashboards)
2. Promote Aurora Global Database secondary to standalone primary in eu-west-1
3. Scale up ECS services in eu-west-1
4. Update Route 53 to point to the eu-west-1 ALB
5. Notify stakeholders via email

Average manual failover time: 45 minutes. The CTO wants this reduced to < 5 minutes with full automation. However, automated failover must NOT trigger on transient issues — only on sustained failures confirmed by multiple signals.

Which automation architecture achieves reliable automated failover?

## Options

- **A.** Create a CloudWatch composite alarm that triggers only when ALL of the following sub-alarms are in ALARM state simultaneously: (1) Route 53 health check on us-east-1 ALB fails (3 consecutive checks at 10s = 30s), (2) ALB `HealthyHostCount = 0` for > 2 minutes, (3) Aurora writer instance status = "unreachable" for > 1 minute. The composite alarm triggers an EventBridge rule → Step Functions state machine that orchestrates: (a) Aurora Global Database planned failover via `failover-global-cluster` API, (b) ECS `UpdateService` to increase desired count in eu-west-1, (c) Route 53 `ChangeResourceRecordSets` to point to eu-west-1 ALB, (d) SNS notification to stakeholders. Each step includes error handling and automatic rollback if a step fails.
- **B.** Route 53 failover routing with automatic DNS failover based on health checks alone. No additional automation — Route 53 handles everything. Aurora uses auto-failover within the Region.
- **C.** CloudWatch alarm on ALB 5xx errors > 50% → EventBridge → Lambda function that immediately promotes Aurora in eu-west-1, scales ECS, and updates Route 53. No composite alarm — single signal trigger.
- **D.** AWS Fault Injection Simulator (FIS) experiment running continuously — when it detects a real failure (as opposed to an injected one), it triggers the failover automation. Use FIS guardrails to prevent accidental failover.

## Answers

### A. Composite alarm + EventBridge + Step Functions orchestration — ✅ Correct

This architecture provides reliable, automated, multi-signal failover:

- **CloudWatch Composite Alarm — multi-signal confirmation**:

  A composite alarm uses Boolean logic (AND/OR) across multiple sub-alarms. It enters ALARM state only when the defined condition is met:

  ```
  ALARM("Route53-HealthCheck-Failed") 
  AND ALARM("ALB-NoHealthyHosts") 
  AND ALARM("Aurora-Writer-Unreachable")
  ```

  - **Why multiple signals**: A single metric (e.g., 5xx errors) can spike due to a bad deployment, a DDoS attack, or a transient AWS issue — none of which warrant cross-Region failover. Requiring ALL three signals confirms a genuine, sustained infrastructure failure:
    - Route 53 health check fails → external connectivity to the ALB is broken
    - ALB has 0 healthy hosts → all ECS tasks/EC2 instances behind the ALB are unhealthy
    - Aurora writer unreachable → database is down
  - All three simultaneously = us-east-1 is genuinely unavailable. This eliminates false-positive failovers.

  - **Timing**: Route 53 health check detects in 30 seconds. ALB healthy host count alarm with 2-minute evaluation ensures it's not a momentary blip. Aurora status check confirms database failure. Total detection: ~2 minutes.

- **EventBridge Rule — event routing**:
  - The composite alarm state change (`ALARM`) generates a CloudWatch alarm state-change event.
  - An EventBridge rule pattern matches this specific composite alarm:
    ```json
    {
      "source": ["aws.cloudwatch"],
      "detail-type": ["CloudWatch Alarm State Change"],
      "detail": {
        "alarmName": ["DR-Failover-Composite-Alarm"],
        "state": {"value": ["ALARM"]}
      }
    }
    ```
  - EventBridge invokes the Step Functions state machine as the target.

- **Step Functions — orchestrated failover**:

  Step Functions orchestrates the failover sequence with built-in error handling, retries, and timeout:

  **Step 1: Promote Aurora Global Database** (~1-2 minutes)
  - API: `rds:FailoverGlobalCluster` (planned managed failover)
  - Aurora detaches the secondary Region cluster, promotes it to a standalone cluster with read/write capability
  - The secondary already has near-real-time data (Aurora Global Database replication lag < 1 second typical)
  - On failure: Alert team, halt automation (don't proceed to DNS change without database)

  **Step 2: Scale ECS in eu-west-1** (~30 seconds)
  - API: `ecs:UpdateService` with increased `desiredCount` (e.g., from 2 to 20 tasks)
  - ECS Fargate launches additional tasks from the existing task definition. With pre-pulled container images (via ECR replication), tasks start in 15-30 seconds.
  - On failure: Retry 3 times with exponential backoff

  **Step 3: Update Route 53** (~10 seconds)
  - API: `route53:ChangeResourceRecordSets` — update the A/AAAA record (or alias record) to point to the eu-west-1 ALB
  - With TTL set to 30 seconds, most clients resolve the new endpoint within 30-60 seconds
  - On failure: Retry (DNS update is idempotent)

  **Step 4: Notify stakeholders** (~1 second)
  - SNS publish to a topic subscribed by: ops email, Slack channel (via Chatbot), PagerDuty webhook
  - Include: timestamp, which alarms triggered, which failover steps completed/failed

  **Total automated failover time**: 2 minutes (detection) + 2 minutes (Aurora promotion) + 30 seconds (ECS scaling) + 10 seconds (DNS) = **< 5 minutes**.

- **Error handling and rollback**:
  - Step Functions supports `Catch` and `Retry` blocks on each step. If Aurora promotion fails, the state machine can: (a) alert the on-call engineer, (b) skip DNS change (don't route traffic to a Region with no database), (c) log the failure for post-incident analysis.
  - Each step is idempotent — running the state machine twice doesn't cause inconsistency.

### B. Route 53 failover routing alone — ❌ Incorrect

- **Route 53 failover routing handles DNS only**: Route 53 can automatically switch DNS from the primary (us-east-1 ALB) to secondary (eu-west-1 ALB) based on health checks. But it does NOT:
  - Promote Aurora Global Database secondary (database is still read-only in eu-west-1)
  - Scale up ECS services (warm standby is scaled down — 2 tasks can't handle production traffic)
  - Notify stakeholders
- With Route 53 failover alone, traffic arrives at eu-west-1 but: the database is read-only (all writes fail), and the 2 ECS tasks are overwhelmed by production traffic. The application is "available" but completely broken.
- **Aurora within-Region failover**: Aurora Multi-AZ failover only works within a Region. If the entire Region is down, within-Region failover doesn't help — cross-Region promotion is needed.
- Route 53 DNS failover is one component of the solution, not the entire solution.

### C. Single alarm (5xx > 50%) → Lambda — ❌ Incorrect

- **Single-signal trigger**: ALB 5xx errors > 50% can occur due to:
  - A bad application deployment (the app returns 500s but infrastructure is healthy)
  - A DDoS attack (the app is overwhelmed but can recover after the attack)
  - A single unhealthy task/instance (one container crashes, temporarily spiking error rate)
  - None of these warrant a cross-Region failover. They're resolved by: rolling back the deployment, scaling up, or replacing the unhealthy task.
- **False-positive failover risk**: Triggering cross-Region failover on a transient 5xx spike causes: unnecessary Aurora promotion (which is disruptive — you must re-establish the Global Database after promotion), unnecessary cost increase (production-scale ECS in eu-west-1), and unnecessary client disruption (DNS change, potential connection drops).
- **Lambda vs. Step Functions**: A single Lambda function for the entire failover sequence is fragile — if it times out (15-minute max) during Aurora promotion (which can take 1-3 minutes), the DNS step never executes. Step Functions handles long-running orchestrations with built-in state management, retries, and error handling. Lambda is appropriate for individual steps, not for the orchestration.

### D. FIS for real failure detection — ❌ Incorrect

- **FIS is for injecting failures, not detecting them**: FIS intentionally creates failures (stop instances, throttle API calls, inject CPU stress) in a controlled experiment. It doesn't monitor for real failures.
- **FIS guardrails**: Guardrails define stop conditions for experiments (e.g., "stop if error rate > 10%"). They protect against experiments causing too much damage — they don't trigger failover automation.
- **Correct use of FIS**: Use FIS to TEST the failover automation — inject a simulated Region failure and verify the composite alarm → EventBridge → Step Functions pipeline works correctly. But FIS is the testing tool, not the detection tool. CloudWatch alarms + EventBridge are the detection and response tools.

## Recommendations

- **Composite alarms are essential** for automated DR — never trigger cross-Region failover on a single metric. Require 2-3 independent signals confirming the failure is genuine and sustained.
- **Step Functions over Lambda** for failover orchestration — Step Functions provides: visual execution history, built-in retries with exponential backoff, parallel execution branches, timeouts per step, and error handling with catch blocks. Lambda is a step within the state machine, not the orchestrator.
- **Aurora Global Database failover types**:
  - **Planned managed failover** (`failover-global-cluster`): For controlled failover with zero data loss. Takes 1-3 minutes. Use when you can detect the failure and React promptly.
  - **Unplanned failover** (`failover-global-cluster --allow-data-loss`): For when the primary is completely unreachable. May lose recent transactions (seconds of data). Use when the primary Region is truly unavailable.
- **Test the automation monthly**: Use FIS to simulate a Region failure (stop ECS tasks, block Aurora endpoint), verify the composite alarm fires, and confirm Step Functions completes all steps. Review execution logs.
- **Failback procedure**: After disaster recovery, plan the failback — re-establish Aurora Global Database replication back to us-east-1, scale ECS back, update DNS. Automate this as a separate Step Functions workflow.

## Relevant Links

- [CloudWatch Composite Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html)
- [EventBridge Rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rules.html)
- [Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)
- [Aurora Global Database Failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-disaster-recovery.managed-failover)
- [Route 53 Failover Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
