# Q03: EBS Cost Optimization — Orphaned Volumes, Snapshot Lifecycle, and gp2 to gp3 Migration

## Question

A company's monthly EBS bill is $42,000. An audit reveals three cost waste categories:

**Problem 1 — Orphaned EBS volumes ($8,400/month)**
- 280 EBS volumes in us-east-1 are in "available" state (not attached to any instance)
- Root cause: Auto Scaling group launches create EBS volumes, but when instances terminate, the volumes persist because `DeleteOnTermination` is set to `false` on additional volumes (not root volumes)
- Some volumes contain data that teams might need (investigation logs, debug dumps) — the team can't simply delete all of them
- Requirement: Identify truly unused volumes, archive their data cheaply, then delete the volumes

**Problem 2 — Snapshot sprawl ($6,200/month)**
- 15,000 EBS snapshots totaling 400 TB (after deduplication)
- Many snapshots are from volumes that no longer exist (AMI was replaced, instance was terminated)
- No lifecycle policy — snapshots accumulate indefinitely
- Requirement: Keep snapshots < 30 days for active volumes, archive snapshots 30-90 days old, delete snapshots > 90 days. Critical volumes (tagged `backup:critical`) must keep snapshots for 1 year

**Problem 3 — gp2 volumes still in use ($12,000/month overhead)**
- 500 EBS volumes are still gp2 (legacy default). gp3 provides the same baseline performance (3,000 IOPS, 125 MB/s) at 20% lower cost
- Many gp2 volumes are 500 GB+ but only need 50 GB of storage — they were sized large for IOPS (gp2 IOPS scales with size: 3 IOPS/GB)
- Requirement: Migrate to gp3 and right-size volumes to actual data needs

Which strategy addresses all three problems?

## Options

- **A.** Problem 1: Use **AWS Config rule** `ec2-volume-inuse-check` to identify unattached volumes. For volumes unattached > 30 days, create EBS snapshots → move snapshot data to **S3 Glacier Deep Archive** via snapshot archive (EBS Snapshot Archive) → delete the original volume. Tag volumes with `LastAttachedDate` using CloudTrail + Lambda automation. Problem 2: Implement **Amazon Data Lifecycle Manager (DLM)** policies: (a) Default policy: retain snapshots 30 days, then archive (EBS Snapshot Archive tier at $0.0125/GB/month vs. $0.05/GB standard), delete after 90 days. (b) Critical policy: retain snapshots for 365 days (tag filter: `backup:critical`). Apply retroactively — DLM can't delete existing snapshots, so run a one-time Lambda to clean up old snapshots matching the criteria. Problem 3: Migrate gp2 to **gp3** using `ModifyVolume` API (no downtime, no detachment). Right-size: for volumes needing only 3,000 IOPS, reduce size to actual data usage (gp3 provides 3,000 IOPS regardless of size). Use CloudWatch `VolumeReadOps` + `VolumeWriteOps` to verify actual IOPS needs before migration.
- **B.** Problem 1: Delete all "available" volumes immediately — if data was important, teams would have attached them. Problem 2: Delete all snapshots older than 30 days. Problem 3: Replace gp2 with io2 for better performance.
- **C.** Problem 1: Stop all Auto Scaling groups to prevent new orphaned volumes. Manually review each volume. Problem 2: Disable automatic snapshots. Keep only manual snapshots. Problem 3: Use gp2 but reduce volume sizes (IOPS will decrease proportionally).
- **D.** Problem 1: Use Trusted Advisor to find idle volumes. Snapshot everything to S3 Standard. Delete volumes. Problem 2: Use AWS Backup with retention rules for snapshot lifecycle. Problem 3: Use gp3 with Provisioned IOPS for all volumes.

## Answers

### A. Config + DLM + EBS Snapshot Archive + gp3 migration — ✅ Correct

**Problem 1 — Orphaned Volume Cleanup:**

- **AWS Config `ec2-volume-inuse-check`**: This managed Config rule evaluates all EBS volumes and marks any volume in "available" state (not attached) as `NON_COMPLIANT`. Provides a dashboard of all unattached volumes across the account.

- **CloudTrail + Lambda for tracking**:
  - CloudTrail logs every `DetachVolume` API call with timestamp. A Lambda function triggered by the CloudTrail event tags the volume with `LastDetachedDate = 2024-01-15`. This is critical metadata — it tells you HOW LONG the volume has been unattached.
  - Volumes unattached > 30 days are likely truly orphaned. Volumes unattached < 7 days may be in-progress work.

- **Safe archive workflow**:
  1. Identify volumes unattached > 30 days (Config + tag query)
  2. Create an EBS snapshot of each volume (preserves data)
  3. Archive the snapshot to **EBS Snapshot Archive tier**:
     - Standard snapshot storage: $0.05/GB/month
     - Archive tier: **$0.0125/GB/month** (75% cheaper)
     - Archive has a minimum 90-day retention
     - Restore from archive takes 24-72 hours (acceptable for rarely-needed investigation data)
  4. Delete the original EBS volume → **immediate savings** (no more volume charges)
  5. If a team needs the data later, restore the archived snapshot → create a new volume

- **Cost impact**: 280 volumes averaging 100 GB each = 28,000 GB.
  - Current: 28,000 GB × gp2 $0.10/GB = $2,800/month in volume charges + additional snapshot charges
  - After: Archived snapshots 28,000 GB × $0.0125/GB = $350/month. Savings: **$2,450/month**

- **Prevention**: Fix the root cause — set `DeleteOnTermination = true` on additional EBS volumes in the ASG launch template. This ensures volumes are deleted when instances terminate.

**Problem 2 — Snapshot Lifecycle with DLM:**

- **Amazon Data Lifecycle Manager (DLM)**:
  - DLM automates EBS snapshot creation and deletion based on lifecycle policies.
  - **Policy A (Default)**: Target volumes with tag `backup:standard` (or all volumes without `backup:critical`). Schedule: daily snapshots, retain 30 copies (30 days). After 30 days, snapshots are archived to EBS Snapshot Archive. After 90 days total, delete.
  - **Policy B (Critical)**: Target volumes with tag `backup:critical`. Schedule: daily snapshots, retain 365 copies (1 year). No archival (snapshots stay in standard tier for fast restore).

- **EBS Snapshot Archive for aged snapshots**:
  - DLM policies support automatic archival: after the standard retention period, move snapshots to the archive tier instead of deleting them.
  - Archive tier: $0.0125/GB/month (75% less than standard $0.05/GB).
  - Restore time: 24-72 hours. For disaster recovery of non-critical volumes, this is acceptable.

- **One-time cleanup**: DLM manages future snapshots but doesn't retroactively clean up 15,000 existing snapshots. Run a one-time Lambda script:
  1. List all snapshots (`describe-snapshots`)
  2. For each snapshot: check if the source volume exists, check the snapshot age, check tags
  3. Delete snapshots where: source volume doesn't exist AND age > 90 days AND tag ≠ `backup:critical`
  4. Archive snapshots 30-90 days old (move to archive tier)

- **Cost impact**: Reducing 400 TB of snapshots:
  - Delete snapshots > 90 days (estimated 70% of volume): 280 TB deleted → savings at $0.05/GB = $14,000/month
  - Archive 30-90 day snapshots: 80 TB × ($0.05 - $0.0125) = $3,000/month savings
  - Total snapshot savings: ~$5,000/month (from $6,200 to ~$1,200)

**Problem 3 — gp2 to gp3 Migration:**

- **gp2 vs. gp3 comparison**:

  | Feature | gp2 | gp3 |
  |---|---|---|
  | Baseline IOPS | 3 IOPS/GB (min 100, max 16,000) | **3,000 IOPS** (regardless of size) |
  | Baseline throughput | 128-250 MB/s (scales with size) | **125 MB/s** (regardless of size) |
  | IOPS scaling | Must increase volume size for more IOPS | Provision IOPS independently (up to 16,000) |
  | Throughput scaling | Tied to size | Provision independently (up to 1,000 MB/s) |
  | Price/GB | $0.10/GB | **$0.08/GB** (20% cheaper) |
  | Burst | Yes (burst credits for < 1 TB volumes) | No burst credits needed (baseline is higher) |

- **Why gp2 volumes are oversized**:
  - To get 3,000 IOPS on gp2, you need a 1,000 GB volume (3 IOPS/GB × 1,000 GB = 3,000 IOPS).
  - On gp3, a **100 GB volume** provides 3,000 IOPS at baseline — no oversizing needed.
  - A 1,000 GB gp2 volume: $100/month. A 100 GB gp3 volume with 3,000 IOPS: $8/month + $0/month (3,000 IOPS is free baseline) = **$8/month**. Savings: **92%** per volume.

- **Migration process**:
  - Use `ModifyVolume` API to change type from gp2 to gp3: **no downtime, no detachment, no data copy**. The modification happens live.
  - Volume enters "optimizing" state for a few hours — performance is maintained during this period.
  - **Right-sizing**: After converting to gp3, reduce volume size if actual data is smaller. However, EBS volumes cannot be shrunk in-place — to reduce size, you must: create a snapshot → create a smaller gp3 volume from snapshot → replace the volume. This requires brief downtime.
  - **Alternative**: Convert to gp3 (no downtime) but keep existing size. Even without right-sizing, the 20% per-GB price reduction saves money. Right-size during the next maintenance window.

- **IOPS verification before migration**: Check CloudWatch `VolumeReadOps` + `VolumeWriteOps` over 30 days. If a volume never exceeds 3,000 IOPS, gp3 baseline is sufficient. If a volume needs > 3,000 IOPS, provision additional IOPS on gp3 ($0.005/IOPS/month for IOPS above 3,000).

- **Cost impact**: 500 gp2 volumes averaging 500 GB:
  - Current: 500 × 500 GB × $0.10 = $25,000/month
  - After gp3 (same size): 500 × 500 GB × $0.08 = $20,000/month (20% savings = $5,000)
  - After gp3 + right-sizing (average 200 GB actual need): 500 × 200 GB × $0.08 = $8,000/month (68% savings = $17,000)

### B. Delete everything aggressively + io2 — ❌ Incorrect

- **Deleting all "available" volumes**: Some volumes may contain investigation data needed for compliance, security incidents, or debugging. Deleting without archiving violates data governance. The correct approach is: snapshot → archive → delete volume.
- **Deleting all snapshots > 30 days**: This violates the `backup:critical` requirement (1-year retention). Bulk deletion without checking tags risks destroying critical backup chains.
- **io2 instead of gp3**: io2 costs $0.125/GB + $0.065/IOPS — far more expensive than gp3 ($0.08/GB + free baseline 3,000 IOPS). io2 is for workloads needing > 16,000 IOPS or sub-millisecond latency — most web application volumes don't need this. Migrating from gp2 to io2 would increase costs.

### C. Stop ASGs + disable snapshots + gp2 resize — ❌ Incorrect

- **Stopping ASGs**: This stops creating instances (and thus volumes) — but also stops serving traffic. ASGs exist for a reason (handling load, replacing failed instances). The fix is setting `DeleteOnTermination = true`, not stopping the ASG.
- **Manually reviewing 280 volumes**: Not scalable. Config rules + Lambda automation handle this systematically. Manual review is error-prone and time-consuming.
- **Disabling automatic snapshots**: This eliminates backup protection entirely. Without snapshots, data loss from volume failure is unrecoverable. DLM lifecycle policies manage snapshot costs while maintaining backups.
- **Reducing gp2 volume size**: Reducing a gp2 volume from 500 GB to 100 GB reduces IOPS from 1,500 to 300 (3 IOPS/GB). This would severely degrade performance for most workloads. The gp2-to-gp3 migration solves this — gp3 provides 3,000 IOPS regardless of volume size.

### D. Trusted Advisor + AWS Backup + gp3 with Provisioned IOPS — ❌ Incorrect

- **Trusted Advisor for idle volumes**: Trusted Advisor does check for underutilized EBS volumes, but its checks are less comprehensive than Config rules. Trusted Advisor checks on a fixed schedule (weekly for Business/Enterprise support), while Config evaluates continuously on every configuration change.
- **Snapshot everything to S3 Standard**: EBS snapshots are already stored in S3 (managed by AWS — you don't see them). Moving snapshots to "S3 Standard" is not a valid operation. EBS Snapshot Archive is the correct tiered storage for snapshots — it stores snapshots at reduced cost within the EBS snapshot system.
- **AWS Backup for snapshot lifecycle**: AWS Backup can manage EBS snapshot retention, but DLM is the purpose-built tool for EBS snapshot lifecycle. AWS Backup adds value for cross-service backup management (EBS + RDS + EFS + DynamoDB). For EBS-only snapshot lifecycle, DLM is simpler and has native support for EBS Snapshot Archive tier.
- **gp3 with Provisioned IOPS for ALL volumes**: Most volumes need only the baseline 3,000 IOPS. Provisioning additional IOPS ($0.005/IOPS/month) for volumes that don't need it wastes money. Only provision additional IOPS for volumes where CloudWatch shows utilization exceeding the 3,000 baseline.

## Recommendations

- **Automate orphan detection**: Set up Config rule + EventBridge + Lambda to tag volumes on detachment and alert/archive after 30 days. Prevention is cheaper than cleanup.
- **DLM is essential**: Every production account should have DLM lifecycle policies. Snapshot sprawl is one of the most common hidden AWS costs.
- **gp3 as default**: Change the default EBS volume type in launch templates from gp2 to gp3. There is no scenario where gp2 is better than gp3 — gp3 is cheaper with equal or better performance.
- **EBS Snapshot Archive**: Use for snapshots you must retain but rarely access. 75% cost reduction with 24-72 hour restore time.
- **`DeleteOnTermination` audit**: Check all launch templates and launch configurations. Additional volumes (non-root) default to `DeleteOnTermination = false` — change this for volumes that don't contain persistent data.
- **Cost allocation tags**: Tag all EBS volumes with `Team`, `Environment`, `Application` tags. Use Cost Explorer to attribute EBS costs to specific teams — creating accountability for orphaned resources.

## Relevant Links

- [EBS Snapshot Archive](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html)
- [Data Lifecycle Manager](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html)
- [gp2 to gp3 Migration](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-modify-volume.html)
- [EBS Volume Types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
- [Config ec2-volume-inuse-check](https://docs.aws.amazon.com/config/latest/developerguide/ec2-volume-inuse-check.html)
- [DeleteOnTermination](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html#preserving-volumes-on-termination)
