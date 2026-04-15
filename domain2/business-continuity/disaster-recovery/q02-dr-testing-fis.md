# Q02: DR Testing with AWS Fault Injection Simulator

## Question

A bank runs its loan processing platform on AWS with a warm standby DR configuration. The primary Region is us-east-1 and the DR Region is us-west-2. The architecture includes:
- Aurora Global Database (primary in us-east-1)
- ECS Fargate services behind ALB in both Regions
- Route 53 failover routing to the us-west-2 ALB on health check failure

The CTO wants quarterly DR tests that:
- Validate the full failover chain from detection to recovery
- Measure actual RTO (target: < 15 minutes) and RPO (target: < 1 minute)
- Do NOT impact production traffic during the test
- Test both infrastructure failover and application-level recovery (e.g., in-flight transactions)
- Produce an auditable report for regulators

Which DR testing strategy should the solutions architect recommend?

## Options

- **A.** Use AWS FIS to create an experiment that: (1) injects an Aurora Global Database managed planned failover to us-west-2, (2) stops ECS tasks in us-east-1 to trigger Route 53 health check failure, and (3) records start/end timestamps. Run the experiment during a low-traffic maintenance window. After validation, initiate a planned failback by promoting us-east-1 back to primary. Use CloudWatch Logs Insights queries to calculate actual RTO (time from first alarm to full traffic serving in us-west-2) and RPO (Aurora replication lag at failover time). Export FIS experiment results and CloudWatch metrics to S3 for the audit report.
- **B.** Perform a tabletop exercise with the operations team. Walk through the DR runbook verbally and identify gaps. Document findings in a shared document. No actual infrastructure changes are made.
- **C.** Manually delete the Aurora cluster in us-east-1 and observe whether the secondary promotes. Redirect Route 53 manually. Measure recovery time with a stopwatch. Recreate the primary cluster from backup after the test.
- **D.** Deploy a complete copy of the production environment in us-west-2 using CloudFormation. Run synthetic traffic against the copy to validate application behavior. Tear down the copy after testing. Do not test actual failover routing or database promotion.

## Answers

### A. FIS-driven controlled failover + measurements + audit trail — ✅ Correct

This is the comprehensive, low-risk approach:
- **FIS experiment**: Orchestrates multiple failure injections in a controlled, repeatable manner. FIS supports Aurora actions (`aws:rds:failover-db-cluster`), ECS actions (stop tasks), and network actions. The experiment can be templated and versioned.
- **Planned Aurora Global Database failover**: The managed planned failover feature promotes the secondary with zero data loss (RPO = 0 for planned). For unplanned failover testing, use `aws:rds:failover-db-cluster` on the global cluster — this measures actual replication lag at failover time.
- **Maintenance window**: Running during low-traffic hours minimizes customer impact. Route 53 failover routing means traffic shifts to us-west-2 ALB — if us-west-2 is already warm, users experience brief DNS TTL delay but service continues.
- **Observability-based measurement**: CloudWatch metrics capture exact timestamps — failover detection time, DNS propagation, application readiness in us-west-2. These produce auditable RTO calculations, not subjective "stopwatch" measurements.
- **FIS + CloudWatch export**: Experiment metadata (start, actions, outcomes) plus metric exports provide the documented evidence regulators need.

### B. Tabletop exercise only — ❌ Incorrect

Tabletop exercises are valuable for identifying runbook gaps and team alignment, but they do **not validate** the actual infrastructure. A verbally-reviewed runbook may have incorrect IAM permissions, misconfigured health checks, or stale AMIs that only surface during real execution. Tabletop should supplement, not replace, actual failover testing. No auditable technical evidence is produced.

### C. Manual destructive testing — ❌ Incorrect

- **Deleting the Aurora cluster** is destructive and risky — if the secondary fails to promote, production data could be lost.
- Manual DNS redirect adds human latency and error risk.
- Measuring with a stopwatch is imprecise and not auditable.
- Recreating the primary from backup takes hours and leaves the system in a degraded state during recovery.
- Aurora Global Database promotion is the correct mechanism — not cluster deletion.

### D. Parallel environment with synthetic traffic — ❌ Incorrect

This validates application behavior but **does not test** the actual failover chain: Route 53 health check → DNS cutover → Aurora promotion → application recovery. The real failure path is untested. The copy environment may also differ from production in subtle ways (different capacity, different data set). The most critical component — actual failover routing between Regions — is skipped entirely.

## Recommendations

- **AWS FIS** supports stop conditions — if the experiment causes unexpected impact, it automatically halts and rolls back. Set a CloudWatch alarm as a stop condition (e.g., error rate > 5% in the DR Region).
- **Combine DR test types**: Tabletop (quarterly, for runbook review) + FIS automated failover (quarterly, for infrastructure validation) + full-scale Game Day (annually, with team under time pressure).
- **Measure RTO components separately**: DNS propagation time + database promotion time + application startup time + health check pass time = total RTO. This identifies which component is the bottleneck.
- **RPO measurement**: Query Aurora's `AuroraGlobalDBReplicationLag` CloudWatch metric at the moment of failover. If lag was 200 ms, RPO = 200 ms.
- Store all DR test evidence in an **S3 bucket with Glacier lifecycle** — regulators may request historical test results spanning years.
- Consider using **FIS experiment templates in CloudFormation** to version-control your test definitions alongside your infrastructure.

## Relevant Links

- [AWS FIS](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
- [FIS Action Reference](https://docs.aws.amazon.com/fis/latest/userguide/fis-actions-reference.html)
- [Aurora Global Database Planned Failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-disaster-recovery.managed-failover)
- [DR Testing Best Practices](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/testing-disaster-recovery.html)
