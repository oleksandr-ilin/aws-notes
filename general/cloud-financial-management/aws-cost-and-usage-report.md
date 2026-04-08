# AWS Cost and Usage Report (CUR)

## Service Overview
The Cost and Usage Report exports detailed billing and usage data to S3 for analysis and cost allocation.

## Tasks this service can solve
- Detailed chargeback and cost allocation reporting
- Feeding billing data into BI or analytics systems
- Long-term storage of usage records for audits

## Alternatives
| Name | Type | Short description | Pro vs CUR | Cons vs CUR | Price comparison |
|---|---|---:|---|---|---|
| Third-party billing exports | Commercial | Managed ingestion to FinOps tools | Turnkey analytics & UI | Additional cost | Commercial fees vs storage/processing costs for CUR |
| Cost Explorer APIs | AWS | Programmatic access to summarized cost data | Easier summaries | Less granularity | CUR is raw and more detailed; may cost more to process |

## Limitations
- CUR files can be large and require storage and processing; schema complexity requires ETL.

## Price info
- Generating CUR is free; primary costs are S3 storage and processing of the exported files.

## Network & Multi-region considerations
- CUR is delivered to an S3 bucket in a chosen region; use cross-region replication or centralized analytics buckets for multi-region reporting.

## When Not to Use This Service
- If you only need quick visualizations and aggregated trends, `Cost Explorer` may be simpler than CUR.

## DR strategy
- Ensure CUR S3 buckets have proper lifecycle, replication (CRR) and backups; maintain processing ETL code in source control to rebuild reports if needed.

## Popular use cases
- Chargeback reports, detailed resource-level cost analysis, and feeding data into custom FinOps dashboards.
