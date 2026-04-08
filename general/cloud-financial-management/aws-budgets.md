# AWS Budgets

## Service Overview
AWS Budgets lets you set custom cost and usage budgets and receive alerts when thresholds are exceeded.

## Tasks this service can solve
- Set monthly cost/usage budgets and alerts
- Track budget forecasts and savings plan utilization
- Enforce cost guardrails for teams and projects

## Alternatives
| Name | Type | Short description | Pro vs Budgets | Cons vs Budgets | Price comparison |
|---|---|---:|---|---|---|
| Custom billing pipelines | OSS/Custom | Export billing data and run custom alerts | Fully flexible rules | More ops and development | Higher ops cost; Budgets is managed and low-effort |
| Third-party FinOps tools | Commercial | Advanced cost allocation and forecasting | Richer analytics & integrations | Additional licensing cost | Often higher total cost but more features |

## Limitations
- Setup can be manual for many accounts; some advanced alerts require integration with SNS/CloudWatch.

## Price info
- Budgets itself has a basic free tier; advanced notifications or high-volume usage may incur charges (check current AWS terms).

## Network & Multi-region considerations
- Budgets is account/region aware for certain metrics; consolidate billing (Organizations) to monitor across accounts and regions.

## When Not to Use This Service
- If you need deep, cross-cloud FinOps analytics and forecasting, consider commercial FinOps platforms instead.

## DR strategy
- Budgets configurations should be exported or stored in source control (JSON templates) and be re-applied via automation (CloudFormation/CLI) if accounts are recreated.

## Popular use cases
- Team-level cost alerts, budget guardrails for projects, and tracking Savings Plans utilization.
