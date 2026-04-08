# Amazon AppFlow

## Service Overview
Amazon AppFlow is a low-code, fully managed service for securely transferring data between SaaS applications and AWS services.

## Tasks this service can solve
- SaaS-to-S3 or SaaS-to-Salesforce data syncs
- Simple transformations and field mapping
- Scheduled or event-driven data flows

## Alternatives
| Name | Type | Short description | Pro vs AppFlow | Cons vs AppFlow | Price comparison |
|---|---|---:|---|---|---|
| Custom ETL on Lambda/Glue | AWS | Build bespoke connectors and jobs | Full control | More ops and development | Higher ops cost; AppFlow cheaper for standard connectors |
| Mulesoft / Boomi | Commercial | Enterprise integration platforms | More enterprise adapters and governance | Cost and complexity | Commercial license costs higher |
| Airbyte / Meltano | OSS | Open-source connectors and ELT | Avoid vendor lock-in | Self-hosting required | Ops cost vs AppFlow managed pricing |

## Limitations
- Connector set is curated; very custom SaaS connectors may require custom development.
- Connector throughput and per-flow limits.

## Price info
- Charged per-flow run and data processed; connector-specific pricing may apply.

## Network & Multi-region considerations
- AppFlow connections are regional; sensitive data flows should consider region residency. Use per-region flows or centralize data in a designated analytics region.
- Integrates with S3, Redshift, Salesforce, and other SaaS endpoints.

## Popular use cases
- Periodic syncing of CRM leads to S3/Redshift
- SaaS billing ingestion into data lake
- Cross-system event forwarding for analytics

## When Not to Use This Service
- Not a fit for highly custom ETL with complex transformations or when you require extreme control over connectors—use Glue or custom ETL instead.

## DR strategy
- Ensure destination storage (S3/Redshift) is replicated and maintain flow definitions in source control. For critical flows, run parallel flows in another region or establish replayability from source systems.
