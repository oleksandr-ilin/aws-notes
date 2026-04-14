# Q03: RI Sharing in AWS Organizations

## Question

A company has 5 business units, each operating in its own AWS account under AWS Organizations with consolidated billing enabled. The marketing business unit purchased 20 Standard Reserved Instances (m5.xlarge, us-east-1) for its steady-state workloads. However, the data science business unit is now running 10 m5.xlarge instances in us-east-1 and wants to benefit from the marketing unit's unused RI capacity. The marketing unit objects, stating that their unused RIs should not subsidize other business units' costs. The CFO wants each business unit's AWS bill to accurately reflect its own committed discounts.

How should the solutions architect address this situation?

## Options

- **A.** Move each business unit into a separate AWS Organization so that RI benefits are not shared between accounts.
- **B.** Turn off Reserved Instance sharing on the management account. Each business unit will only benefit from its own purchased RIs. Instruct the data science unit to purchase its own RIs.
- **C.** Use cost allocation tags to track RI benefits per business unit. Apply the tags to all instances and use AWS Cost Explorer to create chargeback reports showing the correct RI attribution.
- **D.** Create Service Control Policies (SCPs) that restrict the data science account from using m5.xlarge instances, preventing it from consuming the marketing unit's RI benefits.

## Answers

### B. Disable RI sharing on the management account — ✅ Correct

In AWS Organizations with consolidated billing, Reserved Instance discounts are shared across all member accounts by default. However, the management account can **turn off RI sharing** for specific accounts or the entire organization. When RI sharing is disabled, each account's RIs only apply to its own usage. The data science unit can then purchase its own RIs. This directly addresses the marketing unit's concern while maintaining accurate per-unit billing.

### A. Separate Organizations — ❌ Incorrect

Moving business units to separate Organizations would prevent RI sharing but would also lose all benefits of consolidated billing, including volume discounts, Savings Plans sharing, and centralized governance via SCPs and Control Tower. This is a drastic, disproportionate solution to a simple billing configuration issue.

### C. Cost allocation tags — ❌ Incorrect

Cost allocation tags can help track costs, but they don't prevent RI benefits from being applied across accounts. The marketing unit's RIs would still be consumed by the data science unit's matching instances. Tags only provide visibility — they don't enforce billing boundaries.

### D. SCPs to restrict instance types — ❌ Incorrect

SCPs control API permissions, not billing. Preventing the data science unit from running m5.xlarge instances would block their legitimate workloads just to avoid RI sharing. SCPs are a security and governance tool, not a cost management tool. The data science unit needs to run those instance types for their work.

## Recommendations

- **RI discount sharing** is enabled by default in AWS Organizations — be intentional about whether this benefits your cost structure.
- Disable RI sharing when business units have separate budgets and need independent cost accountability.
- Keep RI sharing enabled when the organization benefits from pooled discounts (e.g., if one team's off-peak usage consumes another team's unused RI hours).
- The same sharing behavior applies to **Savings Plans** — you can also disable Savings Plans sharing at the management account level.
- Use **AWS Organizations consolidated billing** to realize volume pricing benefits even when RI sharing is disabled.

## Relevant Links

- [Turning Off Reserved Instance Sharing](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/ri-turn-off.html)
- [Reserved Instance Sharing in Organizations](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/ri-behavior.html)
- [AWS Organizations Consolidated Billing](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html)
- [Savings Plans Sharing](https://docs.aws.amazon.com/savingsplans/latest/userguide/sp-applying.html)
