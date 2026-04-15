# Q04: Global Accelerator with S3 Multi-Region Access Points and EFS/FSx Shared Storage

## Question

A company operates a global content management platform with two distinct storage challenges:

**Challenge 1 — Global asset distribution (S3)**
- Marketing teams in North America, Europe, and Asia upload and access 500 TB of media assets (images, videos, documents) stored in S3
- Current: Single bucket in us-east-1 with S3 Cross-Region Replication (CRR) to eu-west-1 and ap-southeast-1
- Problem: Applications must be coded with Region-specific bucket endpoints. When a new Region is added, all applications must be updated. Additionally, upload requests from Asia go to us-east-1 (single primary) — slow uploads and cross-Region data transfer costs
- Requirement: A single global endpoint that automatically routes reads AND writes to the nearest Region. Adding a new Region should NOT require application changes

**Challenge 2 — Shared file system for compute (EFS/FSx)**
- A data processing pipeline runs on 100 EC2 instances across 3 AZs in us-east-1
- Each instance processes files and writes results to a shared directory
- Current: Results written to instance-local EBS volumes, then synced to S3 via cron job every 10 minutes
- Problems: (a) If an instance terminates, unsynced results are lost (up to 10 minutes of work), (b) other instances can't read results until S3 sync completes, (c) multiple instances may process the same file because they can't see each other's in-progress markers
- Requirement: A shared POSIX file system that all 100 instances mount simultaneously, survives instance termination, and provides strong consistency for file locking

Which combination of services solves both challenges?

## Options

- **A.** Challenge 1: **S3 Multi-Region Access Points** with AWS Global Accelerator. Create an S3 Multi-Region Access Point spanning buckets in us-east-1, eu-west-1, and ap-southeast-1 (with CRR between them). Applications use the single Multi-Region Access Point ARN — Global Accelerator automatically routes each request to the nearest S3 bucket based on network latency. Writes are replicated via CRR. Adding a new Region (e.g., sa-east-1) requires adding a bucket and updating the MRAP configuration — zero application code changes. Challenge 2: **Amazon EFS** (Elastic File System) mounted across all 100 instances. EFS provides a POSIX-compliant shared file system with strong read-after-write consistency, file locking (`flock`/`fcntl`), and automatic replication across 3 AZs. Instances mount EFS via NFS. If an instance terminates, all written data is immediately available from other instances. EFS Throughput Modes: Elastic throughput for bursty workloads.
- **B.** Challenge 1: CloudFront with S3 origins in each Region. CloudFront routes to the nearest edge location, which fetches from the nearest S3 bucket. Challenge 2: Amazon FSx for Lustre linked to S3. All instances mount FSx for Lustre for high-performance shared access.
- **C.** Challenge 1: Route 53 latency-based routing with three S3 static website endpoints. Applications resolve the DNS name and access the nearest bucket. Challenge 2: EBS Multi-Attach (io2 volumes) shared across instances.
- **D.** Challenge 1: S3 Transfer Acceleration for fast uploads. Single bucket in us-east-1. Challenge 2: Amazon FSx for Windows File Server for POSIX file sharing.

## Answers

### A. S3 Multi-Region Access Points + EFS — ✅ Correct

**Challenge 1 — S3 Multi-Region Access Points (MRAP):**

- **What is S3 Multi-Region Access Point?**
  - A named, global S3 endpoint that spans buckets in multiple Regions. The MRAP ARN looks like: `arn:aws:s3::123456789012:accesspoint/mrap-name.mrap`
  - Applications use this single ARN (or the MRAP alias endpoint) for all S3 operations — `GetObject`, `PutObject`, `ListObjects`, etc.
  - AWS Global Accelerator (built into MRAP) routes each request to the S3 bucket with the lowest network latency from the client's location.

- **Read routing (GET)**:
  - A user in Tokyo issues `GetObject` → Global Accelerator routes to ap-southeast-1 bucket (nearest). A user in Frankfurt → eu-west-1 bucket. Automatic, no application logic needed.
  - If the nearest bucket doesn't have the object (CRR hasn't replicated yet), the request returns 404 — it does NOT fall back to another Region automatically. For this reason, CRR replication completion time matters for read consistency.

- **Write routing (PUT)**:
  - A user in Tokyo uploads a file → Global Accelerator routes the upload to ap-southeast-1 bucket (nearest). The upload enters the S3 bucket closest to the user — low latency, no cross-Region transfer on the upload path.
  - CRR then asynchronously replicates the object to us-east-1 and eu-west-1. With S3 Replication Time Control (RTC), 99.99% of objects replicate within 15 minutes.
  - This is fundamentally different from a single-primary architecture: writes flow to the nearest Region, not to a single Region.

- **Adding new Regions**:
  - Create a new S3 bucket in sa-east-1. Add it to the MRAP configuration. Set up CRR rules. The MRAP endpoint now includes sa-east-1 — applications using the MRAP ARN automatically start routing Brazilian users to sa-east-1. Zero code changes.
  - Without MRAP: applications must be updated with the new bucket name, new Region endpoints, and routing logic. With 50 microservices, this is a significant change management effort.

- **Failover**: MRAP supports active-active or active-passive routing. If a Region becomes unavailable, MRAP routes traffic to the next-nearest Region. Configure failover controls to set routing preferences (e.g., all traffic to us-east-1 with failover to eu-west-1).

- **Global Accelerator integration**: MRAP uses Global Accelerator's Anycast IP addresses. Clients connect to the nearest AWS edge location (200+ edge locations) and are routed over AWS's private backbone to the nearest S3 bucket. This reduces latency and avoids public internet routing.

**Challenge 2 — Amazon EFS:**

- **POSIX-compliant shared file system**:
  - EFS provides a standard NFS v4.1 file system. Linux instances mount it like any NFS share: `mount -t nfs4 fs-01234.efs.us-east-1.amazonaws.com:/ /mnt/shared`
  - Supports standard POSIX semantics: file locking (`flock`, `fcntl`), hard/soft links, permissions (`chmod`/`chown`), and directory operations.
  - **Strong consistency**: EFS provides read-after-write consistency for all operations. An instance that writes a file can immediately read it from any other instance — no replication lag or eventual consistency.

- **Multi-AZ replication**:
  - EFS automatically replicates data across all AZs in the Region. If an AZ fails, EFS continues serving from other AZs with zero data loss.
  - This is transparent — instances don't need to know which AZ the data is stored in. The mount target in each AZ handles routing.

- **Instance termination resilience**:
  - EFS data persists independently of EC2 instances. If an instance terminates mid-write, the partially written file may be incomplete, but all previously completed writes are safe on EFS.
  - Compare to EBS: an EBS volume is attached to ONE instance. If that instance terminates unexpectedly, data on the EBS volume may be lost (if termination deletes the volume) or inaccessible (if another instance must mount it).
  - EFS decouples storage from compute — a core reliability pattern from the "What to expect" text.

- **File locking for coordination**:
  - The processing pipeline uses `flock` (file locking) to mark files as in-progress. Instance A acquires a lock on `file-001.dat` → writes a lock file or uses advisory locking → Instance B sees the lock and skips `file-001.dat`. This prevents duplicate processing without external coordination (DynamoDB, SQS).
  - EFS advisory locks are released automatically if the instance holding the lock terminates (the NFS client timeout expires and the lock is released).

- **Throughput modes**:
  - **Elastic throughput** (recommended for bursty workloads): Automatically scales throughput up to 10 GiB/s for reads and 3 GiB/s for writes. You pay for actual throughput used. Ideal for the 100-instance processing pipeline with variable load.
  - **Provisioned throughput**: Fixed throughput regardless of storage size. Use when you need consistent high throughput.
  - **Bursting throughput**: Scales with storage size (50 MiB/s per TB base, burst to 100 MiB/s). Only suitable for small file systems with low sustained throughput needs.

- **EFS storage classes**: Standard (for frequently accessed files) and Infrequent Access (IA, for files accessed less than once every 14 days). EFS Lifecycle Management automatically moves cold files to IA — reducing storage cost by up to 92%.

### B. CloudFront + FSx for Lustre — ❌ Incorrect

- **CloudFront for S3 multi-Region routing**:
  - CloudFront caches content at edge locations but routes **origin fetches** to a specific S3 bucket (or origin group). With origin groups, CloudFront can failover from a primary to secondary S3 origin — but it doesn't route to the nearest of 3 origins based on latency.
  - CloudFront is optimized for **read-heavy** workloads (caching). For write operations (`PutObject`), CloudFront can use S3 Transfer Acceleration — but it uploads to a single Region, not the nearest of multiple Regions.
  - Adding a new Region as a CloudFront origin requires reconfiguring the distribution's origin settings — not as seamless as MRAP.

- **FSx for Lustre**:
  - FSx for Lustre is a high-performance file system designed for HPC, ML training, and media processing — workloads requiring > 100 GB/s throughput and sub-millisecond latency.
  - FSx for Lustre is linked to S3 — it can lazy-load objects from S3 on first access and write results back to S3. This is powerful for batch processing.
  - **However**: FSx for Lustre is deployed in a **single AZ** — it does NOT provide multi-AZ durability for persistent data. If the AZ fails, FSx for Lustre is unavailable. For the pipeline requiring AZ-failure resilience, EFS (multi-AZ) is more appropriate.
  - FSx for Lustre is the right choice for HPC workloads needing maximum throughput. For general shared file access with durability requirements, EFS is preferred.

### C. Route 53 latency routing + EBS Multi-Attach — ❌ Incorrect

- **Route 53 latency-based routing to S3 endpoints**:
  - S3 static website endpoints can be used with Route 53, but this only works for HTTP/HTTPS access (website hosting). S3 API operations (SDK `GetObject`, `PutObject`) use the S3 API endpoint, not the website endpoint.
  - Route 53 DNS resolution adds latency (DNS lookup + resolution + TTL caching). MRAP with Global Accelerator uses Anycast routing — no DNS resolution step, immediate routing to the nearest edge.
  - Route 53 resolves to a single Region per DNS query — the client connects to one Region. MRAP can transparently route individual requests to different Regions within the same connection session.

- **EBS Multi-Attach (io2)**:
  - EBS Multi-Attach allows a single io2 volume to be attached to up to 16 Nitro-based instances in the **same AZ**. Limitations:
    - **Same AZ only**: All instances must be in the same AZ as the volume. Cannot span AZs — violates the multi-AZ requirement.
    - **16 instance limit**: The pipeline has 100 instances — far exceeding the 16-instance Multi-Attach limit.
    - **No file system coordination**: Multi-Attach provides block-level access, not a file system. The instances must use a cluster-aware file system (GFS2, OCFS2) to avoid data corruption. Standard Linux file systems (ext4, xfs) cannot be used with Multi-Attach.
  - EBS Multi-Attach is designed for clustered databases and applications with built-in cluster-aware I/O — not for general shared file access.

### D. S3 Transfer Acceleration + FSx for Windows — ❌ Incorrect

- **S3 Transfer Acceleration**:
  - Transfer Acceleration uses CloudFront edge locations to accelerate uploads to a single S3 bucket. It does NOT route to the nearest of multiple buckets — all uploads go to the same (single) destination bucket.
  - Does not solve the "single global endpoint for reads AND writes across multiple Regions" requirement. Transfer Acceleration accelerates uploads; MRAP routes to the nearest Region.

- **FSx for Windows File Server**:
  - FSx for Windows provides SMB protocol (Windows file sharing) — NOT POSIX/NFS. Linux instances require NFS (POSIX semantics). While FSx for Windows can serve NFS via Linux SMB clients, this adds complexity and is not the intended use case.
  - For Linux POSIX file sharing, EFS is the native AWS service. For Windows SMB file sharing, FSx for Windows is correct.
  - FSx for Windows does support Multi-AZ deployments (active-standby) — but the POSIX requirement makes EFS the correct choice.

## Recommendations

- **S3 Multi-Region Access Points** are the recommended solution for applications that need a single global S3 endpoint with automatic nearest-Region routing. They simplify multi-Region S3 architectures significantly.
- **MRAP failover controls**: Configure active-active or active-passive routing. Active-active delivers lowest latency but requires CRR for data consistency. Active-passive simplifies consistency (single primary) but increases latency for non-primary Regions.
- **EFS for shared Linux storage**: EFS is the default choice for shared POSIX file systems on AWS. Use FSx for Lustre only when you need > 100 GB/s throughput (HPC/ML).
- **EFS mount targets**: Create mount targets in every AZ where your instances run. Each mount target gets an IP in the subnet — ensure your subnets have sufficient available IPs.
- **EFS performance**: For the 100-instance pipeline, Elastic throughput mode handles burst traffic without pre-provisioning. Monitor `PercentIOLimit` and `TotalIOBytes` CloudWatch metrics.
- **Storage decision matrix**:
  - Block storage (single instance): EBS
  - Shared POSIX (Linux, multi-AZ): EFS
  - High-performance parallel (HPC): FSx for Lustre
  - Windows SMB: FSx for Windows
  - NetApp ONTAP compatibility: FSx for ONTAP
  - Object storage (any scale): S3

## Relevant Links

- [S3 Multi-Region Access Points](https://docs.aws.amazon.com/AmazonS3/latest/userguide/MultiRegionAccessPoints.html)
- [Global Accelerator with MRAP](https://docs.aws.amazon.com/AmazonS3/latest/userguide/MultiRegionAccessPointBucketReplication.html)
- [Amazon EFS](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html)
- [EFS Throughput Modes](https://docs.aws.amazon.com/efs/latest/ug/performance.html)
- [EFS vs FSx](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/storage-services.html)
- [FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [EBS Multi-Attach](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes-multi.html)
