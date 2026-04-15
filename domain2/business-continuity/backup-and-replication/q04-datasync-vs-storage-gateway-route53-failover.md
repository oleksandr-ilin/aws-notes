# Q03: DataSync vs Storage Gateway and Route 53 Failover Configuration

## Question

A company is building a hybrid DR solution. Their on-premises data center has:

**Data replication requirement:**
- 50 TB NFS file share that must be replicated to AWS for DR
- Files change incrementally (~200 GB/day of new and modified files)
- Full copy needed initially, then ongoing incremental synchronization
- RPO < 4 hours for the file data
- On-premises applications must continue accessing files via NFS during replication

**Traffic failover requirement:**
- Production website runs on on-premises servers at IP 203.0.113.10
- DR environment in AWS: ALB in us-east-1
- On failure, traffic must shift from on-premises to AWS within 60 seconds
- The team wants Route 53 to detect failure automatically and reroute — no manual DNS changes
- During failover transition, some users will still have cached DNS pointing to the old IP — this window must be minimized

Which combination of data replication and traffic failover services meets both requirements?

## Options

- **A.** **Data**: AWS DataSync agent on-premises, scheduled task running every 2 hours to transfer changed files from the NFS share to S3 in us-east-1. DataSync uses incremental transfer — only changed bytes in modified files are transferred. **Traffic**: Route 53 failover routing policy — primary record pointing to on-premises IP (203.0.113.10) with a health check (HTTP/HTTPS to the on-prem website), secondary record pointing to the ALB in AWS. Set DNS TTL to 30 seconds. When the health check fails (3 consecutive failures at 10-second intervals = 30 seconds), Route 53 stops returning the primary record and returns the secondary (ALB) record. Clients with cached DNS will resolve the new record within 30 seconds (one TTL cycle).
- **B.** **Data**: AWS Storage Gateway (File Gateway) on-premises, presenting the NFS share backed by S3. Applications read/write via the File Gateway NFS mount point. Gateway caches hot files locally and asynchronously uploads to S3. **Traffic**: Route 53 weighted routing — 100% weight to on-premises, 0% to AWS. On failure, manually change weights to 0%/100%.
- **C.** **Data**: AWS Transfer Family (SFTP) for file transfer to S3. Nightly batch upload via cron job. **Traffic**: Route 53 latency-based routing with health checks. Primary in on-premises Region, secondary in us-east-1.
- **D.** **Data**: Direct Connect with an S3 VPC endpoint for file transfer. Custom rsync script running every hour. **Traffic**: Global Accelerator with endpoint groups for on-premises (Elastic IP) and AWS (ALB). Health check-based failover.

## Answers

### A. DataSync (scheduled incremental) + Route 53 failover routing (TTL 30s) — ✅ Correct

**Data Replication — AWS DataSync:**

- **DataSync agent**: A VM deployed on-premises (VMware, Hyper-V, or KVM) that connects to the NFS share and transfers data to AWS targets (S3, EFS, FSx).

- **Initial full copy**: DataSync performs an initial full transfer of 50 TB. With a 1 Gbps Direct Connect or VPN, initial sync takes ~5 days (50 TB ÷ 1 Gbps ≈ 111 hours). DataSync uses up to 10 Gbps bandwidth per task and parallelizes transfers across multiple streams.

- **Incremental synchronization**: DataSync compares source and destination file metadata (timestamps, sizes, checksums) and transfers only changed files. The 200 GB/day of changes takes ~25 minutes per sync at 1 Gbps. Scheduling every 2 hours provides sub-4-hour RPO.

- **On-premises NFS access continues**: DataSync READS from the NFS share — it doesn't modify or lock files. On-premises applications continue reading and writing to the NFS share normally during replication. DataSync captures changes on the next scheduled execution.

- **Transfer optimizations**: DataSync compresses data in-flight, validates data integrity (checksums), and supports bandwidth throttling to avoid saturating the network link.

- **DataSync vs Storage Gateway for this scenario**:
  - DataSync is for **data transfer and synchronization** — scheduled, batch-oriented, full and incremental copy. It's a replication tool.
  - Storage Gateway (File Gateway) is for **ongoing hybrid access** — it replaces or extends the local NFS share with S3 backing. Applications access files through the gateway.
  - The requirement says "on-premises applications must continue accessing files via NFS during replication" — they keep using their existing NFS share. DataSync copies data FROM this share TO S3 without replacing it. Storage Gateway would replace the NFS share with a gateway-backed mount point — a bigger architectural change.

**Traffic Failover — Route 53 Failover Routing:**

- **Failover routing policy**:
  - Primary record: `www.example.com` → `203.0.113.10` (on-premises) with associated health check
  - Secondary record: `www.example.com` → ALB DNS name (alias record in us-east-1)
  - Route 53 evaluates the health check. If healthy → returns the primary record (on-premises IP). If unhealthy → returns the secondary record (ALB).

- **Health check configuration**:
  - Protocol: HTTPS (or HTTP), checking the on-premises website endpoint
  - Request interval: 10 seconds (fast health check)
  - Failure threshold: 3 consecutive failures
  - **Detection time**: 3 failures × 10 seconds = **30 seconds** to mark unhealthy
  - **Total failover time** ≈ 30 seconds (detection) + up to 30 seconds (DNS TTL expiry on clients) = **< 60 seconds** for most clients

- **DNS TTL = 30 seconds**:
  - TTL controls how long DNS resolvers cache the record. With 30-second TTL:
    - Best case: Client's cached record expires immediately → resolves new IP instantly
    - Worst case: Client cached the record 1 second ago → waits 29 more seconds before re-resolving
  - Average failover experience: ~45 seconds from health check failure to new traffic routing.
  - **Why not TTL = 0**: Most DNS resolvers ignore TTL of 0 and cache for at least 30-60 seconds anyway. TTL = 30 is the practical minimum for reliable behavior.

- **Why failover routing and not other policies**:
  - **Failover**: Exactly 2 records (primary/secondary), deterministic behavior, health check-driven — designed for DR.
  - **Weighted**: Requires manual weight changes on failure — not automatic.
  - **Latency-based**: Routes to the lowest-latency endpoint — if on-premises is closer, it would route there even during partial failure. Not deterministic for DR.

### B. Storage Gateway + weighted routing (manual failover) — ❌ Incorrect

- **Storage Gateway (File Gateway)**: File Gateway presents an NFS mount point backed by S3. This is an ongoing hybrid storage solution — it replaces the existing NFS share with a gateway-backed share. Applications would need to remount to the gateway's NFS endpoint.
  - **Architectural change**: The requirement says applications must "continue accessing files via NFS during replication" — they want to keep their existing NFS infrastructure unchanged. File Gateway introduces a new NFS server (the gateway), requiring reconfiguration of all NFS clients.
  - **Continuous replication**: File Gateway uploads files to S3 asynchronously as they're written — this is continuous, not scheduled. RPO is typically minutes (time for the gateway to upload the latest writes). This is BETTER than the 4-hour RPO requirement but comes at the cost of architectural change.
  - **Use case clarity**: If the team is willing to migrate their NFS workflow to File Gateway, it provides superior RPO. If they want to keep the existing NFS share unchanged, DataSync is the non-disruptive choice.

- **Route 53 weighted routing with manual failover**: Weighted routing distributes traffic by percentage. Setting 100/0 weights means all traffic goes to on-premises. On failure, someone must manually update weights to 0/100.
  - **No automatic failover**: The requirement is "Route 53 to detect failure automatically and reroute — no manual DNS changes." Weighted routing doesn't react to health check failures — the weights are static until manually changed.
  - Failover routing is the correct policy for automatic DR switchover.

### C. Transfer Family (SFTP) + latency-based routing — ❌ Incorrect

- **AWS Transfer Family (SFTP)**: Transfer Family provides managed SFTP/FTPS/FTP servers backed by S3 or EFS. Files must be explicitly uploaded via SFTP protocol.
  - **Nightly batch**: A nightly cron job uploading to SFTP provides 24-hour RPO (worst case — disaster strikes just before the next nightly upload). This violates the < 4 hour RPO requirement.
  - **Not an NFS integrator**: Transfer Family doesn't mount NFS shares. A script would need to enumerate changed files, then upload via SFTP — custom development that DataSync does natively.
  - **File-level transfer**: SFTP transfers entire files. A 10 GB file with 1 KB of changes requires re-transferring 10 GB. DataSync transfers only changed bytes within files — far more efficient.

- **Latency-based routing**: Routes users to the endpoint with lowest network latency. If on-premises and AWS are in similar latency zones, Route 53 may oscillate between them unpredictably. Latency-based routing is for performance optimization, not for deterministic DR failover where "primary always wins unless unhealthy."

### D. Direct Connect + rsync + Global Accelerator — ❌ Incorrect

- **Custom rsync script**: rsync is a valid file synchronization tool, but:
  - Requires maintaining a custom script (error handling, logging, monitoring, retry logic).
  - rsync to S3 requires an intermediary (EC2 instance or AWS CLI sync) — rsync doesn't speak the S3 protocol natively.
  - DataSync provides all rsync functionality (incremental transfers, checksums, bandwidth throttling) as a managed service with built-in monitoring, scheduling, and CloudWatch integration.
  - "Custom rsync" is operational burden that DataSync eliminates.

- **Global Accelerator for on-premises failover**: Global Accelerator endpoints must be AWS resources (ALB, NLB, EC2 instances, Elastic IPs). An on-premises IP (203.0.113.10) cannot be a Global Accelerator endpoint. The on-premises server would need to be fronted by an AWS resource (NLB with a VPN/Direct Connect connection), adding complexity.
  - Route 53 failover routing can point to ANY IP address (including on-premises) — no AWS intermediary required.

## Recommendations

- **DataSync for scheduled file replication**: When the source NFS share must remain unchanged and the team wants a managed, scheduled sync tool. Schedule DataSync tasks at intervals that meet your RPO (e.g., every 2 hours for < 4 hour RPO).
- **Storage Gateway (File Gateway) for continuous hybrid access**: When applications should transparently use S3 as their primary storage through an NFS/SMB interface. Better RPO (minutes vs. hours) but requires architectural change.
- **DataSync vs Storage Gateway decision matrix**:
  - Keep existing NFS, replicate to AWS → **DataSync**
  - Replace existing NFS with S3-backed storage → **File Gateway**
  - Migrate data to AWS (one-time or periodic) → **DataSync**
  - On-premises applications need cloud-backed caching → **File Gateway**
- **Route 53 failover routing TTL**: Set TTL to 30-60 seconds for DR records. Lower TTL = faster failover but more DNS queries (higher Route 53 cost — $0.60/million queries for standard records).
- **Route 53 health check types**: Use HTTP/HTTPS with string matching (validate response body contains expected text) for more reliable failure detection than simple TCP checks.
- **Combine health checks with alarms**: Route 53 can use CloudWatch alarm-based health checks — the health check reflects the alarm state. This allows complex failure criteria (e.g., composite alarm combining CPU, error rate, and application-specific metrics).

## Relevant Links

- [AWS DataSync](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html)
- [AWS Storage Gateway File Gateway](https://docs.aws.amazon.com/filegateway/latest/filefsxw/what-is-file-fsxw.html)
- [DataSync vs Storage Gateway](https://docs.aws.amazon.com/datasync/latest/userguide/migrate-data.html)
- [Route 53 Failover Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
- [Route 53 Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-creating.html)
- [Route 53 DNS TTL](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-failover.html)
