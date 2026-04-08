Overview
AWS Transfer Family provides managed protocols (SFTP/FTPS/FTP) integrated with S3 or EFS for partner and legacy file transfers.

Tasks
- Create Transfer endpoint, connect user authentication (SCP/LDAP/Service-managed), map storage to S3/EFS, monitor transfers.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Self-hosted SFTP on EC2 | Full control | Ops overhead |
| Third-party managed SFTP | Feature-rich | Extra cost

Limitations
- Protocol-specific limits and endpoint per-hour costs.

Price info
- Endpoint-hour + per-GB data transfer and per-user charges may apply.

Network & multi-region
- Endpoints are regional; ensure network path to S3/EFS and consider VPC endpoints.

Popular use cases
- Partner file exchanges, lift-and-shift of legacy SFTP services.

When Not to Use
- If you can migrate partners to HTTP APIs or S3-native interfaces.

DR strategy
- Use replicated S3 buckets and endpoint configurations stored as code for fast rebuild.
