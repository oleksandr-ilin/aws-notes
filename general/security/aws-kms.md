Overview
Managed key management service for encryption keys, including multi‑region keys and CMKs.

Tasks
- Create and manage keys, grants, rotation, and use in services for encryption.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| CloudHSM | FIPS-level HSM | Higher cost & management |
| Customer-managed keys off‑AWS | Control | Complex integration |

Limitations
- API quotas and regional keys; multi-region keys add complexity.

Price info
- Per‑key-month and per‑API-request charges for some operations.

Network & multi-region
- Keys are regional unless using multi‑region keys; replicate data encryption strategies accordingly.

Popular use cases
- Envelope encryption for S3, EBS, RDS, and custom app encryption.

When Not to Use
- HSM-level compliance requirements—use CloudHSM.

DR strategy
- Backup key policies and ensure recovery IAM roles; plan for regional key unavailability.
