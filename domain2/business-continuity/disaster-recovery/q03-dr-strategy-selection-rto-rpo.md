# Q02: DR Strategy Selection Based on RTO/RPO Requirements

## Question

A company runs four business applications with different criticality levels. The CTO asks the solutions architect to design DR strategies that meet each application's SLA while minimizing overall DR cost.

| Application | RTO | RPO | Current Architecture | Monthly Revenue Impact of Downtime |
|---|---|---|---|---|
| Payment Processing | < 1 minute | Zero data loss | EC2 + Aurora in us-east-1 | $2M/hour |
| E-commerce Website | < 15 minutes | < 5 minutes | ECS Fargate + DynamoDB in us-east-1 | $500K/hour |
| Internal CRM | < 4 hours | < 1 hour | EC2 + RDS PostgreSQL in us-east-1 | $50K/hour |
| Monthly Reporting | < 24 hours | < 24 hours | EC2 + S3 in us-east-1 | $5K/day |

Which combination of DR strategies across eu-west-1 correctly matches cost to criticality?

## Options

- **A.** Payment Processing: **Multi-site active-active** (Aurora Global Database with write forwarding, Global Accelerator routing to both Regions, DynamoDB Global Tables for session state). E-commerce: **Warm standby** (scaled-down ECS cluster running in eu-west-1, DynamoDB Global Tables, Route 53 health check failover). CRM: **Pilot light** (RDS cross-Region read replica in eu-west-1, AMIs replicated via EC2 Image Builder, infrastructure defined in CloudFormation but not launched). Reporting: **Backup and restore** (AWS Backup cross-Region copy to eu-west-1, CloudFormation templates stored in S3 for rebuild).
- **B.** All four applications: **Warm standby** in eu-west-1 with different scale levels. Payment Processing at 100% scale, E-commerce at 50%, CRM at 25%, Reporting at 10%.
- **C.** Payment Processing: **Pilot light**. E-commerce: **Warm standby**. CRM: **Multi-site active-active**. Reporting: **Pilot light**.
- **D.** Payment Processing: **Multi-site active-active**. E-commerce: **Multi-site active-active**. CRM: **Warm standby**. Reporting: **Pilot light**.

## Answers

### A. Multi-site (Payment) + Warm standby (E-commerce) + Pilot light (CRM) + Backup-restore (Reporting) — ✅ Correct

Each DR strategy matches the application's RTO/RPO requirements and cost tolerance:

- **Payment Processing → Multi-site active-active (RTO < 1 min, RPO = 0)**:
  - **Only active-active achieves < 1 minute RTO with zero data loss.** Both Regions serve traffic simultaneously — there's no failover delay because the standby is already active.
  - **Aurora Global Database with write forwarding**: The primary Region handles writes; the secondary Region forwards writes to the primary. If the primary fails, Aurora Global Database promotes the secondary to primary in < 1 minute with typical RPO < 1 second.
  - **Global Accelerator**: Routes users to the nearest healthy endpoint. Detects primary Region failure in ~30 seconds and reroutes traffic to the secondary Region — total failover time < 1 minute.
  - **Cost justification**: At $2M/hour downtime cost, even minutes of outage exceed the annual cost of running a second active Region.

- **E-commerce → Warm standby (RTO < 15 min, RPO < 5 min)**:
  - **Warm standby** keeps a scaled-down but fully functional copy of the application running in eu-west-1. ECS services run at minimum task count (e.g., 2 tasks instead of 20).
  - **DynamoDB Global Tables**: Provides < 1 second replication lag — well within the 5-minute RPO. Global Tables replicate automatically; no manual failover needed for the data layer.
  - **Route 53 failover**: Health checks detect us-east-1 failure, DNS failover routes traffic to eu-west-1 within 1-2 minutes. ECS Auto Scaling ramps up to full capacity within 5-10 minutes. Total RTO ≈ 10-15 minutes.
  - **Cost**: Running a small ECS cluster 24/7 in the standby Region is moderate — much cheaper than multi-site active-active, but faster than pilot light.

- **CRM → Pilot light (RTO < 4 hours, RPO < 1 hour)**:
  - **Pilot light** keeps only the data layer running in eu-west-1. RDS cross-Region read replica continuously replicates data (RPO of minutes, well within 1 hour).
  - **Compute is NOT running**: EC2 instances are not launched. AMIs are replicated, and CloudFormation templates define the infrastructure — but nothing is provisioned until DR is activated.
  - **DR activation**: Promote RDS read replica to standalone (5-10 min), launch EC2 instances from AMIs via CloudFormation or Auto Scaling (10-20 min), update DNS. Total RTO ≈ 1-2 hours — well within 4-hour requirement.
  - **Cost**: Only RDS read replica + AMI storage + S3 for templates. No compute costs until DR is activated.

- **Reporting → Backup and restore (RTO < 24 hours, RPO < 24 hours)**:
  - **Cheapest DR strategy.** No infrastructure runs in eu-west-1. AWS Backup copies daily snapshots (EC2 AMIs, S3 data) cross-Region.
  - **DR activation**: Restore from backups in eu-west-1 using CloudFormation. Launch EC2, restore S3 data. Total RTO ≈ 4-8 hours for a simple reporting application.
  - **RPO = time since last backup**: Daily backups → worst-case 24-hour data loss. Acceptable for monthly reporting that can be re-run.
  - **Cost**: Only backup storage in eu-west-1 (~$0.025/GB/month). At $5K/day downtime impact, this is the most cost-efficient approach.

### B. Warm standby for all at different scales — ❌ Incorrect

- **Payment Processing at 100% warm standby**: 100% scale warm standby is essentially active-active in terms of cost, but without the active traffic routing — the standby is idle but fully provisioned. This wastes money (paying for 100% compute that sits idle) while still having a failover delay (DNS propagation, health check detection). Multi-site active-active costs the same but achieves < 1 minute RTO because traffic already flows to both Regions.
- **Reporting at 10% warm standby**: Paying for even 10% standby for a reporting app with 24-hour RTO tolerance is unnecessary. Backup and restore costs near-zero for storage-only DR.
- The "one strategy fits all" approach doesn't optimize cost-to-criticality. It either over-provisions for low-criticality apps or under-provisions for high-criticality ones.

### C. Pilot light for Payment + active-active for CRM — ❌ Incorrect

- **Pilot light for Payment Processing**: Pilot light requires 30-60 minutes to launch and configure compute infrastructure during DR activation. This exceeds the < 1 minute RTO by 30-60×. Payment processing at $2M/hour cannot tolerate this delay.
- **Multi-site active-active for CRM**: CRM has a 4-hour RTO and $50K/hour impact. Running full active-active infrastructure for a 4-hour-tolerant application is massive cost over-engineering. Pilot light achieves the same with 1/10th the cost.
- This option inverts the cost-criticality relationship — spending the most on the least critical application.

### D. Multi-site for both Payment + E-commerce — ❌ Incorrect

- **Multi-site active-active for E-commerce**: E-commerce has a 15-minute RTO — warm standby achieves this comfortably. Active-active costs 2× the infrastructure but only saves ~10 minutes of failover time for a $500K/hour application. The incremental cost of active-active (thousands/month in duplicate compute) exceeds the incremental downtime cost saved (~$80K for 10 minutes).
- **Pilot light for Reporting**: Pilot light keeps the data layer running — for a reporting app with 24-hour RTO and data in S3 (already durable), there's no need for a live database replica. Backup and restore is sufficient and cheaper.
- This over-provisions for E-commerce and slightly over-provisions for Reporting.

## Recommendations

- **DR strategy selection rule**:
  - RTO < 1 min, RPO ≈ 0 → **Active-active** (most expensive)
  - RTO < 15 min, RPO < 5 min → **Warm standby**
  - RTO < 4 hours, RPO < 1 hour → **Pilot light**
  - RTO < 24 hours, RPO < 24 hours → **Backup and restore** (cheapest)
- **Always calculate the cost of downtime vs. cost of DR infrastructure** — DR that costs more than the downtime it prevents is over-engineered.
- **Test DR regularly** with AWS Fault Injection Service — untested DR plans fail when needed. Test quarterly at minimum.
- **Aurora Global Database** is the gold standard for RDS-based multi-Region DR — sub-second replication, < 1 minute failover, automatic promotion.
- **DynamoDB Global Tables** provide the simplest multi-Region data layer — no manual replication configuration needed.

## Relevant Links

- [AWS DR Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [DynamoDB Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)
- [Route 53 Health Checks and Failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html)
- [AWS Backup Cross-Region Copy](https://docs.aws.amazon.com/aws-backup/latest/devguide/cross-region-backup.html)
