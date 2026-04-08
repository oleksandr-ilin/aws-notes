# AWS Reserved Instances

## Service Overview
Reserved Instances (RIs) provide capacity and pricing discounts for EC2 instances in exchange for a term commitment.

## Tasks this service can solve
- Lower costs for predictable, steady-state instances
- Reserve capacity in specific AZs for capacity planning

## Alternatives
| Name | Type | Short description | Pro vs RIs | Cons vs RIs | Price comparison |
|---|---|---:|---|---|---|
| Savings Plans | AWS | More flexible commitment-based discounts | More flexibility across compute types | RIs can offer zonal capacity reservations | Variable; RIs sometimes cheaper for specific instances |
| Spot Instances | AWS | Use spare capacity at large discounts | Very low cost | Interruptible workloads only | Much cheaper but not suitable for stateful always-on services |

## Limitations
- Commitments are term-based and may not match changing workload needs; zonal RIs tie capacity to AZs.

## Price info
- Pricing depends on instance family, tenancy, term length, and payment option (no upfront, partial, all upfront).

## Network & Multi-region considerations
- Zonal RIs provide capacity reservations within AZs; plan for multi-AZ redundancy when using zonal reservations.

## When Not to Use This Service
- Avoid for highly variable workloads where commitment risk outweighs savings; consider on-demand or spot for variable patterns.

## DR strategy
- Use multi-AZ deployments and avoid relying on single zonal RI capacity for critical systems. Keep infrastructure-as-code to re-provision in other zones/regions if needed.

## Popular use cases
- Core database or application servers with predictable utilization.
