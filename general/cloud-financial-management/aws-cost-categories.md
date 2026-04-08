# AWS Cost Categories

## Service Overview
Cost Categories let you define logical groupings of costs (by team, product, or environment) for reporting and allocation.

## Tasks this service can solve
- Create business-oriented views of spend (by product/team)
- Enable consistent tagging and reporting across accounts

## Alternatives
| Name | Type | Short description | Pro vs Cost Categories | Cons vs Cost Categories | Price comparison |
|---|---|---:|---|---|---|
| Tag-based reporting on CUR | OSS/Custom | Build groupings using tags in CUR processing | Full custom control | More processing and ops | Ops cost vs managed category feature |
| Third-party FinOps categorization | Commercial | Automated grouping and allocation | Richer rules and UI | Licensing cost | Higher but with more features

## Limitations
- Categories rely on accurate tags and account structures; mis-tagging reduces value.

## Price info
- Feature is part of Cost Management; costs are primarily in CUR processing and storage.

## Network & Multi-region considerations
- Categories operate across consolidated billing; ensure tag strategies are enforced across regions.

## When Not to Use This Service
- If your organization has inconsistent or unmanaged tagging, invest in tagging governance first before relying on categories.

## DR strategy
- Preserve categorization rules in source control and document mapping to tags/accounts so they can be re-applied if needed.

## Popular use cases
- Product-level cost allocation, showback/chargeback reports, and budget assignments for teams.
