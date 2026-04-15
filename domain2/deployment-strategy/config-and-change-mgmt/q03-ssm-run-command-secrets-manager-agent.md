# Q03: SSM Run Command, Secrets Manager, and Agent Deployment Planning

## Question

A company is deploying a new three-tier application on AWS. The architecture uses Auto Scaling groups of EC2 instances (Amazon Linux 2023) in private subnets. Post-deployment requirements:

1. **Server bootstrap**: After each new instance launches (via ASG scale-out), it must automatically: install the application, configure log shipping to CloudWatch Logs, set the correct timezone, and join an Active Directory domain.
2. **Database credentials**: The application connects to an Aurora PostgreSQL database. The database password must be rotated every 30 days automatically. The application must always use the current password without manual updates or restarts.
3. **On-demand operations**: The operations team needs to run ad-hoc commands across all application servers (e.g., "collect diagnostic logs," "restart the application service," "update a config file") without SSH access — security policy prohibits SSH/RDP.
4. **SSM Agent connectivity**: Instances are in private subnets with no internet access (no NAT Gateway, no Internet Gateway). SSM must still function.

Which combination of AWS services and configurations meets all requirements?

## Options

- **A.** (1) SSM State Manager associations with a custom SSM Document that runs on instance launch (via ASG lifecycle hook) — the document installs the app, configures CloudWatch agent, sets timezone, and runs `domain-join` via SSM. (2) AWS Secrets Manager with automatic rotation enabled (Lambda rotation function) — the application uses the Secrets Manager SDK `GetSecretValue` API to fetch the current password at connection time (no hardcoded credentials). (3) SSM Run Command for ad-hoc operations — operators select target instances by tag and execute pre-built SSM Documents. Session Manager for interactive terminal access (replaces SSH). (4) VPC Interface Endpoints for SSM: `ssm`, `ssmmessages`, `ec2messages`, and `secretsmanager` endpoints in the private subnets — SSM Agent communicates via PrivateLink without internet access.
- **B.** (1) EC2 User Data script for bootstrap. (2) RDS password stored in Parameter Store (standard). (3) SSH via a bastion host for ad-hoc commands. (4) NAT Gateway for SSM Agent internet access.
- **C.** (1) AWS OpsWorks Chef recipes for bootstrap. (2) Secrets Manager with rotation. (3) SSM Run Command. (4) SSM Agent communicates via S3 Gateway endpoint.
- **D.** (1) SSM State Manager for bootstrap. (2) Secrets Manager with rotation. (3) SSM Run Command. (4) Install a VPN client on each instance to connect to the Systems Manager service endpoint.

## Answers

### A. State Manager + Secrets Manager rotation + Run Command/Session Manager + VPC endpoints — ✅ Correct

This is the comprehensive Systems Manager-native approach:

- **(1) SSM State Manager for bootstrap**:
  - **State Manager associations**: Define a desired configuration state and automatically apply it to managed instances. When an ASG launches a new instance, the State Manager association detects the new instance (via tag targeting) and applies the SSM Document.
  - **Custom SSM Document (Automation or Command document)**: A sequence of steps:
    1. Install application packages via `aws:runShellScript`
    2. Install and configure CloudWatch Agent via `AWS-ConfigureAWSPackage` and `AmazonCloudWatch-ManageAgent`
    3. Set timezone via `timedatectl set-timezone`
    4. Domain join via `AWS-JoinDirectoryServiceDomain` managed document
  - **ASG lifecycle hook integration**: The ASG can pause instance launch until the bootstrap is complete. An EventBridge rule for ` EC2 Instance-launch Lifecycle Action` triggers the SSM Automation, which signals the lifecycle hook on completion.
  - **Why not User Data**: User Data runs once at first boot and has limited error handling. If a step fails, the instance launches unhealthy. State Manager re-applies the desired state on a schedule — if configuration drifts, it self-heals. State Manager also logs execution results to S3/CloudWatch.

- **(2) AWS Secrets Manager with automatic rotation**:
  - **Automatic rotation**: Secrets Manager invokes a Lambda rotation function every 30 days. The function: (1) creates a new password, (2) updates the Aurora database password, (3) updates the secret value, (4) tests the new credentials. This is a four-step rotation: `createSecret` → `setSecret` → `testSecret` → `finishSecret`.
  - **Application integration**: The application calls `secretsmanager:GetSecretValue` at connection time (or during connection pool refresh). It always gets the current, valid password. **No hardcoded credentials** — the secret ARN is the only configuration the application needs.
  - **Caching**: AWS provides a Secrets Manager caching client (available for Java, Python, .NET, Go, JavaScript) that caches the secret value locally and refreshes automatically. This reduces API calls and handles rotation seamlessly — the cached client detects when the secret has rotated and fetches the new value.
  - **Aurora integration**: Secrets Manager has native integration with Aurora — the `rds` managed rotation Lambda function handles Aurora-specific password update steps (modifying the master password, testing connections, managing staging labels).

- **(3) SSM Run Command + Session Manager — no SSH**:
  - **Run Command**: Execute SSM Documents against multiple instances simultaneously. Operators select targets by tag (e.g., `Environment=Production, Role=AppServer`), choose a document (e.g., `AWS-RunShellScript`), and execute. Output is streamed to S3/CloudWatch. No SSH key management, no bastion host, no open port 22.
  - **Rate control**: Run Command supports `max-concurrency` (e.g., "run on 10 instances at a time") and `max-errors` (e.g., "stop if 3 failures") — safe for production operations.
  - **Session Manager**: Provides interactive shell access (bash/PowerShell) through the SSM Agent — replaces SSH entirely. Sessions are logged to S3/CloudWatch for audit. IAM policies control who can start sessions and on which instances.
  - **Security**: No inbound ports needed. SSM Agent makes an outbound HTTPS connection to the SSM service (via VPC endpoint). All traffic is encrypted and authenticated via IAM.

- **(4) VPC Interface Endpoints for private subnet SSM**:
  - SSM Agent requires connectivity to 3 service endpoints:
    - `com.amazonaws.{region}.ssm` — for SSM API calls (Run Command, State Manager)
    - `com.amazonaws.{region}.ssmmessages` — for Session Manager connections
    - `com.amazonaws.{region}.ec2messages` — for SSM Agent polling for commands
  - Additionally, `com.amazonaws.{region}.secretsmanager` for the application to call Secrets Manager.
  - **VPC Interface Endpoints (PrivateLink)**: Create ENIs in the private subnets that route traffic to the AWS service without traversing the internet. The SSM Agent connects to these local ENIs via private IP addresses.
  - **No NAT Gateway needed**: This satisfies the "no internet access" requirement while allowing full SSM functionality.
  - **Additional endpoints** may be needed: `s3` (Gateway endpoint for SSM output logging), `logs` (CloudWatch Logs for log shipping), `monitoring` (CloudWatch metrics).

### B. User Data + Parameter Store + SSH + NAT Gateway — ❌ Incorrect

- **EC2 User Data**: Runs once at boot (unless `cloud-init` is reconfigured). No self-healing — if bootstrap partially fails, the instance runs in a degraded state. No execution history or audit trail. No centralized management — each ASG launch template contains its own user data script.
- **Parameter Store (Standard) for passwords**: Parameter Store stores values BUT:
  - Standard tier does NOT support automatic rotation. You must build custom rotation logic (Lambda + EventBridge scheduled rule).
  - Standard tier has a 10,000 parameter limit per Region and lower throughput (40 TPS standard operations).
  - Secrets Manager provides built-in rotation, cross-Region replication, and an Aurora-integrated rotation function — purpose-built for credentials.
- **SSH via bastion host**: Violates the "no SSH/RDP" security policy. Bastion hosts require: public subnet placement, inbound port 22 security group rule, SSH key management and distribution, and are a lateral movement risk if compromised.
- **NAT Gateway**: Requires an Internet Gateway in the VPC and a route to 0.0.0.0/0 — the private subnet can route to the internet. This may violate security policies requiring complete internet isolation. NAT Gateway also costs $0.045/GB processed — VPC endpoints are cheaper for AWS service traffic.

### C. OpsWorks + S3 Gateway endpoint — ❌ Incorrect

- **AWS OpsWorks**: OpsWorks (Chef/Puppet) is a legacy configuration management service. AWS recommends SSM for new deployments. OpsWorks added operational complexity (Chef server management, recipe development, Berkshelf dependency management) that SSM eliminates.
- **S3 Gateway endpoint for SSM**: The S3 Gateway endpoint only provides access to S3. SSM Agent requires connectivity to the `ssm`, `ssmmessages`, and `ec2messages` service endpoints — NOT S3. An S3 endpoint is a supplement (for output logging) but doesn't replace the required SSM Interface endpoints.
- Secrets Manager and Run Command portions are correct.

### D. VPN client on each instance — ❌ Incorrect

- **VPN client per instance**: Installing a VPN client on every instance to reach Systems Manager endpoints is over-engineered and fragile. VPN connections require: a VPN server (another managed component), client certificates per instance, bandwidth limitations, and adds latency to every SSM API call.
- VPC Interface Endpoints are the standard, AWS-recommended solution — no additional software installation, no VPN infrastructure, no certificate management.
- State Manager, Secrets Manager, and Run Command portions are correct.

## Recommendations

- **SSM Agent is pre-installed** on Amazon Linux 2, Amazon Linux 2023, and recent Windows AMIs. For custom AMIs or older OS versions, include SSM Agent installation in the AMI build pipeline (EC2 Image Builder).
- **VPC Endpoints required for private subnets**: Create Interface endpoints for `ssm`, `ssmmessages`, `ec2messages`. Add `s3` Gateway endpoint for output logging. Add `secretsmanager`, `logs`, `monitoring` as needed by your application.
- **Secrets Manager vs. Parameter Store for credentials**:
  - Secrets Manager: Built-in rotation, cross-Region replication, higher cost ($0.40/secret/month + $0.05/10K API calls).
  - Parameter Store Advanced: Supports expiration policies but no built-in rotation. Lower cost ($0.05/advanced parameter/month). No rotation Lambda included.
  - **Use Secrets Manager** for database passwords, API keys, and any credential that needs automatic rotation.
- **State Manager vs. User Data**: Use State Manager for ongoing configuration management (self-healing, scheduled compliance). Use User Data for one-time, idempotent bootstrap that cannot drift (installing the SSM Agent itself, for example).
- **Run Command best practices**: Create custom SSM Documents for common operations (restart app, collect logs, update config). Version control documents in Git. Use tag-based targeting (`aws:autoscaling:groupName`) for ASG instances.

## Relevant Links

- [SSM State Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-state.html)
- [SSM Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html)
- [SSM Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [Secrets Manager Automatic Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [VPC Endpoints for SSM](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html)
- [SSM Agent Prerequisites](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-prereqs.html)
