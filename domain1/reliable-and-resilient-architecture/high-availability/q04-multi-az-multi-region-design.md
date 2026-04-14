# Q04: Multi-AZ and Multi-Region Architecture Design

## Question

A company runs a three-tier web application and wants to design for **99.99% availability**. The current architecture runs entirely in us-east-1 across two Availability Zones. Recent AZ-level outages have caused full application downtime because the database (single-AZ RDS MySQL) was in the affected AZ.

The solutions architect must redesign the architecture to survive **any single AZ failure** with zero downtime and **any single Region failure** with less than 30 minutes of downtime. Which design meets BOTH requirements with the LEAST cost?

## Options

- **A.** Deploy the application across **three AZs** in us-east-1 with RDS Multi-AZ, ALB across all three AZs, and Auto Scaling group spanning three AZs. For multi-Region, set up a warm standby in us-west-2 with Aurora Global Database.
- **B.** Deploy active/active across us-east-1 and us-west-2, each with three AZs, Aurora Global Database, and Route 53 latency-based routing. Both Regions handle production traffic.
- **C.** Deploy across two AZs in us-east-1 with RDS Multi-AZ. Set up a pilot light in us-west-2 with cross-Region RDS read replica and AMIs copied weekly.
- **D.** Deploy across three AZs in us-east-1 with Aurora Multi-AZ. Take hourly automated snapshots and copy to us-west-2. During Region failure, restore from snapshots.

## Answers

### A. Three AZs with warm standby — ✅ Correct

**AZ failure requirement:** RDS Multi-AZ automatically fails over to the standby in a different AZ (typically 60–120 seconds). ALB and Auto Scaling across **three AZs** ensures the application continues serving traffic even when one AZ is lost — the remaining two AZs absorb the load.

**Region failure requirement:** Warm standby in us-west-2 with Aurora Global Database provides sub-30-minute recovery. The warm standby has scaled-down but running infrastructure that can quickly scale up. Aurora Global Database promotion takes ~1 minute, and Auto Scaling handles capacity.

This is the **least cost** option among those that meet both requirements — warm standby costs less than active/active.

### B. Multi-Region active/active — ❌ Incorrect

Active/active meets both requirements easily but is **more expensive** than necessary. Running full production capacity in two Regions doubles the compute cost. The question asks for "LEAST cost" — warm standby achieves the <30 minute Region-failure RTO at a fraction of the cost.

### C. Pilot light with two AZs — ❌ Incorrect

Two problems: (1) Only two AZs — if one fails, the remaining AZ might not have enough capacity to handle full load, risking downtime. Three AZs provides better redundancy. (2) Pilot light with weekly AMI copy takes longer to recover than 30 minutes — launching instances, configuring networking, and scaling up from zero compute typically takes 1–4 hours.

### D. Three AZs with hourly snapshots — ❌ Incorrect

Three AZs handles the AZ-failure requirement, but hourly snapshot copy + restore from snapshot for Region failure **exceeds 30 minutes** for any non-trivial database. Restoring a large Aurora database from a snapshot can take 30–60+ minutes, plus compute provisioning time. Aurora Global Database (option A) is the correct approach for <30 minute Region RTO.

## Recommendations

- **Three AZs > Two AZs** for critical workloads. With two AZs, losing one AZ means the remaining AZ must handle 100% of traffic — it may not have enough capacity. With three AZs, losing one still leaves 67% capacity.
- **99.99% availability** typically requires multi-AZ + multi-Region protection.
- **RDS Multi-AZ** is for AZ failures (automatic). **Cross-Region read replicas / Aurora Global Database** is for Region failures (requires promotion).
- Match RTO requirements to DR strategy:
  - <30 min RTO → Warm standby or better
  - <5 min RTO → Active/active
  - Hours → Pilot light or backup/restore
- **ALB automatically distributes across healthy AZs** — no manual intervention needed when an AZ fails.

## Relevant Links

- [RDS Multi-AZ Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [Aurora Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [Auto Scaling across AZs](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-benefits.html)
- [AWS Availability Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)
