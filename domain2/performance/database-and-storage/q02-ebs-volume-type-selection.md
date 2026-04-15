# Q02: EBS Volume Type Selection and Storage Performance Optimization

## Question

A company runs three workloads on EC2 instances in us-east-1, each with distinct storage I/O patterns:

**Workload 1 — Relational Database (self-managed PostgreSQL)**
- 500 GB database files
- Random read/write I/O with 16 KB block size
- Requires 25,000 sustained IOPS with < 1 ms latency
- Current gp2 volume (1 TB, 3,000 baseline IOPS) is bottlenecked — database queries timeout

**Workload 2 — Video Rendering Pipeline**
- Processes 4K video frames sequentially
- Reads and writes large frames (2-50 MB) in sequential order
- Throughput-bound: needs 500 MB/s sustained read/write
- IOPS is irrelevant — the workload is sequential, not random
- Cost-sensitive: runs 12 hours/day

**Workload 3 — Boot volumes for 200 web servers (Auto Scaling group)**
- 50 GB each, contains OS + application binaries
- Burst read patterns during deployment (lots of reads for ~2 minutes)
- Low sustained I/O during normal operation (< 100 IOPS average)
- Cost is the primary concern — 200 volumes × unit price matters

Which EBS volume type should be selected for each workload?

## Options

- **A.** Workload 1: io2 Block Express (64,000 IOPS provisioned, 500 GB). Workload 2: st1 (throughput-optimized HDD, 2 TB). Workload 3: gp3 (3,000 IOPS baseline, 125 MB/s — no additional IOPS provisioned).
- **B.** Workload 1: gp3 with 16,000 IOPS provisioned and 1,000 MB/s throughput. Workload 2: io2 with 10,000 IOPS provisioned for consistent performance. Workload 3: gp2 (1 TB for higher baseline IOPS via burst credits).
- **C.** Workload 1: io2 Block Express (25,000 IOPS provisioned, 500 GB, sub-ms latency). Workload 2: st1 (throughput-optimized HDD, 2 TB for 500 MB/s throughput). Workload 3: gp3 (50 GB, default 3,000 IOPS, 125 MB/s baseline — sufficient for burst deploys).
- **D.** All three: gp3 with different IOPS/throughput provisioned. Workload 1: gp3 with 16,000 IOPS. Workload 2: gp3 with 1,000 MB/s. Workload 3: gp3 default.

## Answers

### C. io2 Block Express (DB) + st1 (Video) + gp3 default (Boot) — ✅ Correct

Each volume type is matched to the workload's I/O pattern:

- **Workload 1 → io2 Block Express (25,000 IOPS, 500 GB)**:
  - **Why io2 Block Express**: This workload needs 25,000 sustained IOPS with sub-millisecond latency. gp3 maxes out at 16,000 IOPS — insufficient. io2 Block Express supports up to 256,000 IOPS and 4,000 MB/s per volume with sub-millisecond latency.
  - **IOPS:GB ratio**: io2 Block Express supports up to 1,000 IOPS per GB. 500 GB × 1,000 = 500,000 max IOPS — provisioning 25,000 is well within limits.
  - **Sub-millisecond latency**: io2 Block Express provides single-digit microsecond latency for random I/O — critical for database workloads where query performance depends on I/O latency.
  - **Cost**: io2 pricing is $0.125/GB/month + $0.065/IOPS/month. 500 GB + 25,000 IOPS = $62.50 + $1,625 = $1,687.50/month. Expensive, but database performance directly impacts revenue.
  - **Why not gp3**: gp3 max is 16,000 IOPS — below the 25,000 requirement. gp3 also has higher latency for sustained random I/O compared to io2.

- **Workload 2 → st1 (throughput-optimized HDD, 2 TB)**:
  - **Why st1**: The workload is sequential (large frames read/written in order) and throughput-bound (500 MB/s needed). st1 is designed for exactly this: sequential, throughput-intensive workloads.
  - **st1 throughput**: Baseline throughput is 40 MB/s per TB. At 2 TB = 80 MB/s baseline, with burst up to 250 MB/s per TB = 500 MB/s maximum. For sustained 500 MB/s, use a larger volume (e.g., 4-8 TB) or RAID 0 across two st1 volumes.
  - **Cost**: st1 is $0.045/GB/month. 2 TB = $90/month. Compare to io2 for the same throughput: massively more expensive and unnecessary for sequential I/O.
  - **Why not io2**: io2 is optimized for random IOPS — paying for 10,000 IOPS when the workload is sequential wastes the IOPS capability while potentially still not achieving 500 MB/s throughput (io2 throughput depends on IOPS × block size).
  - **Important**: st1 cannot be used as a boot volume. The video rendering data is stored on an attached st1 volume, with the OS on a separate gp3 boot volume.

- **Workload 3 → gp3 (50 GB, default 3,000 IOPS, 125 MB/s)**:
  - **Why gp3 default**: Every gp3 volume gets 3,000 baseline IOPS and 125 MB/s regardless of size — even a 50 GB volume. This is sufficient for burst reads during deployment (2-minute spike) and minimal sustained I/O (< 100 IOPS average).
  - **Cost**: gp3 is $0.08/GB/month. 50 GB × 200 volumes = $800/month. No additional IOPS provisioned needed — the 3,000 baseline handles deployment bursts.
  - **gp3 vs. gp2**: gp2 baseline IOPS = 3 IOPS/GB. A 50 GB gp2 volume gets only 150 baseline IOPS (with burst credits to 3,000). After burst credits deplete (during prolonged deployment), performance drops to 150 IOPS. gp3 provides 3,000 IOPS regardless of size — more predictable.
  - **gp3 is 20% cheaper than gp2** at the same size ($0.08 vs. $0.10/GB/month). For 200 volumes: gp3 = $800/month, gp2 = $1,000/month.

### A. io2 Block Express + st1 + gp3 — ❌ Partially Incorrect

- Nearly identical to option C, but **io2 Block Express with 64,000 IOPS** is over-provisioned. The requirement is 25,000 IOPS. Provisioning 64,000 IOPS costs $4,160/month in IOPS alone vs. $1,625/month for 25,000. Over-provisioning by 2.5× wastes $2,535/month.
- The st1 and gp3 portions are correct.

### B. gp3 + io2 + oversized gp2 — ❌ Incorrect

- **gp3 with 16,000 IOPS for database**: gp3 max IOPS is 16,000 — below the 25,000 requirement. This volume literally cannot provide enough IOPS for the database workload.
- **io2 for video rendering**: io2 is optimized for random I/O (IOPS-intensive). Provisioning 10,000 IOPS for a sequential workload wastes the IOPS capability. Throughput with 10,000 IOPS × 256 KB block size = ~2,500 MB/s, but the workload uses 2-50 MB sequential blocks — io2's random I/O optimization doesn't help. st1 provides the needed throughput at 1/10th the cost.
- **gp2 1 TB for boot volumes**: 1 TB gp2 for a 50 GB boot volume wastes 95% of the storage capacity. gp2's baseline IOPS scale with size (3 IOPS/GB), so 1 TB = 3,000 IOPS baseline — but gp3 provides 3,000 IOPS at 50 GB. Using 20× more storage to get the same IOPS is wasteful: 1 TB gp2 = $100/month vs. 50 GB gp3 = $4/month per volume. Across 200 volumes: $20,000 vs. $800/month.

### D. gp3 for all workloads — ❌ Incorrect

- **gp3 with 16,000 IOPS for database**: Same problem as option B — gp3 max is 16,000 IOPS, below the 25,000 requirement.
- **gp3 with 1,000 MB/s for video**: gp3 max throughput is 1,000 MB/s — this works for the 500 MB/s requirement. However, gp3 at 1,000 MB/s costs $0.08/GB + $0.040/IOPS (for additional IOPS) + throughput provisioning. At the required volume size, st1 is significantly cheaper for sequential workloads.
- **One-size-fits-all limitation**: gp3 is an excellent general-purpose volume but it has ceilings (16,000 IOPS, 1,000 MB/s). Purpose-built volume types (io2 for high IOPS, st1 for throughput) exist specifically for workloads that exceed gp3's capabilities or where gp3 is cost-suboptimal.

## Recommendations

- **Volume type selection rule**:
  - Random I/O, high IOPS: **io2/io2 Block Express** (up to 256,000 IOPS)
  - General purpose, balanced: **gp3** (up to 16,000 IOPS, 1,000 MB/s)
  - Sequential reads/writes, throughput: **st1** (up to 500 MB/s)
  - Cold storage, infrequent access: **sc1** (lowest cost HDD)
- **gp3 is the default choice** unless the workload exceeds 16,000 IOPS or cost optimization for sequential access makes st1 better.
- **Always use gp3 over gp2** for new volumes — gp3 is 20% cheaper with deterministic performance (IOPS independent of volume size).
- **io2 Block Express** requires Nitro-based instances (R5b, R6i, C6i, M6i, etc.) — verify instance type compatibility.
- **EBS-optimized instances**: Ensure the EC2 instance type supports enough EBS bandwidth for the volume's IOPS/throughput. An m5.large (max 3,600 Mbps EBS bandwidth) can't fully utilize an io2 volume with 25,000 IOPS — use m5.4xlarge or larger.

## Relevant Links

- [EBS Volume Types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
- [io2 Block Express](https://docs.aws.amazon.com/ebs/latest/userguide/provisioned-iops-ssd-vol-type.html)
- [gp3 Volumes](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose-ssd-volumes.html)
- [st1 Throughput Optimized HDD](https://docs.aws.amazon.com/ebs/latest/userguide/hdd-vols.html)
- [EBS Performance](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-ec2-config.html)
