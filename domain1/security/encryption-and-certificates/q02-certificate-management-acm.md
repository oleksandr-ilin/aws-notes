# Q02: Certificate Management at Scale

## Question

A company operates 200 microservices across 5 AWS Regions. Each microservice uses HTTPS and requires SSL/TLS certificates. The certificates are used on Application Load Balancers, Amazon CloudFront distributions, and Amazon API Gateway endpoints. The company needs to:
- Automatically provision and renew SSL/TLS certificates
- Support both public-facing certificates and internal (private) certificates for service-to-service communication
- Centrally manage certificate lifecycle across all Regions
- Avoid manual certificate rotation to prevent outages from expired certificates

Which approach should the solutions architect recommend?

## Options

- **A.** Use AWS Certificate Manager (ACM) to provision public certificates for internet-facing services. Use ACM Private Certificate Authority (PCA) to issue private certificates for internal service-to-service communication. ACM automatically renews certificates deployed on ALB, CloudFront, and API Gateway.
- **B.** Purchase SSL/TLS certificates from a third-party Certificate Authority. Import them into ACM in each Region. Set up CloudWatch alarms to alert when certificates are nearing expiration. Manually renew and re-import certificates.
- **C.** Generate self-signed certificates using OpenSSL for all services. Import them into ACM. Configure automated renewal using a Lambda function that regenerates certificates monthly.
- **D.** Use AWS KMS to generate certificates for all services. KMS natively integrates with ALB and CloudFront for TLS termination.

## Answers

### A. ACM (public) + ACM Private CA (private) — ✅ Correct

This is the AWS-recommended approach for certificate management at scale:

**ACM for public certificates**:
- **Free** public certificates for use with integrated services (ALB, CloudFront, API Gateway, Elastic Beanstalk).
- **Automatic renewal**: ACM automatically renews certificates before expiration — no manual intervention.
- **DNS or email validation**: Prove domain ownership once; renewals happen automatically.
- **Multi-Region**: ACM certificates are Regional (except CloudFront, which requires us-east-1). Provision certificates in each Region where you need them.

**ACM Private Certificate Authority (PCA)**:
- Issues **private certificates** for internal service-to-service mTLS communication.
- Manages the full private CA hierarchy (root CA, subordinate CAs).
- Supports custom certificate fields, short-lived certificates, and certificate templates.
- Integrates with ACM for automated provisioning and renewal.
- Can issue certificates at scale (thousands) via API.

### B. Third-party CA + manual import — ❌ Incorrect

Imported certificates in ACM are **not automatically renewed** — ACM cannot renew certificates it didn't issue. Manual renewal of 200+ certificates across 5 Regions is operationally risky — a single missed renewal causes an outage. The cost of third-party certificates for 200 services is also significant when ACM public certificates are free.

### C. Self-signed certificates — ❌ Incorrect

Self-signed certificates are not trusted by browsers — they'll cause TLS warnings for public-facing services. While they can work for internal communication, managing self-signed certificates at scale (200 services) with a Lambda-based renewal system is fragile and custom. ACM Private CA provides proper private certificate issuance with managed renewal.

### D. KMS for certificates — ❌ Incorrect

KMS manages **encryption keys**, not certificates. KMS cannot generate X.509 certificates, doesn't issue certificate chains, and doesn't integrate with ALB/CloudFront for TLS termination. ACM is the certificate management service; KMS is the key management service.

## Recommendations

- ACM **public certificates are free** — always use ACM for certificates on integrated AWS services.
- ACM certificates can be deployed on: **ALB, NLB, CloudFront, API Gateway, Elastic Beanstalk, and CloudFormation**.
- ACM certificates **cannot be exported** (the private key never leaves ACM). For services not integrated with ACM (e.g., EC2-based web servers), use ACM PCA or import third-party certificates.
- **CloudFront certificates** must be in **us-east-1** regardless of the distribution's origin Region.
- **ACM PCA pricing**: ~$200/month per private CA + per-certificate fees. For large-scale private PKI, consider **short-lived certificates** (< 7 days) at reduced cost.
- Use **ACM + API Gateway** custom domain names for consistent HTTPS on API endpoints.

## Relevant Links

- [AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- [ACM Private CA](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html)
- [ACM Certificate Renewal](https://docs.aws.amazon.com/acm/latest/userguide/managed-renewal.html)
- [ACM Integrated Services](https://docs.aws.amazon.com/acm/latest/userguide/acm-services.html)
