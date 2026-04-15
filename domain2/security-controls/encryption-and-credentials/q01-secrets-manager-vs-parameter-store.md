# Q01: Credential Management — Secrets Manager vs Parameter Store

## Question

A company runs microservices on ECS Fargate that need access to:
- Database credentials for RDS PostgreSQL (rotated every 30 days)
- A third-party API key that changes quarterly (manual rotation by the vendor)
- Application configuration values (feature flags, timeout settings) that change frequently
- A TLS private key used by the application for mTLS communication with a partner

Requirements:
- Database credentials must be rotated automatically without application downtime
- Secrets must be encrypted at rest with a customer-managed KMS key
- The application must retrieve the latest secret value without redeployment
- Cost should be minimized for non-sensitive configuration data
- Access to secrets must be auditable

Which credential management strategy should the solutions architect implement?

## Options

- **A.** Store database credentials in AWS Secrets Manager with automatic rotation enabled (30-day schedule) using the built-in RDS rotation Lambda. Store the third-party API key in Secrets Manager (manual rotation, rotation reminder notification). Store application configuration values in Systems Manager Parameter Store (Standard tier, String type). Store the TLS private key in Secrets Manager (SecureBinary type). Configure ECS task definitions to reference Secrets Manager ARNs and Parameter Store names as environment variables. Use a customer-managed KMS key for all Secrets Manager secrets.
- **B.** Store all values (DB credentials, API keys, configs, TLS key) in Secrets Manager with automatic rotation. Use a customer-managed KMS key for encryption.
- **C.** Store all values in Systems Manager Parameter Store SecureString parameters. Use a customer-managed KMS key for encryption. Build a custom Lambda function for database credential rotation.
- **D.** Store credentials in environment variables in the ECS task definition. Store the TLS key in an S3 bucket with server-side encryption. Use AWS Config to audit access.

## Answers

### A. Secrets Manager (secrets) + Parameter Store (config) — ✅ Correct

This optimally splits between the two services:
- **Secrets Manager for DB credentials**: Built-in RDS rotation using a Lambda function that updates the password in both Secrets Manager and the RDS instance. Supports single-user and alternating-user rotation strategies. No application downtime — the app retrieves the latest secret on the next call.
- **Secrets Manager for API key**: Even without automatic rotation, Secrets Manager provides versioning, encryption, and rotation reminder notifications. When the vendor provides a new key, update the secret value — the app gets it next retrieval.
- **Parameter Store for config values**: Standard-tier Parameter Store is free (up to 10,000 parameters). String type for non-sensitive feature flags and timeout settings. No encryption needed for non-sensitive data, saving KMS costs. Frequent changes are handled naturally — Parameter Store has no rotation complexity.
- **Secrets Manager for TLS private key**: SecureBinary type can store binary data (PEM/DER keys). Encrypted with CMK and versioned.
- **ECS integration**: Task definitions natively support pulling from both Secrets Manager (`valueFrom` with ARN) and Parameter Store (`valueFrom` with name) — values are injected at task launch without application code changes.
- **Auditability**: Both services log access via CloudTrail. Secrets Manager additionally logs rotation events.
- **Cost optimization**: Parameter Store Standard is free; Secrets Manager charges per-secret per-month (~$0.40). Using Parameter Store for non-sensitive config avoids unnecessary Secrets Manager costs.

### B. All in Secrets Manager — ❌ Partially incorrect

- Storing configuration values (feature flags, timeout settings) in Secrets Manager is functional but **unnecessarily expensive** — $0.40/secret/month for non-sensitive data that Parameter Store handles for free.
- Automatic rotation for ALL values adds unnecessary complexity: feature flags don't need rotation, and the third-party API key can't be auto-rotated (the vendor controls it).
- This works but violates the cost minimization requirement.

### C. All in Parameter Store — ❌ Partially incorrect

- Parameter Store SecureString can store secrets encrypted with KMS — functionally similar to Secrets Manager for storage.
- However, **Parameter Store has no built-in rotation**: you must build and maintain a custom Lambda function for database credential rotation. This Lambda needs to coordinate the password change in RDS and update the parameter — reimplementing what Secrets Manager provides natively.
- Custom rotation Lambda is a maintenance burden and security surface — if the Lambda fails, credentials may be out of sync between the parameter and the database.
- Parameter Store lacks rotation auditing, versioning stages (AWSCURRENT/AWSPREVIOUS), and rotation failure notifications.

### D. Environment variables + S3 — ❌ Incorrect

- **Environment variables in task definitions** are visible in the ECS console and API responses — credentials are exposed in plaintext to anyone with `ecs:DescribeTaskDefinition` permissions.
- Changing credentials requires updating the task definition and redeploying — violates the "without redeployment" requirement.
- **TLS private key in S3** is accessible to anyone with S3 read permissions — even with SSE, the key is decrypted on read. Secrets Manager provides tighter access control and auditability.
- **AWS Config** tracks resource configuration changes, not access patterns — CloudTrail is the correct audit tool for secret access.

## Recommendations

- **Rule of thumb**: If it needs rotation or is highly sensitive → Secrets Manager. If it's configuration or low-sensitivity → Parameter Store.
- **ECS native integration** with both services eliminates the need for application-level SDK calls to retrieve secrets at startup — use `secrets` and `environment` in container definitions.
- **Secrets Manager rotation**: Use the multi-user strategy (alternating users) for zero-downtime rotation — the old credential remains valid until the next rotation.
- **Parameter Store Advanced tier** ($0.05/parameter/month) adds parameter policies (expiration notifications) and higher throughput — consider for parameters needing lifecycle management.
- **Cross-account secrets**: Secrets Manager supports resource policies for cross-account access. Parameter Store does not — use Secrets Manager for shared secrets.

## Relevant Links

- [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [Secrets Manager RDS Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets-rds.html)
- [Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [ECS Secrets Integration](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data-secrets.html)
