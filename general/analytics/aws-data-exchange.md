Overview
AWS Data Exchange is a marketplace for subscribing to third-party datasets and exchanging data securely.

Tasks
- Browse publishers, subscribe to datasets, configure delivery and access.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Direct partner sharing | Simple | Manual setup

Limitations
- Licensing and provider SLAs vary per dataset.

Price info
- Pricing depends on dataset subscriptions and usage terms.

Network & multi-region
- Delivered to S3 buckets in specified regions; consider data residency.

Popular use cases
- Purchasing curated market data, demographic datasets, and enriched feeds.

When Not to Use
- If you require zero cost or internal-only datasets.

DR strategy
- Retain local copies in S3 with cross-region replication if needed.
