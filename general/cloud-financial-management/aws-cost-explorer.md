# AWS Cost Explorer

## Service Overview
Cost Explorer provides interactive visualizations, forecasting, and rightsizing recommendations for AWS spend.

## Tasks this service can solve
- View cost trends and forecasts
- Identify idle or oversized resources for rightsizing
- Analyze spend by tag, account, or service

## Alternatives
| Name | Type | Short description | Pro vs Cost Explorer | Cons vs Cost Explorer | Price comparison |
|---|---|---:|---|---|---|
| Third-party FinOps tools | Commercial | Advanced analytics and multi-cloud support | Deeper features and multi-cloud | Extra licensing cost | Higher but provides cross-cloud views |
| Custom analytics on CUR | OSS/Custom | Build dashboards from CUR | Fully customizable | More engineering effort | Ops cost vs managed tooling |

## Limitations
- Data granularity and lookback windows are limited compared to raw CUR data.

## Price info
- Cost Explorer is free to use; some APIs or advanced features may have limits—processing costs are indirect.

## Network & Multi-region considerations
- Works across consolidated billing accounts; ensure tagging strategies and account structure support desired views.

## When Not to Use This Service
- If you need raw, per-API-call billing data for auditing or custom processing, use CUR instead.

## DR strategy
- Preserve custom reports and saved views (export configurations) and keep any automation scripts in source control; re-create views if the account is restored.

## Popular use cases
- Monthly cost reviews, identifying rightsizing opportunities, and forecasting next-quarter spend.
