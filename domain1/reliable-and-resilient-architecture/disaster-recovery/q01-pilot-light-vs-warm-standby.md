# Q01: Pilot Light vs. Warm Standby

## Question

A global e-commerce company runs its primary production workload in **us-east-1**. The application uses Amazon EC2 instances behind an Application Load Balancer, Amazon RDS MySQL Multi-AZ for the database, and Amazon S3 for static assets. The company's RTO is **2 hours** and RPO is **15 minutes**. The company wants to minimize ongoing DR infrastructure costs while meeting these objectives.

Which disaster recovery strategy should the solutions architect recommend?

## Options

- **A.** Set up a multi-site active/active deployment in us-west-2 with identical infrastructure, using Amazon Route 53 latency-based routing to distribute traffic between both Regions.
- **B.** Configure a pilot light strategy in us-west-2 with a cross-Region read replica of RDS MySQL, AMIs copied to us-west-2, and launch templates ready. During failover, promote the read replica, launch EC2 instances from AMIs, and update Route 53.
- **C.** Set up a warm standby in us-west-2 with a scaled-down version of the production environment running at all times, including a smaller Auto Scaling group, a promoted RDS read replica, and Route 53 failover routing.
- **D.** Use AWS Backup to take daily snapshots of all resources and store them in us-west-2. During failover, restore all resources from the latest snapshots and update DNS manually.

## Answers

### B. Pilot Light — ✅ Correct

A pilot light strategy keeps only the essential core components running in the DR Region (e.g., a cross-Region RDS read replica replicating continuously). Compute resources are pre-configured (AMIs, launch templates) but **not running**, keeping costs minimal. With an RPO of 15 minutes (achievable via RDS cross-Region replication) and an RTO of 2 hours (sufficient time to launch EC2, promote the replica, and switch DNS), pilot light is the best fit.

### A. Multi-site active/active — ❌ Incorrect

Multi-site provides the lowest RTO/RPO (near-zero) but runs **full duplicate infrastructure** at all times. This far exceeds the requirements (2h RTO, 15m RPO) and is the most expensive option. The question specifically asks to minimize ongoing costs.

### C. Warm standby — ❌ Incorrect

Warm standby runs a scaled-down but **always-on** environment. While it would meet the RTO/RPO requirements, it costs significantly more than pilot light because compute resources run continuously. Since the 2-hour RTO gives enough time to launch resources from AMIs, warm standby is over-provisioned for this scenario.

### D. AWS Backup with daily snapshots — ❌ Incorrect

Daily snapshots give an RPO of up to **24 hours**, which violates the 15-minute RPO requirement. Additionally, restoring all resources from snapshots would likely exceed the 2-hour RTO due to the time needed to restore databases, launch instances, and configure networking.

## Recommendations

- **Pilot light** is ideal when RTO allows time to spin up resources (1–4 hours) and RPO requires near-continuous replication for data stores.
- Always match the DR strategy cost to the actual RTO/RPO requirements — don't over-provision.
- Use cross-Region RDS read replicas for low RPO; use AWS Backup/snapshots only when RPO tolerance is measured in hours.
- Pre-bake AMIs and use launch templates to minimize instance launch time during failover.

## Relevant Links

- [AWS Disaster Recovery Workloads](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)
- [Pilot Light DR Strategy](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/pilot-light.html)
- [Amazon RDS Cross-Region Read Replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html#USER_ReadRepl.XRgn)
- [AWS Well-Architected — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
