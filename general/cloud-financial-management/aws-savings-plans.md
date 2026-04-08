# AWS Savings Plans

## Service Overview
Savings Plans provide flexible, commitment-based discounts on compute usage (EC2, Fargate, Lambda) in exchange for a usage commitment.

## Tasks this service can solve
- Reduce compute costs with commitment-based discounts
- Simplify purchasing compared to Reserved Instances

## Alternatives
| Name | Type | Short description | Pro vs Savings Plans | Cons vs Savings Plans | Price comparison |
|---|---|---:|---|---|---|
| Reserved Instances | AWS | Instance-family specific discounts | Savings Plans are more flexible across compute types | RIs can be cheaper for specific cases | Savings depend on commitment term and usage alignment |
| No commitment | N/A | Pay-as-you-go on-demand pricing | Flexibility | Higher cost | On-demand typically higher than committed discounts |

## Limitations
- Requires predictable baseline compute usage; committing incorrectly can lead to suboptimal savings.

## Price info
- Discounts are applied based on commitment (1- or 3-year terms) and payment options; evaluate Savings Plans vs RIs for best fit.

## Network & Multi-region considerations
- Savings Plans apply across the billing account; ensure consolidated billing to maximize application across accounts.

## When Not to Use This Service
- Not suitable for highly unpredictable or extremely short-term workloads where commitments add risk.

## DR strategy
- Track and report Savings Plans utilization; maintain financial runbooks for renegotiation or conversion; no direct DR requirements but preserve billing data and offer reallocation strategies across accounts.

## Popular use cases
- Steady-state web services and compute-heavy workloads with predictable usage.
