# Q01: Reserved Instances vs. Savings Plans

## Question

A company runs a fleet of 200 Amazon EC2 instances across us-east-1 and eu-west-1. The workload is stable: 150 instances run 24/7 as c5.xlarge, and the remaining 50 vary between c5.xlarge and m5.xlarge depending on workload demands. The company previously used Standard Reserved Instances for all instances but now wants more flexibility. The team is evaluating Compute Savings Plans, EC2 Instance Savings Plans, and Reserved Instances for the next 1-year commitment.

Which purchasing strategy should the solutions architect recommend to maximize cost savings while maintaining flexibility?

## Options

- **A.** Purchase Standard Reserved Instances for all 200 instances as c5.xlarge in both Regions. Use the RI Marketplace to sell unused RIs if the instance mix changes.
- **B.** Purchase EC2 Instance Savings Plans for the 150 stable c5.xlarge instances. Purchase Compute Savings Plans for the remaining 50 instances that vary between c5.xlarge and m5.xlarge.
- **C.** Purchase Compute Savings Plans for the entire fleet of 200 instances. This provides maximum flexibility across instance families, sizes, Regions, and compute services.
- **D.** Purchase Convertible Reserved Instances for all 200 instances. Convert them between c5 and m5 families as workload demands change.

## Answers

### B. EC2 Instance Savings Plans + Compute Savings Plans — ✅ Correct

This strategy optimizes the tradeoff between savings and flexibility:
- **EC2 Instance Savings Plans** (for the stable 150 instances) offer up to 72% savings — the highest savings rate for Savings Plans — and apply to a specific instance family in a specific Region (c5 in us-east-1 / eu-west-1). Since these instances are stable, the reduced flexibility is acceptable.
- **Compute Savings Plans** (for the variable 50 instances) offer up to 66% savings and automatically apply across any instance family, size, Region, OS, or tenancy. This covers the c5↔m5 variation without any changes needed.

### C. Compute Savings Plans for all — ❌ Incorrect

Compute Savings Plans provide excellent flexibility but offer a lower discount rate (up to 66%) compared to EC2 Instance Savings Plans (up to 72%). For the 150 stable instances that will remain c5.xlarge, using Compute Savings Plans leaves money on the table. The best strategy is to use the more specific (higher discount) plan where possible and the flexible plan where needed.

### A. Standard Reserved Instances for all — ❌ Incorrect

Standard RIs provide the highest discount but cannot be exchanged for different instance families. If the 50 variable instances change from c5 to m5, the RIs for those instances would go unused. Selling on the RI Marketplace is possible but introduces delays, uncertainty, and the overhead of managing listings. This was the exact problem the company wants to solve.

### D. Convertible Reserved Instances for all — ❌ Incorrect

Convertible RIs allow exchanging for different instance types but offer a lower discount than Standard RIs (~54% vs. ~72% for 1-year). They also require manual conversion actions and can only be exchanged for equal or greater value. Savings Plans provide a simpler, more flexible model than Convertible RIs.

## Recommendations

- **Hierarchy of savings** (highest to lowest discount): Standard RI > EC2 Instance Savings Plan > Compute Savings Plan > Convertible RI > On-Demand.
- Use **EC2 Instance Savings Plans** when you're committed to a specific instance family in a specific Region.
- Use **Compute Savings Plans** when you need flexibility across families, sizes, Regions, or even between EC2, Fargate, and Lambda.
- **Savings Plans apply automatically** — no need to match instance types; they apply to your highest-savings-eligible usage first.
- Use **AWS Cost Explorer's Savings Plans recommendations** to analyze usage and get right-sized commitment recommendations.

## Relevant Links

- [Savings Plans](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
- [Savings Plans types](https://docs.aws.amazon.com/savingsplans/latest/userguide/savings-plans-types.html)
- [EC2 Reserved Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html)
- [Understanding Savings Plans Pricing](https://aws.amazon.com/savingsplans/pricing/)
