# Q03: Encryption in Transit — TLS Termination, ACM Certificates, and End-to-End Encryption

## Question

A healthcare company is migrating a patient portal to AWS. The application architecture is:

```
Users → CloudFront → ALB → ECS Fargate (containers) → Aurora MySQL
```

The compliance officer requires:

1. **HTTPS everywhere**: All traffic from the user's browser to the database must be encrypted in transit — no plaintext HTTP at any hop
2. **TLS certificate management**: Certificates must auto-renew without manual intervention. The portal domain is `patient.example.com`
3. **End-to-end TLS**: Traffic between ALB and ECS containers must be encrypted (not just browser to ALB). The containers run on port 443 with their own TLS certificates
4. **CloudFront to ALB encryption**: CloudFront must connect to the ALB over HTTPS. The ALB must reject any HTTP connection from CloudFront
5. **Database encryption in transit**: Application connections to Aurora must use TLS. Non-TLS connections must be refused
6. **Certificate origin**: Public-facing certificates must come from a trusted CA. Internal certificates (ALB-to-container) can use private CAs
7. **Minimum TLS version**: TLS 1.2 minimum everywhere — no TLS 1.0 or 1.1

Which configuration satisfies all requirements?

## Options

- **A.** CloudFront: custom SSL certificate from ACM (us-east-1) for `patient.example.com`; viewer protocol policy = "redirect-to-https"; origin protocol policy = "https-only"; minimum origin TLS = TLS 1.2. ALB: HTTPS listener (port 443) with ACM public certificate for `patient.example.com`; security policy = `ELBSecurityPolicy-TLS13-1-2-2021-06` (TLS 1.2 minimum); target group health check over HTTPS. ECS containers: TLS-enabled with ACM Private CA-issued certificates; containers terminate TLS on port 443. Aurora: `require_secure_transport = ON` parameter; RDS CA bundle in the connection string for certificate verification. All ACM public certificates auto-renew. ACM Private CA certificates auto-renew via ACM.
- **B.** CloudFront: default CloudFront certificate (`*.cloudfront.net`). ALB: HTTP listener (port 80) with redirect to HTTPS. ECS containers: plaintext HTTP on port 8080. Aurora: default settings (TLS optional).
- **C.** CloudFront: ACM certificate for `patient.example.com`. ALB: HTTPS listener with a self-signed certificate. ECS: TLS with self-signed certificates. Aurora: `require_secure_transport = ON`. Manual certificate replacement every 12 months.
- **D.** CloudFront: ACM certificate. ALB: HTTPS with ACM certificate. ECS: TLS terminated at ALB only (containers receive plaintext on port 80). Aurora: TLS enabled. Third-party CA certificates with 2-year validity, manually uploaded to ACM.

## Answers

### A. ACM public + ACM Private CA + full TLS chain + Aurora TLS enforcement — ✅ Correct

This configuration encrypts traffic at every hop with proper certificate management:

**Hop 1: User browser → CloudFront (HTTPS)**

- **CloudFront custom SSL certificate**: ACM public certificate for `patient.example.com` deployed in **us-east-1** (CloudFront requires certificates in us-east-1 regardless of where other resources are deployed).
- **Viewer protocol policy = "redirect-to-https"**: Any HTTP request from the browser is automatically 301-redirected to HTTPS. This ensures ALL user traffic is encrypted.
- **Minimum TLS version**: CloudFront security policy can enforce TLS 1.2 minimum (`TLSv1.2_2021` or newer policy). Clients with TLS 1.0/1.1 are rejected.
- **ACM auto-renewal**: ACM public certificates are free, auto-validated (via DNS CNAME validation record), and auto-renewed 60 days before expiry. No manual certificate management.

**Hop 2: CloudFront → ALB (HTTPS)**

- **Origin protocol policy = "https-only"**: CloudFront connects to the ALB exclusively over HTTPS. No fallback to HTTP.
- **Minimum origin SSL protocol = TLS 1.2**: CloudFront negotiates at least TLS 1.2 with the ALB origin.
- **ALB HTTPS listener**: ALB listens on port 443 with its own ACM public certificate for `patient.example.com`. The ALB has NO HTTP listener (port 80) — it physically cannot accept unencrypted connections.
- **ALB security policy**: `ELBSecurityPolicy-TLS13-1-2-2021-06` enforces TLS 1.2 as the minimum version. TLS 1.0 and 1.1 cipher suites are excluded from this policy.
- **Certificate validation**: CloudFront validates the ALB's certificate against trusted CAs. ACM public certificates are issued by Amazon's public CA (trusted by all major browsers and operating systems).

**Hop 3: ALB → ECS containers (HTTPS)**

- **End-to-end TLS**: The ALB target group uses HTTPS protocol (port 443). The ALB re-encrypts traffic before forwarding to the ECS containers. The containers run a TLS-enabled web server (e.g., Nginx with TLS, or application framework TLS).
- **ACM Private CA certificates**: Internal ALB-to-container communication uses certificates issued by AWS Certificate Manager Private Certificate Authority (ACM Private CA). Private CA certificates:
  - Are NOT publicly trusted (browsers won't trust them) — that's fine because only the ALB connects to the containers.
  - Auto-renew via ACM — no manual certificate rotation.
  - Cost: ACM Private CA charges $400/month per CA + $0.75 per certificate issued.
- **Why Private CA for internal traffic**: Public CA certificates require domain validation (proving you own the domain). Internal services often use internal hostnames or IP addresses. Private CA certificates can be issued for any Subject name without external validation — appropriate for service-to-service communication.
- **ALB health checks over HTTPS**: The target group health check uses HTTPS to verify the container is healthy. The ALB trusts the container's private CA certificate for health check purposes.

**Hop 4: ECS containers → Aurora MySQL (TLS)**

- **`require_secure_transport = ON`**: This Aurora MySQL parameter (set in the DB parameter group) forces ALL client connections to use TLS. Non-TLS connections are rejected with: `ERROR 3159: Connections using insecure transport are prohibited.`
- **RDS CA certificate**: Aurora uses an RDS-managed CA certificate. The application's connection string includes the RDS CA bundle (`rds-combined-ca-bundle.pem`) to verify the Aurora certificate on connection:
  ```
  mysql -h cluster.abcdef.us-east-1.rds.amazonaws.com --ssl-ca=rds-combined-ca-bundle.pem --ssl-mode=VERIFY_FULL
  ```
- **`ssl-mode=VERIFY_FULL`**: Verifies both the certificate chain AND the hostname. Prevents man-in-the-middle attacks.
- **RDS CA bundle rotation**: RDS periodically rotates its CA certificate (every 5 years). AWS publishes new CA bundles, and applications must update the bundle in their connection configuration. RDS supports staging two CAs simultaneously to enable zero-downtime rotation.

**TLS 1.2 minimum — enforced at every hop:**
- CloudFront → TLS security policy
- ALB → ELB security policy
- ECS containers → web server TLS configuration (e.g., Nginx `ssl_protocols TLSv1.2 TLSv1.3;`)
- Aurora → RDS enforces TLS 1.2+ by default for MySQL 8.0

### B. Default certificate + HTTP + plaintext containers — ❌ Incorrect

- **Default CloudFront certificate (`*.cloudfront.net`)**: The portal would be accessed at `d1234.cloudfront.net` instead of `patient.example.com`. This is unprofessional and doesn't meet brand requirements. More critically, if the custom domain is used with a CNAME to CloudFront but without a matching SSL certificate, browsers show certificate mismatch errors.
- **ALB HTTP listener with redirect**: While the redirect sends clients to HTTPS, there's a brief moment where the initial request (potentially containing session cookies or credentials) travels over plaintext HTTP before the redirect. HSTS headers mitigate this for subsequent requests, but the first request is vulnerable.
- **Plaintext ECS containers (port 8080)**: Traffic from ALB to containers is unencrypted. In a VPC, this traffic traverses AWS's internal network — while AWS encrypts VPC traffic at the physical layer, compliance frameworks (HIPAA, PCI-DSS) often require encryption at the application layer for PHI/PCI data. "ALB terminates TLS, sends plaintext to containers" fails strict end-to-end encryption requirements.
- **Aurora default TLS (optional)**: Without `require_secure_transport = ON`, Aurora accepts both TLS and non-TLS connections. A misconfigured application or a developer testing locally might connect without TLS, sending PHI in plaintext. Enforcement is critical for compliance.

### C. Self-signed certificates + manual rotation — ❌ Incorrect

- **Self-signed ALB certificate**: ALB does NOT support self-signed certificates. ALB HTTPS listeners require certificates from ACM or imported certificates from a recognized CA. A self-signed certificate uploaded to ACM would be rejected by clients (browsers show security warnings for self-signed certificates on public-facing sites).
- **Self-signed container certificates**: While technically functional for ALB-to-container TLS, self-signed certificates require manual rotation. When they expire (every 12 months in this option), the application goes down until new certificates are deployed. ACM Private CA with auto-renewal eliminates this risk.
- **Manual rotation every 12 months**: Human-dependent certificate rotation is error-prone. Missed rotations cause outages. ACM auto-renewal exists specifically to eliminate this operational risk. For a healthcare application with uptime requirements, manual rotation is unacceptable.

### D. TLS terminated at ALB only + third-party CA with manual upload — ❌ Incorrect

- **TLS terminated at ALB, plaintext to containers**: The requirement explicitly states "traffic between ALB and ECS containers must be encrypted" — end-to-end TLS. Option D terminates TLS at the ALB and sends plaintext HTTP to containers on port 80. This breaks the end-to-end encryption requirement.
- **Third-party CA certificates manually uploaded to ACM**: ACM does support imported certificates (from DigiCert, GlobalSign, etc.), but:
  - Imported certificates do NOT auto-renew in ACM. You must manually re-import before expiry. ACM sends expiration notifications, but the renewal itself is manual.
  - ACM public certificates are free and auto-renewing. Third-party CA certificates cost $100-$500/year per certificate and require manual intervention.
  - There is no compliance or technical reason to prefer third-party CA certificates over ACM public certificates for AWS-hosted services.
- **2-year certificate validity**: Since September 2020, public CAs issue certificates with a maximum validity of 397 days (13 months). 2-year public certificates are no longer available from any trusted CA.

## Recommendations

- **ACM for all AWS-hosted TLS**: Use ACM public certificates for public-facing endpoints (CloudFront, ALB, API Gateway). They're free, auto-renewing, and trusted globally.
- **ACM Private CA for internal TLS**: Use Private CA for service-to-service encryption (ALB-to-container, container-to-container). Cost: $400/month per CA — justified for organizations with many internal services.
- **CloudFront certificates MUST be in us-east-1**: This is a common exam gotcha. Even if your ALB is in eu-west-1, the CloudFront certificate must be provisioned in us-east-1.
- **DNS validation over email validation**: For ACM certificate issuance, use DNS validation (add a CNAME record). It's automatable, works with Route 53, and supports auto-renewal. Email validation requires manual approval and may fail if email isn't monitored.
- **HSTS headers**: Add `Strict-Transport-Security: max-age=31536000; includeSubDomains` to enforce HTTPS in browsers even if a user types `http://`. CloudFront can add this via response headers policy.
- **Certificate pinning**: Avoid certificate pinning in mobile apps — ACM rotates certificates automatically, and a pinned certificate will cause connection failures after rotation.

## Relevant Links

- [ACM Public Certificates](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html)
- [ACM Private CA](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html)
- [CloudFront SSL/TLS](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html)
- [ALB HTTPS Listener](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)
- [ALB Security Policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html)
- [Aurora TLS Connections](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Security.html)
- [RDS Certificate Rotation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL-certificate-rotation.html)
