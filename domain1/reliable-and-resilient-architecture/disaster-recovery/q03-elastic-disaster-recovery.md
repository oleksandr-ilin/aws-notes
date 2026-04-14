# Q03: AWS Elastic Disaster Recovery

## Question

A healthcare company runs a legacy monolithic application on several Amazon EC2 instances with large attached Amazon EBS volumes containing patient records. The application uses an Amazon RDS for Oracle database. Regulatory requirements mandate that the company must be able to recover the full environment in a secondary Region within **4 hours** (RTO) with no more than **1 hour** of data loss (RPO). The operations team has limited experience with infrastructure-as-code tools.

Which approach provides the MOST operationally efficient DR solution? (Select ONE)

## Options

- **A.** Use AWS Elastic Disaster Recovery (DRS) to continuously replicate the EC2 instances and EBS volumes to the DR Region. Configure RDS cross-Region read replica for the Oracle database. During failover, launch recovery instances via the DRS console and promote the RDS replica.
- **B.** Create a CloudFormation template that defines the entire infrastructure. Take hourly EBS snapshots and copy them to the DR Region. During failover, deploy the CloudFormation stack using the copied snapshots.
- **C.** Use AWS Application Migration Service to perform a one-time migration of the servers to the DR Region. Keep the DR instances stopped unless a disaster occurs. Use AWS DMS for continuous database replication.
- **D.** Manually create AMIs of each EC2 instance nightly and copy them to the DR Region. Export the RDS database using mysqldump hourly and store in S3 with cross-Region replication.

## Answers

### A. AWS Elastic Disaster Recovery (DRS) — ✅ Correct

AWS Elastic Disaster Recovery continuously replicates EC2 instances and their attached EBS volumes at the block level to the DR Region using lightweight replication agents. This provides sub-second RPO for compute/storage. Combined with an RDS cross-Region read replica (which provides minutes-level RPO for the database), this solution meets both the 1-hour RPO and 4-hour RTO. DRS provides a simple console-based failover process, making it ideal for teams with limited IaC experience.

### B. CloudFormation with hourly snapshots — ❌ Incorrect

While CloudFormation provides repeatable infrastructure, this approach requires significant IaC expertise that the operations team lacks. Hourly EBS snapshots meet the RPO requirement, but restoring from snapshots and deploying a full CloudFormation stack can be time-consuming and error-prone, potentially exceeding the 4-hour RTO. Cross-Region snapshot copy also introduces delays.

### C. Application Migration Service with stopped instances — ❌ Incorrect

AWS Application Migration Service (MGN) is designed for **one-time migrations**, not ongoing DR. Stopped instances become stale immediately — any changes to the primary instances (OS patches, config updates, new volumes) won't be reflected. This approach provides no continuous replication for compute resources, defeating the purpose of DR.

### D. Nightly AMIs with mysqldump — ❌ Incorrect

Nightly AMIs give an RPO of up to **24 hours**, violating the 1-hour RPO. Using mysqldump for an Oracle RDS database is incorrect (it's a MySQL tool). Even if the correct export tool were used, logical exports of large databases are slow and don't provide point-in-time recovery. This is also highly manual and error-prone.

## Recommendations

- **AWS Elastic Disaster Recovery** is purpose-built for server-level DR with continuous block-level replication. It's the go-to for EC2-based workloads.
- DRS replicates at the block level — it captures **all** changes to EBS volumes without application-level agents.
- For databases, pair DRS (for compute) with **RDS cross-Region read replicas** (for data) for a complete DR solution.
- DRS provides drill/test capabilities without impacting production — use regular DR drills to validate your recovery process.
- Know that DRS replaces the older CloudEndure Disaster Recovery service.

## Relevant Links

- [AWS Elastic Disaster Recovery](https://docs.aws.amazon.com/drs/latest/userguide/what-is-drs.html)
- [AWS DRS — How It Works](https://docs.aws.amazon.com/drs/latest/userguide/how-drs-works.html)
- [RDS Cross-Region Read Replicas for Oracle](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/oracle-read-replicas.html)
- [DR Strategies on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)
