# Amazon Managed Blockchain

## Service Overview
Amazon Managed Blockchain is a managed service that makes it easy to create and manage scalable blockchain networks using Hyperledger Fabric and Ethereum.

## Tasks this service can solve
- Consortium blockchain hosting and node management
- Smart contract deployment and transaction processing
- Governance for multi-party workflows

## Alternatives
| Name | Type | Short description | Pro vs Managed Blockchain | Cons vs Managed Blockchain | Price comparison |
|---|---|---:|---|---|---|
| Self-hosted Hyperledger / Ethereum | OSS | Run blockchain nodes on EC2 or on-prem | Full control and customization | Heavy ops and governance burden | Ops cost higher; managed service simplifies lifecycle |
| Third-party blockchain platforms | Commercial | Managed blockchain offerings | Enterprise SLAs and integrations | Vendor lock-in and cost | Commercial pricing varies; often higher than managed native AWS service |

## Limitations
- Transaction throughput and latency depend on network configuration and consensus mechanisms.
- Cost and complexity grow with node count and data retention.

## Price info
- Charged per-node-hour, data storage, and transaction-related operations; network membership fees may apply.

## Network & Multi-region considerations
- Managed Blockchain is regional; for multi-region requirements, use cross-region architectures and off-chain replication or application-level redundancy.
- Integrate with IAM, VPC endpoints, and CloudWatch for monitoring.

## Popular use cases
- Supply-chain provenance and multi-party settlement
- Identity and credential verification across organizations
- Auditable transaction ledgers for consortiums

## When Not to Use This Service
- Not suited for simple centralized ledgers or where traditional databases suffice; blockchain adds cost and complexity. For internal single-organization workflows, prefer RDS/DynamoDB.

## DR strategy
- Back up ledger data and snapshots; maintain disaster recovery procedures for node replacement and membership reconfiguration. For critical multi-party workflows, define off-chain backups and cross-region archival to S3.
