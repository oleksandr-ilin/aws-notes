# Q04: Data-in-Transit Encryption Strategies

## Question

A healthcare company is migrating to AWS and must ensure all data in transit is encrypted to satisfy HIPAA requirements. The architecture includes:
- An Application Load Balancer serving a web application on ECS Fargate
- ECS tasks communicating with Amazon RDS (PostgreSQL) and Amazon ElastiCache (Redis)
- An internal API layer using Amazon API Gateway (private) accessed by backend services
- File transfers from on-premises systems via AWS Transfer Family to Amazon S3
- Replication of EBS snapshots to a DR Region

The security team requires end-to-end TLS enforcement with no exceptions and the ability to detect any unencrypted connections. Which combination of controls should the solutions architect implement? *(Choose 3)*

## Options

- **A.** Configure the ALB with an HTTPS listener using an ACM certificate. Set the target group to use HTTPS (port 443) for end-to-end encryption to the ECS tasks. Enable the `aws:SecureTransport` condition key in S3 bucket policies to deny unencrypted uploads. Configure RDS to require SSL by setting `rds.force_ssl = 1` in the parameter group.
- **B.** Use ACM to provision certificates for the ALB and terminate TLS at the load balancer. Use HTTP between the ALB and ECS tasks since traffic stays within the VPC. Enable S3 default encryption (SSE-S3) to protect uploads. Set RDS `rds.force_ssl = 0` since the VPC provides network isolation.
- **C.** Enable in-transit encryption for ElastiCache Redis by creating the cluster with `TransitEncryptionEnabled = true`. Configure ECS tasks with the Redis TLS endpoint. For the private API Gateway, use a VPC endpoint (interface type) which enforces HTTPS by default. Use VPC Flow Logs with Athena queries to detect non-TLS connections by identifying traffic on non-443 ports.
- **D.** Use AWS PrivateLink for all service-to-service communication to replace TLS. Place all resources in private subnets with no internet access. Use NACLs to block port 80 across all subnets. Rely on EBS encryption at rest for snapshot replication security.
- **E.** Configure AWS Transfer Family with the FTPS or SFTP protocol (not plain FTP). Enable EBS snapshot cross-Region copy encryption using a KMS key in the DR Region. Create an AWS Config custom rule to detect security groups allowing inbound port 80 (HTTP) and auto-remediate by removing the rule via SSM Automation.

## Answers

### A. ALB end-to-end HTTPS + S3 SecureTransport + RDS force SSL — ✅ Correct

- **ALB end-to-end HTTPS**: HTTPS listener decrypts at the ALB, then re-encrypts to the target group over HTTPS (port 443). ECS tasks must run a TLS-enabled server. This ensures encryption throughout — not just client-to-ALB.
- **`aws:SecureTransport` condition**: A bucket policy condition (`"aws:SecureTransport": "false"` → Deny) blocks any non-HTTPS S3 API requests. This enforces encryption for all uploads and downloads, including those from Transfer Family.
- **`rds.force_ssl = 1`**: Forces all connections to the PostgreSQL instance to use SSL. Non-SSL connection attempts are rejected at the database level. The SSL certificate is provided by RDS and verifiable by clients.

### B. TLS termination at ALB only — ❌ Incorrect

Terminating TLS at the ALB and using HTTP to backends means traffic between ALB and ECS is **unencrypted** inside the VPC. While VPC traffic is isolated, HIPAA requires encryption of all PHI in transit without exception — "VPC isolation" is not considered encryption. S3 default encryption (SSE-S3) is **at-rest** encryption, not in-transit. Setting `rds.force_ssl = 0` allows unencrypted database connections.

### C. ElastiCache TLS + API Gateway VPC endpoint + Flow Logs detection — ✅ Correct

- **ElastiCache in-transit encryption**: Must be enabled at cluster creation (`TransitEncryptionEnabled`). Once enabled, Redis clients must use TLS endpoints. AUTH tokens can add an extra layer of authentication.
- **Private API Gateway via VPC endpoint**: Interface VPC endpoints for API Gateway use HTTPS by default — all traffic is encrypted and stays on the AWS network.
- **VPC Flow Logs + Athena**: While Flow Logs don't inspect TLS content, monitoring traffic on non-standard ports (e.g., port 80, port 6379 without TLS) can flag potentially unencrypted communication patterns.

### D. PrivateLink as TLS replacement — ❌ Incorrect

PrivateLink provides **private connectivity** (traffic stays on AWS backbone), but it is **not encryption**. PrivateLink endpoints use HTTPS by default for AWS services, but for custom services, TLS must still be configured. Blocking port 80 via NACLs does not ensure encryption — services could use other unencrypted ports. EBS encryption at rest does not protect data during cross-Region snapshot copy transit (though AWS does encrypt cross-Region snapshot copies in transit automatically).

### E. Transfer Family SFTP/FTPS + EBS copy encryption + Config detection — ✅ Correct

- **Transfer Family SFTP/FTPS**: SFTP (SSH File Transfer) and FTPS (FTP over TLS) encrypt file transfers. Plain FTP transmits credentials and data in cleartext — must be avoided for HIPAA.
- **EBS cross-Region snapshot encryption**: When copying a snapshot cross-Region, specifying a KMS key in the destination Region encrypts the copy. AWS encrypts the data in transit during the copy operation.
- **Config rule + SSM auto-remediation**: Detects security groups allowing HTTP (port 80) and automatically removes the offending rule — a detective and corrective control that supports continuous compliance monitoring.

## Recommendations

- **End-to-end TLS principle**: Encrypt at every hop — client → ALB → backend → database. VPC isolation is a network control, not an encryption control.
- **Service-specific enforcement**: Each AWS service has its own mechanism for requiring TLS: RDS parameter groups, ElastiCache cluster settings, S3 bucket policies, Transfer Family protocol selection.
- **Certificate management**: Use ACM for ALB/CloudFront/API Gateway certificates (auto-renewing). For ECS tasks and custom servers, use ACM Private CA or manage certificates via Secrets Manager.
- **Detection layering**: Combine preventive controls (force_ssl, SecureTransport) with detective controls (Flow Logs, Config rules) for defense in depth.
- **AWS services encrypt cross-Region replication in transit**: S3 CRR, EBS snapshot copy, RDS snapshot copy, and DynamoDB Global Tables all encrypt replication traffic — but always verify and enforce with KMS keys.

## Relevant Links

- [RDS Force SSL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.SSL.html)
- [ElastiCache In-Transit Encryption](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/in-transit-encryption.html)
- [S3 SecureTransport Condition](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [ACM Private Certificate Authority](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html)
- [AWS Transfer Family Protocols](https://docs.aws.amazon.com/transfer/latest/userguide/create-server.html)
