# Q01: Improving Secrets Management, Credential Rotation, and Compliance Auditing

## Question

A security audit of a company's AWS environment (3 accounts: Production, Staging, Shared Services) revealed the following findings:

**Finding 1 — Hardcoded credentials (Critical)**
- 12 Lambda functions have database credentials hardcoded in environment variables (plaintext visible in the Lambda console)
- 5 ECS task definitions reference RDS credentials as plaintext environment variables
- A developer committed an AWS access key to a public GitHub repository (revoked within 2 hours, but the window existed)

**Finding 2 — No credential rotation (High)**
- RDS master password hasn't been changed in 14 months
- 30 IAM users have access keys older than 12 months (some never rotated)
- Third-party API keys stored in SSM Parameter Store (SecureString) but never rotated

**Finding 3 — No compliance monitoring (Medium)**
- No mechanism to detect NEW instances of hardcoded credentials
- No alerting when credentials exceed age thresholds
- No automated remediation — all credential issues found by manual audit

The CISO requires: (a) eliminate all plaintext credentials from Lambda and ECS, (b) implement automatic rotation with zero application downtime, (c) continuous compliance monitoring with automated alerting, and (d) prevent credential leakage to code repositories.

Which improvement strategy addresses all findings?

## Options

- **A.** (a) Migrate all database credentials and API keys to **AWS Secrets Manager**. Lambda functions reference secrets via the Secrets Manager ARN in the function configuration (AWS handles retrieval, caching, and injection). ECS task definitions reference secrets using the `secrets` property with `valueFrom: arn:aws:secretsmanager:...` — ECS agent retrieves the secret at task launch. Remove all plaintext environment variables. (b) Enable **Secrets Manager automatic rotation** for RDS credentials using the Lambda rotation function template. Rotation schedule: every 30 days. For multi-user rotation (zero-downtime): Secrets Manager creates a second user, rotates between them — the application always has valid credentials. For IAM access keys: use **AWS Config rule** `access-keys-rotated` (maxAccessKeyAge: 90) + **SSM Automation** to disable keys > 90 days and notify the owner. (c) Deploy **AWS Config rules**: `secretsmanager-rotation-enabled-check` (verify rotation is configured), `secretsmanager-scheduled-rotation-success-check` (verify rotation succeeded), `lambda-function-settings-check` (detect Lambda env vars containing credential patterns). **EventBridge + SNS** alerting on Config compliance changes. (d) Enable **Amazon CodeGuru Reviewer** on all repositories — it scans for leaked credentials, hardcoded secrets, and insecure patterns in pull requests. Additionally, configure **GitHub secret scanning** (or equivalent) to detect AWS access keys in commits.
- **B.** Move all credentials to SSM Parameter Store (SecureString). Rotate credentials manually every quarter. Use Trusted Advisor to check for exposed access keys. Block commits with a pre-commit Git hook on developer machines.
- **C.** Use IAM roles everywhere — no passwords, no access keys. Replace RDS with DynamoDB (uses IAM for authentication). Remove all credential management tools.
- **D.** Encrypt all environment variables using Lambda's built-in encryption with a KMS key. Keep credentials in environment variables but encrypt them. Rotate manually using a calendar reminder. Monitor with CloudTrail.

## Answers

### A. Secrets Manager + automatic rotation + Config rules + CodeGuru — ✅ Correct

**Finding 1 — Eliminating Plaintext Credentials:**

- **Secrets Manager for centralized secret storage**:
  - Store each credential as a Secrets Manager secret: RDS credentials, API keys, connection strings.
  - Secrets are encrypted at rest with KMS (customer-managed or AWS-managed key). IAM policies control who can access each secret.
  - **Secret versioning**: Secrets Manager maintains versions (`AWSCURRENT`, `AWSPREVIOUS`). During rotation, both versions are valid simultaneously — ensuring zero-downtime transitions.

- **Lambda integration**:
  - AWS Lambda natively integrates with Secrets Manager. In the Lambda function configuration, specify secrets to retrieve:
    ```json
    {
      "Variables": {},
      "Secrets": [
        {
          "SecretArn": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/rds-creds",
          "SecretKey": "DB_PASSWORD",
          "EnvironmentVariable": "DB_PASSWORD"
        }
      ]
    }
    ```
  - Lambda retrieves the secret at function initialization and injects it as an environment variable. The secret value is NOT visible in the Lambda console — only the ARN reference is visible.
  - Lambda caches the secret value for the duration of the execution environment (warm container). When the secret rotates, the next cold start retrieves the new value.

- **ECS integration**:
  - ECS task definitions support Secrets Manager references in the `secrets` container definition:
    ```json
    {
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/rds-creds:password::"
        }
      ]
    }
    ```
  - The ECS agent retrieves the secret when launching the task. The secret value is injected as an environment variable inside the container but is NOT stored in the task definition itself.
  - **IAM task execution role**: The ECS task execution role must have `secretsmanager:GetSecretValue` permission for the specific secret ARN.

**Finding 2 — Automatic Rotation:**

- **Secrets Manager multi-user rotation for RDS (zero-downtime)**:
  - Secrets Manager ships with Lambda rotation function templates for RDS MySQL, PostgreSQL, Oracle, SQL Server, and generic databases.
  - **Single-user rotation**: Rotates the password of a single database user. During the rotation window (seconds), the old password is invalidated and the new password is set. Applications using the old cached password may fail for a few seconds.
  - **Multi-user rotation** (recommended for production): Creates TWO database users (e.g., `app_user_1` and `app_user_2`). Secrets Manager alternates: rotates `app_user_1`'s password while `app_user_2` is active. Applications always connect with the currently active user. **Zero-downtime** — at no point are both users invalid.
  - **Rotation schedule**: Every 30 days (configurable). Secrets Manager automatically invokes the rotation Lambda, which: generates a new password, updates the RDS user, updates the secret value, and tests the new credentials.

- **IAM access key rotation with Config + SSM**:
  - Config rule `access-keys-rotated` with `maxAccessKeyAge: 90` evaluates all IAM users.
  - Keys > 90 days → `NON_COMPLIANT` → notification via EventBridge → SNS to the key owner: "Rotate your access key within 30 days."
  - Keys > 120 days → SSM Automation runbook disables the key: `iam:UpdateAccessKey Status=Inactive`. The user must create a new key to regain access.
  - Keys > 180 days → SSM Automation deletes the key.
  - **Best practice**: Wherever possible, replace IAM access keys with IAM roles (EC2 instance profiles, ECS task roles, Lambda execution roles). Keys should be used only for external services that can't assume AWS roles.

**Finding 3 — Continuous Compliance Monitoring:**

- **Config rules for secrets compliance**:
  - `secretsmanager-rotation-enabled-check`: Ensures all Secrets Manager secrets have rotation enabled. New secrets created without rotation are flagged as `NON_COMPLIANT`.
  - `secretsmanager-scheduled-rotation-success-check`: Verifies that the last rotation attempt succeeded. Catches broken rotation functions (e.g., Lambda rotation function has a bug, IAM permission is missing).
  - `lambda-function-settings-check`: Custom Config rule (Lambda-backed) that scans Lambda function environment variables for patterns matching credentials (e.g., regex for `AKIA`, `password=`, `secret_key=`). Flags functions with potential hardcoded credentials.

- **EventBridge alerting**: Config compliance state changes generate EventBridge events. Rules match `NON_COMPLIANT` state → SNS notification to the security team. Re-evaluation happens on every configuration change (continuous monitoring, not periodic).

**Finding 4 — Preventing Repository Leakage:**

- **Amazon CodeGuru Reviewer**: Integrates with GitHub, CodeCommit, and Bitbucket. Automatically reviews pull requests for:
  - Hardcoded credentials (AWS access keys, database passwords, API tokens)
  - Security vulnerabilities (SQL injection, insecure deserialization)
  - Code quality issues
  - Provides inline comments on the PR with recommendations.

- **GitHub secret scanning**: GitHub's built-in feature (free for public repos, available for private repos with GitHub Advanced Security) scans commits for known credential patterns (AWS access keys, OAuth tokens, etc.) and alerts immediately.

### B. SSM Parameter Store + manual rotation + Git hooks — ❌ Incorrect

- **SSM Parameter Store (SecureString)**: Stores encrypted parameters, but:
  - Parameter Store does NOT support automatic rotation. You must build custom rotation logic (Lambda + EventBridge schedule) to rotate secrets in Parameter Store. Secrets Manager has built-in rotation with pre-built Lambda templates for RDS.
  - Parameter Store `SecureString` is suitable for configuration values that rarely change (API endpoints, feature flags). For credentials that must rotate, Secrets Manager is purpose-built.
  - Parameter Store Standard tier is free (up to 10,000 parameters). Secrets Manager charges $0.40/secret/month. For credentials with rotation requirements, Secrets Manager's cost is justified by the built-in rotation automation.

- **Manual quarterly rotation**: Every manual rotation is a risk: the rotation might fail (wrong password set), the application might not be updated (configuration lag), or the rotation might be forgotten (calendar reminder dismissed). Automatic rotation eliminates all three risks.

- **Pre-commit Git hooks on developer machines**: Git hooks are local — each developer must install them. A developer who reinstalls their machine, uses a different computer, or intentionally bypasses the hook (`git commit --no-verify`) will not be protected. Server-side scanning (CodeGuru Reviewer, GitHub secret scanning) is mandatory because it can't be bypassed.

### C. IAM roles only — ❌ Incorrect

- **"Use IAM roles everywhere"**: IAM roles work for AWS service authentication (Lambda → DynamoDB, ECS → S3). But:
  - RDS database authentication: IAM authentication is supported for RDS MySQL and PostgreSQL, but has limitations: 256 new connections/second limit per instance, 20 IAM DB connections per instance. For high-volume production databases, password authentication with Secrets Manager rotation is more reliable.
  - Third-party API keys: External APIs (Stripe, Twilio, SendGrid) use API keys — they don't understand IAM roles. These keys must be stored in Secrets Manager.
  - Replacing RDS with DynamoDB just to avoid password management is a drastic re-architecture driven by an operational concern. The database technology should match the access pattern, not the authentication mechanism.
- IAM roles should be used wherever possible, but Secrets Manager fills the gaps where IAM roles can't.

### D. Encrypted environment variables + manual rotation — ❌ Incorrect

- **Lambda encrypted environment variables**: Lambda can encrypt environment variables at rest with a KMS key. However:
  - The environment variables are decrypted at runtime and available to the function — anyone with `lambda:GetFunction` permission can read the decrypted values. The encryption protects at rest only, not from authorized AWS users.
  - The credential value is still stored IN the Lambda configuration — it's not externalized to a secret management service. Updating the credential requires updating the Lambda configuration and redeploying.
  - No rotation automation — changing the password requires: updating the database, updating EVERY Lambda function that uses it, and deploying all functions. With Secrets Manager, you update the secret once — all functions retrieve the new value on next cold start.
- **Calendar reminder for rotation**: Humans forget. Calendar reminders are dismissed. Automated rotation with Secrets Manager is reliable, auditable, and consistent.
- **CloudTrail for monitoring**: CloudTrail logs API calls (who called `GetSecretValue`, who modified a Lambda function). It doesn't evaluate compliance posture (are all secrets rotated? do any Lambda functions have hardcoded credentials?). Config rules provide continuous compliance evaluation; CloudTrail provides audit trail.

## Recommendations

- **Secrets Manager is the default for credentials that rotate**: Database passwords, API keys, OAuth tokens. Use Parameter Store for non-sensitive configuration (feature flags, endpoints, tuning parameters).
- **Multi-user rotation for production databases**: Eliminates the brief authentication failure window during rotation. Worth the complexity of managing two database users.
- **Scan ALL repositories**: Enable CodeGuru Reviewer (or equivalent) and GitHub secret scanning. A single committed credential can compromise an entire environment.
- **Minimize IAM access keys**: Every IAM access key is a long-lived credential that must be managed, rotated, and monitored. Replace with IAM roles wherever possible. For remaining keys, enforce Config rule-based rotation.
- **Secrets Manager resource policy**: Restrict secret access by account, OU, or service using resource policies. Production secrets should only be accessible from the Production account.

## Relevant Links

- [Secrets Manager Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [Secrets Manager Multi-User Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets-two-users.html)
- [ECS Secrets Manager Integration](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data-secrets.html)
- [Lambda Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_lambda.html)
- [Config Secrets Manager Rules](https://docs.aws.amazon.com/config/latest/developerguide/secretsmanager-rotation-enabled-check.html)
- [CodeGuru Reviewer](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/welcome.html)
