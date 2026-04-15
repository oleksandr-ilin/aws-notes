# Q03: AWS Elastic Disaster Recovery (DRS) for Server Replication

## Question

A company runs 40 application servers (Windows Server 2019 and Amazon Linux 2) on EC2 in us-east-1. Each server runs a stateful application with locally-attached EBS volumes containing application data, configuration files, and transaction logs. The CISO requires:

- **RPO < 1 second** — virtually no data loss during Region failover
- **RTO < 30 minutes** — full application availability in eu-west-1 within 30 minutes of declaring a disaster
- **Continuous replication** — not snapshot-based (snapshots have RPO of hours)
- **Non-disruptive DR testing** — test failover monthly without impacting production
- **No agent-less option** — the team accepts installing a replication agent on each server
- **Cost-efficient standby** — the DR Region should NOT run full-scale production instances 24/7

Which DR solution meets all requirements?

## Options

- **A.** AWS Elastic Disaster Recovery (DRS). Install the AWS Replication Agent on each of the 40 source servers. DRS continuously replicates block-level data from EBS volumes to low-cost staging volumes (gp3) in eu-west-1. On failover, DRS launches recovery instances from the replicated volumes using a pre-configured launch template (instance type, VPC, subnet, security groups). Monthly DR drills use DRS's non-disruptive drill feature — it launches test instances from replicated data without affecting production replication. Failback is supported after the disaster is resolved.
- **B.** Cross-Region AMI copy on a scheduled basis (every 15 minutes). Use CloudFormation to launch instances from the latest AMIs in eu-west-1 during a disaster. Use EventBridge + Lambda to automate AMI creation.
- **C.** EBS snapshots with cross-Region copy every hour via AWS Backup. Create new EC2 instances from snapshots in eu-west-1 during disaster. Use CloudFormation with parameter overrides for the DR Region.
- **D.** Use AWS DataSync to continuously replicate EBS data to S3 in eu-west-1. On failover, restore S3 data to new EBS volumes, attach to pre-launched EC2 instances. Use S3 versioning for point-in-time recovery.

## Answers

### A. AWS Elastic Disaster Recovery (DRS) — ✅ Correct

DRS is the purpose-built service for this exact scenario:

- **Continuous block-level replication**:
  - The AWS Replication Agent installed on each source server captures every block-level write to the EBS volumes and replicates it to the DR Region over an encrypted connection.
  - Replication is continuous (not scheduled) — every disk write is replicated within seconds. This achieves **sub-second RPO** (the RPO is the replication lag, typically < 1 second for steady-state workloads).
  - In the DR Region, DRS maintains **staging area volumes** — low-cost EBS volumes (gp3) that store the replicated block data. These are NOT running EC2 instances — no compute cost during normal operations.

- **Fast recovery (< 30 minutes RTO)**:
  - On failover, DRS launches recovery instances from the latest replicated volumes. The process:
    1. DRS takes a point-in-time snapshot of the staging volumes
    2. Launches EC2 instances from the snapshot using pre-configured launch settings (instance type, VPC, subnet, security group, IAM role)
    3. Attaches the recovered volumes to the instances
    4. Instances boot the OS and application from the replicated disk — identical state to the source server at the time of failover
  - Total launch time: 5-20 minutes depending on volume size and instance type. Well within 30-minute RTO.

- **Non-disruptive DR drills**:
  - DRS drill feature launches test instances from the replicated data in an isolated VPC (or specified subnets). The drill does NOT interrupt, pause, or affect ongoing replication to the staging volumes.
  - The team validates: application functionality, data integrity, network connectivity, and performance. After testing, terminate the drill instances — replication continues uninterrupted.
  - Monthly drills provide documented evidence for compliance audits that DR works as designed.

- **Cost-efficient standby**:
  - During normal operation, the DR Region runs ONLY:
    - Staging area EBS volumes (gp3 at $0.08/GB/month) — e.g., 40 servers × 500 GB = 20 TB × $0.08 = $1,600/month
    - DRS replication server (small EC2 instance) — one replication server handles multiple source servers
  - No application-tier EC2 instances run until failover is declared. This is far cheaper than warm standby (scaled-down but running instances) or active-active (full duplicate infrastructure).

- **Failback after disaster resolution**:
  - Once us-east-1 is restored, DRS supports failback: reverse replication from eu-west-1 back to us-east-1, then cutover to return to the primary Region. This avoids manual data synchronization.

- **Supported OS**: Windows Server 2008+, Amazon Linux, Ubuntu, RHEL, CentOS, SUSE, Debian. Both the Windows and Linux servers are supported.

### B. Cross-Region AMI copy every 15 minutes — ❌ Incorrect

- **AMI copy timing**: Creating an AMI requires stopping the instance (for consistent snapshots) or taking an inconsistent snapshot (application data may be corrupted). Even with `--no-reboot`, AMI creation captures a point-in-time snapshot — not continuous replication.
- **15-minute AMI creation**: Each AMI copy takes 10-30 minutes to complete (snapshot + cross-Region copy). At 40 servers every 15 minutes, you're running 40 concurrent AMI copies — potentially throttled by EC2 API limits (image copy quota: 50 concurrent copies per Region).
- **RPO = 15 minutes at best**: The last replication was the most recent AMI copy. If disaster strikes 14 minutes after the last copy, 14 minutes of data is lost. This violates the < 1 second RPO requirement by orders of magnitude.
- **CloudFormation launch time**: Creating instances from AMIs via CloudFormation takes 10-20 minutes. But the AMI must also be fully copied to the DR Region first — if the latest copy was in progress when disaster struck, it may be incomplete.

### C. EBS snapshots via AWS Backup every hour — ❌ Incorrect

- **Hourly snapshots = 1-hour RPO**: AWS Backup can schedule EBS snapshots every hour. Cross-Region copy adds time. RPO is 1-2 hours — far exceeding the < 1 second requirement.
- **Snapshot restoration time**: Creating new EBS volumes from snapshots, launching instances, attaching volumes, and configuring networking takes 30-60 minutes. RTO may exceed 30 minutes for 40 servers.
- **No continuous replication**: Backup is inherently point-in-time. Between snapshots, all writes are at risk. DRS replicates continuously — every block write is captured.
- AWS Backup is the correct tool for backup-and-restore DR strategy (RPO/RTO of hours), not for sub-second RPO requirements.

### D. DataSync to S3 + restore to EBS — ❌ Incorrect

- **DataSync replicates files, not blocks**: DataSync copies files from NFS, SMB, HDFS, or self-managed storage to S3, EFS, or FSx. EBS block devices are not a DataSync source — you can't point DataSync at an EBS volume directly. You'd need to mount the file system and copy files, losing consistency for databases and transaction logs.
- **S3 to EBS restoration**: Restoring from S3 to EBS is not a native operation. You'd need to: launch an instance, create an EBS volume, mount it, copy data from S3 to the volume. For 20 TB across 40 servers, this takes hours — far exceeding 30-minute RTO.
- **DataSync scheduling**: DataSync tasks run on a schedule (minimum every hour for scheduled tasks). This is not continuous replication. RPO = time since last DataSync run.
- DataSync is designed for data migration and scheduled synchronization (e.g., on-premises NAS to S3), not for continuous DR replication of EC2 server volumes.

## Recommendations

- **AWS Elastic Disaster Recovery (DRS)** is the recommended service for server-level DR with sub-second RPO. It replaced CloudEndure Disaster Recovery as the AWS-native solution.
- **DRS vs. database-native replication**: DRS replicates at the block level — it doesn't understand the application or database. For databases, prefer database-native replication (Aurora Global Database, DynamoDB Global Tables, RDS read replicas) for application-consistent failover. Use DRS for application servers, not managed databases.
- **DRS staging area sizing**: Use the smallest EBS volume type (gp3) for staging volumes to minimize standby costs. During recovery, DRS can launch instances with different (larger/faster) volume types than the staging volumes.
- **Test quarterly at minimum**: DRS drill feature makes testing easy. Document drill results: time to launch, application validation, data integrity checks.
- **Network preparation**: Pre-configure VPC, subnets, security groups, and Route 53 records in the DR Region. DRS handles the server recovery — you handle the networking and DNS failover.
- **DRS pricing**: No upfront cost. Per-server replication fee (~$0.028/hour per server) + staging EBS storage costs. 40 servers ≈ $33.60/day in replication fees + storage.

## Relevant Links

- [AWS Elastic Disaster Recovery](https://docs.aws.amazon.com/drs/latest/userguide/what-is-drs.html)
- [DRS Architecture](https://docs.aws.amazon.com/drs/latest/userguide/architecture.html)
- [DRS Drill and Recovery](https://docs.aws.amazon.com/drs/latest/userguide/recovery-launch.html)
- [DRS Failback](https://docs.aws.amazon.com/drs/latest/userguide/failback.html)
- [DR Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)
