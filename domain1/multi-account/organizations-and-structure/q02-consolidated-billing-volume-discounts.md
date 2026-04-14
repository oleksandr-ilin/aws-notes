# Q02: Consolidated Billing and Volume Discounts

## Question

A company operates 50 AWS accounts across 3 AWS Organizations (one per geographic region: Americas, EMEA, APAC). Each Organization has its own management account. The company's total monthly AWS spend is $2 million, but volume discount tiers are calculated per Organization ($800K Americas, $700K EMEA, $500K APAC). The CFO wants to maximize volume pricing discounts for services like S3, EC2 data transfer, and Lambda. The company also wants a single payer for all AWS invoices.

Which approach should the solutions architect recommend?

## Options

- **A.** Consolidate all 50 accounts under a single AWS Organization with one management account. Use Organizational Units to maintain the geographic separation for governance. This aggregates all usage for volume discount calculations.
- **B.** Keep the three separate Organizations. Use AWS Cost Explorer in each management account to negotiate custom volume discounts with AWS based on the company's total combined spend.
- **C.** Create a new AWS Organization as the parent and use AWS Organizations' multi-Organization management feature to link the three existing Organizations for consolidated billing.
- **D.** Keep the three Organizations. Purchase Enterprise Discount Program (EDP) commitments separately for each Organization to get volume discounts.

## Answers

### A. Single Organization with OUs — ✅ Correct

Consolidating all accounts under a single AWS Organization provides:
- **Aggregated usage** across all 50 accounts for volume pricing tiers (S3, data transfer, Lambda, etc.). $2M combined spend qualifies for higher discount tiers than three separate $500-800K Organizations.
- **Single payer** for all invoices via the management account.
- **OUs** maintain geographic segregation for governance, Region-specific SCPs, and compliance (e.g., GDPR for EMEA accounts restricted to eu-* Regions).
- **Consolidated billing** automatically shares RI and Savings Plans benefits across all accounts.

### B. Negotiate per-Organization — ❌ Incorrect

AWS volume discounts are applied automatically based on aggregated usage within an Organization — you cannot manually request cross-Organization aggregation without an EDP. Keeping separate Organizations means each gets discounts based only on its own usage, leaving significant savings on the table.

### C. Multi-Organization linking — ❌ Incorrect

AWS Organizations does not support a "multi-Organization management" or parent-Organization feature. There is no native way to link separate Organizations for consolidated billing. Each Organization is independent. The only way to aggregate billing is to consolidate accounts under a single Organization.

### D. Separate EDP commitments — ❌ Incorrect

The Enterprise Discount Program (EDP) provides custom discounts based on committed spend. However, three separate EDP commitments would each be smaller ($500-800K), yielding lower discount percentages than a single $2M EDP commitment. Consolidating into one Organization before negotiating an EDP would maximize the discount.

## Recommendations

- **Single Organization** is almost always preferable for a single company — it maximizes volume discounts and simplifies management.
- Use **OUs** and **SCPs** to maintain governance boundaries that previously required separate Organizations.
- **Region-restriction SCPs** can enforce geographic compliance (e.g., GDPR) without needing separate Organizations.
- For very large enterprises, an **Enterprise Discount Program (EDP)** provides additional custom discounts on top of volume pricing.
- When consolidating, plan the **management account migration** carefully — migrating accounts between Organizations requires removing from one and inviting to another.

## Relevant Links

- [AWS Organizations Consolidated Billing](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html)
- [Volume Discounts](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/useconsolidatedbilling-discounts.html)
- [Moving Accounts Between Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_move.html)
