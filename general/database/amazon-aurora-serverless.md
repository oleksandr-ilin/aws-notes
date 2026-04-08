# Amazon Aurora Serverless

## Service Overview
Aurora Serverless provides on-demand autoscaling for Aurora using Aurora Capacity Units (ACUs) to scale DB capacity up/down automatically.

## Tasks this service can solve
- Spiky or infrequent relational workloads
- Cost-optimized dev/test or low-traffic prod workloads

## Alternatives
| Name | Type | Short description | Pro vs Serverless | Cons vs Serverless | Price comparison |
|---|---|---:|---|---|---|
| Aurora provisioned | AWS | Standard Aurora with fixed instances | Predictable performance | Higher baseline costs | Provisioned can be cheaper for steady loads |
| RDS on small instances | AWS | Managed relational instances | Simpler billing model | Less elastic scaling | Depends on utilization patterns |

## Limitations
- Cold starts and connection management challenges; not ideal for consistently high throughput.

## Price info
- Billed per ACU-second; additional charges for storage and I/O.

## Network & Multi-region considerations
- Similar to Aurora: use Global Database for cross-region reads; replicate critical data and be mindful of cold-starts across regions.

## When Not to Use This Service
- Avoid for steady-state, high-throughput databases where provisioned Aurora or RDS provides better cost predictability and performance.

## DR strategy
- Use automated backups and cross-region replicas where supported; test scaling behavior and failover during DR drills.

## Popular use cases
- Development databases, low-traffic SaaS customers, and unpredictable workloads.
