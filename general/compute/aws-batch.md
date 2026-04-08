Overview
AWS Batch schedules and runs batch computing workloads on managed EC2 or Fargate resources.

Tasks
- Define job definitions, queues and compute environments; submit and monitor jobs.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom scheduler on ECS/EKS | Flexibility | More ops

Limitations
- Latency for provisioning and queueing; job dependency management complexity.

Price info
- Pay for underlying EC2/Fargate compute used by jobs.

Network & multi-region
- Compute environments are region-scoped; plan data locality and transfer.

Popular use cases
- High-performance computing, large-scale batch ETL, and scientific workloads.

When Not to Use
- Low-latency or interactive workloads.

DR strategy
- Keep job definitions and compute environment configs in source control; use cross-region job visibility if required.
