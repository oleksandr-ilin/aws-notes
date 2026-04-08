Overview
Secure storage, rotation, and retrieval of application secrets and credentials.

Tasks
- Store secrets, configure rotation lambdas, and grant least-privilege access via IAM.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Parameter Store (SSM) | Cheaper for simple secrets | Fewer rotation features |
| Vault (HashiCorp) | Multi-cloud & enterprise features | Operational overhead |

Limitations
- Cost per secret can add up for very large numbers of secrets; API request costs apply.

Price info
- Per-secret per-month plus API request charges for retrieval and rotation.

Network & multi-region
- Secrets are regional; replicate secrets or plan secret provisioning across regions.

Popular use cases
- DB credentials, API keys, TLS cert metadata, and rotating credentials for services.

When Not to Use
- Extremely high-volume short-lived credentials where per-secret cost is prohibitive.

DR strategy
- Backup secret metadata (not values) in IaC and ensure rotation lambdas are versioned; have bootstrap secrets for recovery.
