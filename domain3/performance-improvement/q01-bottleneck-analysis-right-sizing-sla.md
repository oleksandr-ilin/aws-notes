# Q01: Performance Bottleneck Analysis, Right-Sizing, and SLA/KPI Monitoring

## Question

A SaaS company's customer portal has a performance SLA: P95 page load time < 2 seconds. Over the past 3 months, the SLA has been breached 8 times. The architecture is: CloudFront → ALB → EC2 (m5.4xlarge ASG, 12 instances) → RDS MySQL (db.r5.4xlarge) + ElastiCache Redis (r5.xlarge, 3 nodes).

The team has identified performance data but can't pinpoint the root cause:

| Metric | Value | Team's Interpretation |
|---|---|---|
| CloudFront cache hit ratio | 32% | "Probably fine" |
| EC2 average CPU | 35% | "Instances are over-provisioned" |
| EC2 P99 CPU | 88% | Not monitored |
| RDS CPU | 25% | "Database is fine" |
| RDS `ReadIOPS` | 12,000 (gp2 limit: 12,000) | Not monitored |
| ElastiCache hit rate | 45% | "Normal" |
| ElastiCache `EngineCPUUtilization` | 75% (single node) | Not monitored |
| ALB `TargetResponseTime` P95 | 1.8 seconds | "Close to SLA limit" |
| CloudFront origin latency P95 | 1.9 seconds | Not correlated |

The CTO asks: (a) identify the actual bottlenecks causing SLA breaches, (b) recommend right-sizing changes, and (c) implement continuous performance monitoring against the SLA KPI.

Where are the real bottlenecks and what should be changed?

## Options

- **A.** **Bottleneck 1 — RDS IOPS saturation**: RDS `ReadIOPS` at 12,000 = gp2 volume IOPS ceiling. The database is NOT "fine" — it's maxing out disk IOPS. Queries queue waiting for I/O, increasing response time. **Fix**: Migrate RDS storage from gp2 to gp3 with provisioned IOPS (set 16,000 IOPS) or io2 if higher IOPS needed. **Bottleneck 2 — ElastiCache Redis hot key / single-threaded saturation**: `EngineCPUUtilization` at 75% on a single-threaded Redis engine means one core is 75% saturated. Redis can't use multiple cores — it will hit 100% on a single core while overall node CPU appears low. Plus, 45% cache hit rate means 55% of requests bypass cache and hit the database (contributing to IOPS saturation). **Fix**: Enable Redis cluster mode to distribute keys across shards. Review cache TTLs and cache key design to improve hit rate (target > 80%). **Bottleneck 3 — CloudFront cache hit ratio too low**: 32% hit rate means 68% of requests go to the origin (ALB → EC2 → RDS). **Fix**: Review `Cache-Control` headers, extend TTLs for cacheable content, use CloudFront Origin Shield to reduce origin requests, and configure cache key policies to normalize query strings. **Right-sizing**: EC2 average CPU is 35% but P99 is 88% — instances are spiky, not over-provisioned. Right-size by switching from m5.4xlarge to m6g.2xlarge (Graviton, 50% cost reduction) with MORE instances (16 instead of 12) to reduce per-instance P99 load. **SLA monitoring**: Create a CloudWatch dashboard with the SLA KPI: ALB `TargetResponseTime` P95. Set a CloudWatch Anomaly Detection alarm on P95 latency. Create a composite alarm: P95 latency > 1.5 seconds AND (RDS IOPS > 10,000 OR ElastiCache CPU > 60%) — early warning before SLA breach.
- **B.** The EC2 instances are over-provisioned (35% average CPU). Right-size by reducing from m5.4xlarge (16 vCPU) to m5.xlarge (4 vCPU) with fewer instances. This saves cost without addressing database or cache issues.
- **C.** Add more EC2 instances to reduce P99 CPU. Double the RDS instance size to db.r5.8xlarge. Add more ElastiCache nodes (same cluster mode disabled).
- **D.** Replace CloudFront with Global Accelerator for lower latency. Replace RDS MySQL with Aurora for higher performance. Replace ElastiCache Redis with DAX.

## Answers

### A. Fix IOPS saturation + Redis sharding + CloudFront cache optimization + Graviton right-sizing — ✅ Correct

**Bottleneck 1 — RDS IOPS Saturation (root cause of slow queries):**

- **The hidden bottleneck**: RDS CPU at 25% looks healthy, but `ReadIOPS` at 12,000 is the gp2 volume's maximum (gp2 IOPS = 3 × volume_GB, max 16,000; this volume is at its IOPS ceiling). When IOPS capacity is exhausted:
  - New I/O requests are queued by the OS
  - Query execution time increases (queries wait for disk reads)
  - This appears as slow queries in the application — P95 latency increases
  - The database CPU stays low because the CPU is idle WAITING for I/O (IO wait), not processing queries

- **How to verify**: Check the RDS CloudWatch metric `ReadLatency` (average time per read operation). If `ReadLatency` > 5 ms (typical gp3/io2 is < 1 ms), the volume is I/O constrained. Also check `DiskQueueDepth` — values > 2 indicate I/O queuing.

- **Fix**: Migrate from gp2 to **gp3** with provisioned IOPS:
  - gp3 baseline: 3,000 IOPS (regardless of size) — provisionable up to 16,000 IOPS
  - Set 16,000 IOPS on gp3: $0.08/GB + ($0.005 × 13,000 additional IOPS) = volume cost + $65/month for extra IOPS
  - If the workload grows beyond 16,000 IOPS: migrate to io2 (up to 256,000 IOPS)
  - Migration is online: `modify-db-instance --storage-type gp3 --iops 16000` — no downtime for storage modifications

**Bottleneck 2 — ElastiCache Redis Single-Threaded Saturation:**

- **Why 75% engine CPU is critical**: Redis is single-threaded for command execution. `EngineCPUUtilization` measures the utilization of that single thread. At 75%, Redis is spending 75% of its time executing commands — only 25% headroom before it saturates at 100% (at which point all commands queue).
  - Node-level CPU might show 15% (because the instance has 4 cores, and Redis uses only 1). The node appears underutilized — the engine thread is near saturation. This is a common monitoring trap.

- **Low cache hit rate (45%)** compounds the problem:
  - 55% of requests miss the cache and hit RDS → more RDS IOPS consumption → more I/O queuing → slower responses
  - A low hit rate means either: TTLs are too short (data evicted before reuse), cache keys are too specific (each request generates a unique key), or the wrong data is being cached

- **Fix — Redis cluster mode enabled**:
  - Distribute keys across multiple shards (each shard is a separate Redis process with its own thread). 3 shards × 1 primary = 3 independent threads processing commands.
  - Each shard handles ~33% of keyspace → `EngineCPUUtilization` drops from 75% to ~25% per shard.
  - Add read replicas per shard for read-heavy workloads.

- **Fix — Improve cache hit rate**:
  - Extend TTLs: If product data changes hourly, set TTL = 1 hour (not 5 minutes)
  - Normalize cache keys: `product:123:en-US` instead of `product:123:en-US:timestamp` — remove variable components that prevent cache reuse
  - Cache more aggressively: Cache database query results, not just individual objects. Cache the top 100 queries (which represent 80% of traffic)
  - Target: > 80% hit rate. At 80%, only 20% of requests hit RDS → IOPS drops from 12,000 to ~5,000 → no more saturation.

**Bottleneck 3 — CloudFront Cache Hit Ratio:**

- **32% is well below best practice** (60-95% for typical web applications):
  - `Cache-Control` headers may be missing or set to `no-cache` — CloudFront doesn't cache the response
  - Query string variation: `?timestamp=123456` in each request makes every request a unique cache key — 0% hit rate for that path
  - Cookie forwarding: If CloudFront forwards all cookies, each unique cookie combination is a separate cache key

- **Fixes**:
  - Set `Cache-Control: public, max-age=3600` on cacheable responses (product pages, images, CSS/JS)
  - Configure CloudFront cache policy to ignore query strings that don't affect content (e.g., UTM tracking parameters)
  - Use **Origin Shield**: Extra caching layer between CloudFront edges and the origin. All edge locations fetch from Origin Shield (single location) → origin receives fewer requests → reduced ALB/EC2/RDS load. Particularly effective for globally distributed traffic.
  - Improving from 32% to 70% hit rate reduces origin requests by 56% → significant load reduction across the entire stack.

**Right-Sizing Analysis:**

- **Average CPU 35% ≠ over-provisioned**: P99 CPU at 88% means during peak moments, instances are near saturation. If you downsize (fewer vCPUs), P99 exceeds 100% → requests queue → latency spikes → SLA breaches.
- **Correct approach**: Switch from m5.4xlarge (16 vCPU, x86) to m6g.2xlarge (8 vCPU, Graviton). Graviton provides ~25% better performance per vCPU → 8 Graviton vCPUs ≈ 10 x86 vCPUs. Then increase instance count from 12 to 16 → more instances share the load → P99 per instance drops from 88% to ~55%.
- **Cost**: 12 × m5.4xlarge ($0.768/hr) = $9.22/hr → 16 × m6g.2xlarge ($0.308/hr) = $4.93/hr. **46% cost reduction** with better P99 performance.

**SLA/KPI Monitoring:**

- **KPI definition**: SLA is P95 page load < 2 seconds. The proxy metric is ALB `TargetResponseTime` P95 (server-side processing time). Real user latency includes network + client rendering — measure with CloudWatch RUM for true end-user experience.
- **Early warning composite alarm**: Alert when P95 approaches 1.5 seconds (75% of SLA limit) AND a contributing metric (IOPS, cache CPU, cache hit rate) is degraded. This gives the team time to act BEFORE the SLA breach.

### B. CPU-based right-sizing only — ❌ Incorrect

- Reducing to m5.xlarge (4 vCPU) would push P99 CPU well above 100% — requests would queue continuously. The "35% average CPU" metric masks the P99 spikes.
- Worse: this doesn't address the actual bottlenecks (RDS IOPS, ElastiCache saturation, CloudFront hit rate). Even if EC2 were perfectly sized, the database and cache bottlenecks would still cause SLA breaches.
- Right-sizing based on average utilization is a common mistake. Always check P95/P99 utilization before downsizing.

### C. Scale up everything — ❌ Incorrect

- **More EC2 instances**: EC2 isn't the bottleneck — the 1.8s P95 latency is caused by slow RDS queries and cache misses, not EC2 processing. Adding instances doesn't fix slow database queries.
- **Double RDS instance size** (r5.8xlarge): Doubling CPU and memory doesn't fix IOPS saturation. The bottleneck is disk I/O, not CPU or memory. A larger instance with the same gp2 volume hits the same 12,000 IOPS ceiling.
- **More ElastiCache nodes (cluster mode disabled)**: In cluster mode disabled, adding nodes means adding read replicas. Replicas help with read distribution but don't fix the single-threaded engine saturation on the primary. The primary still handles all writes and most command processing. Cluster mode (sharding) is needed to distribute the command processing load.

### D. Replace services — ❌ Incorrect

- **Global Accelerator instead of CloudFront**: Global Accelerator optimizes network routing (TCP/UDP) but doesn't cache content. Replacing CloudFront removes caching entirely — 100% of requests hit the origin. This increases load on every downstream component and worsens latency.
- **Aurora instead of RDS MySQL**: Aurora can deliver higher throughput than RDS MySQL (up to 5× according to AWS), but the bottleneck is EBS IOPS, not MySQL engine throughput. Aurora uses a different storage architecture (distributed, SSD-backed) that may resolve the IOPS bottleneck, but it's a larger migration than simply changing the EBS volume type (which fixes the same problem with zero downtime).
- **DAX instead of ElastiCache**: DAX is for DynamoDB — it cannot cache RDS MySQL query results. The database is RDS MySQL, not DynamoDB. ElastiCache Redis (properly configured with cluster mode and tuned TTLs) is the correct caching solution.

## Recommendations

- **Monitor P99/P95, not averages**: Average metrics hide performance problems. A service at 35% average CPU can have P99 at 90%. Always create dashboards showing percentile metrics.
- **EBS IOPS monitoring is essential**: Create CloudWatch alarms on `ReadIOPS` and `WriteIOPS` at 80% of volume max. IOPS saturation is a silent killer — CPU looks fine, queries are slow.
- **Redis `EngineCPUUtilization` vs `CPUUtilization`**: Monitor `EngineCPUUtilization` (single-thread usage), NOT `CPUUtilization` (overall node CPU). The engine metric is what matters for Redis performance.
- **CloudFront cache hit ratio baseline**: Target > 60% for API responses, > 90% for static assets. Monitor `CacheHitRate` metric. Low hit rates indicate misconfigured cache policies or non-cacheable responses.
- **Composite alarms for SLA monitoring**: Combine the SLA metric (P95 latency) with leading indicators (IOPS, cache CPU, cache hit rate) in a composite alarm. Alert on the leading indicators before the SLA breaches.

## Relevant Links

- [RDS Performance Monitoring](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Monitoring.html)
- [EBS Volume Performance](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
- [ElastiCache Redis Metrics](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/CacheMetrics.Redis.html)
- [CloudFront Cache Statistics](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cache-statistics.html)
- [CloudWatch Anomaly Detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html)
- [CloudWatch RUM](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM.html)
