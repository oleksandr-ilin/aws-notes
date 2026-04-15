# Q03: Performance Monitoring, Right-Sizing Validation, and EFA for HPC

## Question

A company has completed a cloud migration. The CTO wants to validate that all resources are right-sized — not over-provisioned (wasting money) or under-provisioned (degrading performance). Three workloads need assessment:

**Workload 1 — Web tier (20 EC2 instances, c5.2xlarge)**
- ALB → 20 c5.2xlarge instances in an ASG
- CloudWatch shows: average CPU 28%, peak CPU 45%, memory unknown (no custom metrics), network throughput 150 Mbps average
- The team suspects instances are over-provisioned but lacks data to prove it
- Requirement: Get a data-driven recommendation for the correct instance type and size

**Workload 2 — Microservices API (ECS Fargate + Lambda)**
- 12 ECS Fargate services + 30 Lambda functions behind API Gateway
- Users report intermittent slowness on certain API endpoints, but CloudWatch latency metrics show averages are fine (P50 = 80 ms)
- The team suspects a specific downstream service is the bottleneck but can't identify which one
- Requirement: Trace individual requests across all services to identify the latency bottleneck

**Workload 3 — Molecular dynamics simulation (HPC cluster)**
- 64 p4d.24xlarge GPU instances running a distributed simulation across all instances
- Each instance has 8 NVIDIA A100 GPUs and 400 Gbps network bandwidth
- Current: Instances communicate over standard VPC networking (TCP/IP)
- Problem: Inter-node communication latency is 25 microseconds — the simulation requires < 5 microsecond latency for MPI collective operations. GPU compute finishes quickly but spends 60% of time waiting for network communication
- Requirement: Reduce inter-node network latency to < 5 microseconds

Which combination of monitoring and optimization strategies addresses all three workloads?

## Options

- **A.** Workload 1: Enable **AWS Compute Optimizer** — it analyzes 14 days of CloudWatch metrics (CPU, memory, network, disk) and recommends optimal instance types/sizes. Also install the **CloudWatch agent** on all instances to collect memory and disk utilization (not available via default metrics). Compute Optimizer ingests these custom metrics for more accurate recommendations. Workload 2: Enable **AWS X-Ray** tracing across all services. X-Ray traces individual requests end-to-end: API Gateway → Lambda → ECS → downstream services. The **X-Ray service map** visualizes call dependencies and highlights the service with the highest latency contribution. Drill into **trace segments** to identify the exact function/query causing slowness. Workload 3: Attach **Elastic Fabric Adapter (EFA)** to each p4d.24xlarge instance. EFA provides OS-bypass networking with access to the Scalable Reliable Datagram (SRD) protocol — delivering < 5 microsecond MPI latency. Use EFA with a **Cluster Placement Group** to co-locate instances on the same network spine for minimum hop count.
- **B.** Workload 1: Analyze CloudWatch CPU metrics manually and choose a smaller instance type based on peak CPU. Workload 2: Enable CloudWatch Container Insights for ECS and Lambda Insights — they show per-service metrics. Workload 3: Use enhanced networking (ENA) with jumbo frames (9000 MTU).
- **C.** Workload 1: Use AWS Trusted Advisor for right-sizing checks. Workload 2: Enable CloudWatch Contributor Insights to find top API callers. Workload 3: Use AWS ParallelCluster with FSx for Lustre for faster storage I/O.
- **D.** Workload 1: Compute Optimizer with CloudWatch agent. Workload 2: CloudWatch Synthetics canaries to simulate user traffic and measure latency. Workload 3: Use Direct Connect dedicated connection between instances.

## Answers

### A. Compute Optimizer + CloudWatch Agent + X-Ray tracing + EFA — ✅ Correct

**Workload 1 — AWS Compute Optimizer + CloudWatch Agent:**

- **AWS Compute Optimizer**:
  - Compute Optimizer uses ML models to analyze CloudWatch metrics and recommend optimal AWS resource configurations. It supports: EC2 instances, Auto Scaling groups, EBS volumes, Lambda functions, ECS services on Fargate, and RDS instances.
  - **Analysis window**: Evaluates up to 93 days of CloudWatch metric data (14 days minimum for initial recommendations, more data = better accuracy). Enhanced infrastructure metrics (paid feature) extend the analysis window from 14 to 93 days.
  - **Recommendations include**:
    - Current instance type vs. recommended instance type (e.g., c5.2xlarge → c5.xlarge or m6g.xlarge)
    - Projected performance risk (probability of CPU/memory contention with the smaller instance)
    - Estimated monthly savings
    - Instance family changes (e.g., C-family → M-family if the workload is balanced, or C → R if memory-bound)
    - Graviton migration opportunities (e.g., c5.2xlarge → c6g.xlarge for 40% better price-performance)

- **CloudWatch Agent for memory metrics**:
  - Default EC2 CloudWatch metrics include: CPU utilization, network in/out, disk read/write ops, status checks. **Memory utilization is NOT included** in default metrics — it requires the CloudWatch agent.
  - Install the unified CloudWatch agent on each instance. Configure it to collect: `mem_used_percent`, `disk_used_percent`, `swap_used`, `processes_total`.
  - **Why memory matters for right-sizing**: An instance at 28% CPU might be at 90% memory — downsizing CPU (smaller instance) could make memory the bottleneck. Without memory data, Compute Optimizer's recommendations are based on CPU/network only — potentially recommending an instance too small for the memory workload.
  - Compute Optimizer automatically ingests CloudWatch agent memory metrics (if the metric name matches the expected format) for improved recommendations.

- **Right-sizing outcome for this workload**: With CPU at 28%/45% and unknown memory:
  - If memory < 50%: Compute Optimizer likely recommends c5.xlarge (half the size) or c6g.xlarge (Graviton) — saving ~50%.
  - If memory > 80%: Compute Optimizer recommends r-family (memory-optimized) or maintains compute size — different recommendation entirely.
  - This is why memory data is critical for accurate right-sizing.

**Workload 2 — AWS X-Ray Distributed Tracing:**

- **The problem with CloudWatch metrics alone**:
  - CloudWatch shows aggregate P50 latency = 80 ms — looks fine. But P50 masks tail latency: P99 might be 2,000 ms (1 in 100 requests is extremely slow).
  - CloudWatch metrics are per-service aggregates — they don't show which service IN THE REQUEST CHAIN is causing the delay. Is it Lambda cold start? ECS slow query? External API timeout? CloudWatch can't answer this without correlated tracing.

- **X-Ray distributed tracing**:
  - X-Ray instruments each service to generate **trace segments**. A segment captures: start time, end time, service name, HTTP status, and metadata for each service hop in the request chain.
  - **Trace example**: API Gateway (5 ms) → Lambda (15 ms) → ECS-OrderService (200 ms) → ECS-InventoryService (1,800 ms) → DynamoDB (3 ms). The trace immediately reveals ECS-InventoryService is the bottleneck (1,800 ms of the 2,038 ms total).
  
- **X-Ray service map**:
  - Visual dependency graph showing all services and their connections. Each node displays: average latency, error rate, and request count. Nodes with high latency or errors are highlighted (red/yellow).
  - The service map shows: "InventoryService has P99 latency of 1,800 ms and calls an external supplier API that times out intermittently." The team now knows exactly where to focus.

- **X-Ray subsegments**:
  - Within a segment, subsegments break down internal operations: database queries, HTTP calls, SDK calls. For the InventoryService: subsegment 1 = DynamoDB query (3 ms), subsegment 2 = external API call (1,795 ms). The external API is the root cause.

- **Instrumentation**:
  - Lambda: Enable X-Ray active tracing in the function configuration (one checkbox). AWS SDK calls are automatically traced.
  - ECS: Add the X-Ray daemon as a sidecar container. Instrument the application code with the X-Ray SDK (adds 5-10 lines of code).
  - API Gateway: Enable X-Ray tracing in the stage settings.

**Workload 3 — Elastic Fabric Adapter (EFA):**

- **The network bottleneck**:
  - Standard VPC networking uses TCP/IP over the Elastic Network Adapter (ENA). TCP/IP involves: kernel network stack processing, context switches, memory copies, and protocol overhead. Latency: 25-50 microseconds per message.
  - MPI (Message Passing Interface) collective operations (AllReduce, AllGather, Broadcast) send millions of small messages between all nodes. At 25 μs per message × millions of messages = 60% of runtime spent waiting for network.

- **EFA (Elastic Fabric Adapter)**:
  - EFA is a custom-built network interface that provides **OS-bypass** capability. Applications communicate directly with the network hardware, bypassing the OS kernel network stack entirely.
  - **Libfabric API**: Applications use the libfabric API (OFI — Open Fabrics Interfaces) to send/receive messages. This is the same API used by on-premises InfiniBand/RoCE fabric — HPC applications designed for fabric networking work with EFA without modification.
  - **SRD protocol**: EFA uses AWS's Scalable Reliable Datagram (SRD) protocol instead of TCP/IP. SRD provides:
    - Reliable delivery (like TCP) with lower overhead
    - Multi-path routing across the network fabric
    - Congestion control without head-of-line blocking
  - **Latency: < 5 microseconds** for small messages (comparable to on-premises InfiniBand). This meets the simulation's requirement.

- **Cluster Placement Group**:
  - A Cluster Placement Group places instances on the same underlying network switch fabric — minimizing network hops (typically single-hop connectivity).
  - EFA + Cluster Placement Group = minimum possible network latency between instances.
  - **Limitation**: Cluster Placement Groups don't span AZs. All 64 instances must be in the same AZ — sacrificing availability for performance. For HPC workloads, this is acceptable (simulations are batch jobs, not HA services).

- **Supported instances**: EFA is available on: p4d.24xlarge, p5.48xlarge, c5n.18xlarge, hpc6a.48xlarge, and other HPC/GPU instance types. Not all instance types support EFA — check the EFA-supported instances list.

- **EFA vs. ENA**:

  | Feature | ENA (Elastic Network Adapter) | EFA (Elastic Fabric Adapter) |
  |---|---|---|
  | Protocol | TCP/IP, UDP | SRD (OS-bypass) + TCP/IP |
  | Latency | 25-50 μs | < 5 μs |
  | Use case | General networking | HPC, ML training (MPI, NCCL) |
  | OS bypass | ❌ | ✅ |
  | Instance types | All current-gen | Select HPC/GPU types |
  | VPC features | Full (SG, NACL, VPC Peering) | SG only (no NACL, no VPC Peering for SRD traffic) |

### B. Manual CPU analysis + Container Insights + ENA jumbo frames — ❌ Incorrect

- **Manual CPU analysis**: Manually reviewing CPU metrics and choosing a smaller instance misses: memory utilization (unknown), network bandwidth requirements, disk IOPS patterns, and Graviton migration opportunities. Compute Optimizer automates this analysis with ML models trained on millions of workloads — it identifies patterns humans miss (e.g., network-bound workloads misattributed to CPU).

- **Container Insights / Lambda Insights**: These provide per-service metrics (CPU, memory, network for each ECS service or Lambda function). They show "InventoryService CPU is high" but NOT "request X spent 1,800 ms in InventoryService because it waited for an external API." Container Insights provides observability per service; X-Ray provides observability per request across services. For "which service is the bottleneck for slow requests," X-Ray is the correct tool.

- **ENA with jumbo frames (9000 MTU)**: ENA is the standard network adapter — it supports up to 25 Gbps. Jumbo frames reduce per-packet overhead by sending 9 KB frames instead of 1.5 KB. This improves throughput for large transfers but does NOT reduce latency. The bottleneck is latency (25 μs per message), not throughput (p4d has 400 Gbps). Jumbo frames might reduce latency by 1-2 μs — still far from the < 5 μs target. EFA's OS-bypass is required for sub-5 μs latency.

### C. Trusted Advisor + Contributor Insights + ParallelCluster — ❌ Incorrect

- **Trusted Advisor for right-sizing**: Trusted Advisor's "Low Utilization Amazon EC2 Instances" check flags instances with < 10% CPU for 4+ days. This is a blunt check — it doesn't recommend specific alternative instance types, doesn't consider memory/network, and doesn't evaluate Graviton opportunities. Compute Optimizer provides specific, data-driven recommendations with projected performance impact.

- **Contributor Insights**: Contributor Insights identifies the "top N" contributors to a CloudWatch metric — e.g., "which API endpoints have the most 5xx errors" or "which users generate the most requests." This helps find WHO is causing load, not WHERE in the service chain the latency is. The question asks "which downstream service is the bottleneck" — X-Ray traces answer this, not Contributor Insights.

- **ParallelCluster + FSx for Lustre**: AWS ParallelCluster is a managed HPC cluster solution — it automates cluster setup (Slurm scheduler, compute nodes, shared storage). FSx for Lustre provides high-throughput shared storage. These are useful for HPC clusters, but they address cluster management and storage I/O — not network latency. The simulation's bottleneck is inter-node communication latency (25 μs), not storage or job scheduling. EFA is the network solution; ParallelCluster can use EFA but isn't the solution itself.

### D. Compute Optimizer + CloudWatch Synthetics + Direct Connect — ❌ Incorrect

- **Compute Optimizer**: Correct for Workload 1 ✅.

- **CloudWatch Synthetics for Workload 2**: Synthetics runs canary scripts that simulate user traffic and measure endpoint latency from outside. This detects "is the API slow?" — it confirms the problem exists. But it doesn't trace requests through internal services to identify WHICH service is the bottleneck. Synthetics is for external availability and performance monitoring; X-Ray is for internal request tracing. They're complementary, not substitutes.

- **Direct Connect for instance-to-instance communication**: Direct Connect provides dedicated network connections from on-premises to AWS — it doesn't connect EC2 instances to each other. EC2 instances communicate over the VPC network fabric. "Direct Connect between instances" is not a valid configuration. EFA provides the low-latency inter-instance networking needed for HPC.

## Recommendations

- **Enable Compute Optimizer organization-wide**: Opt in all accounts via AWS Organizations. It's free for standard recommendations (enhanced metrics cost extra).
- **Always install CloudWatch agent**: Standard EC2 metrics lack memory and disk usage. The CloudWatch agent adds these critical metrics — essential for right-sizing, capacity planning, and alerting.
- **X-Ray sampling**: In production, use X-Ray sampling (e.g., 5% of requests) to reduce overhead and cost. For debugging specific issues, temporarily increase sampling to 100%.
- **X-Ray + CloudWatch combination**: Use CloudWatch metrics/alarms for detection ("P99 latency exceeded threshold") and X-Ray for diagnosis ("which service caused the spike"). Set up CloudWatch alarms that trigger X-Ray sampling increases.
- **EFA for ML training**: EFA is not just for HPC simulations — ML training with multi-GPU distributed training (using NCCL AllReduce) benefits equally. PyTorch DDP, Horovod, and SageMaker distributed training all leverage EFA.
- **Monitor right-sizing continuously**: Run Compute Optimizer monthly. Workload patterns change — an instance that was right-sized 6 months ago may be oversized now due to code optimizations or undersized due to traffic growth.

## Relevant Links

- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
- [CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
- [X-Ray Service Map](https://docs.aws.amazon.com/xray/latest/devguide/xray-console-servicemap.html)
- [Elastic Fabric Adapter](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [EFA Supported Instance Types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html#efa-instance-types)
- [Cluster Placement Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html#placement-groups-cluster)
