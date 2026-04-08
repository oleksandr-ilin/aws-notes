# AWS Pricing Calculator

## Service Overview
The AWS Pricing Calculator helps estimate costs for architectures and plan budgets before deployment.

## Tasks this service can solve
- Estimate monthly costs for proposed architectures
- Compare pricing options (on-demand, reserved, savings plans)

## Alternatives
| Name | Type | Short description | Pro vs Calculator | Cons vs Calculator | Price comparison |
|---|---|---:|---|---|---|
| Third-party cost estimators | Commercial | Vendor calculators with templates | Specialized templates and scenarios | Extra cost | Varies; calculators are advisory only |
| Manual spreadsheets on CUR data | OSS/Custom | Custom cost models | Fully customizable | Time-consuming and error-prone | Ops cost vs managed calculator

## Limitations
- Estimates may differ from actual bills due to discounts, taxes, or usage patterns.

## Price info
- Tool is free to use; actual billed costs depend on chosen services and regions.

## Network & Multi-region considerations
- Include regional pricing differences when modeling multi-region deployments.

## When Not to Use This Service
- Not a billing source of truth; use CUR or Cost Explorer for historical/actual billing data.

## DR strategy
- Preserve calculator configurations and templates in documentation to rebuild estimates after incidents.

## Popular use cases
- Pre-procurement cost modeling and comparison between deployment options.
