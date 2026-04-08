# Security

## Knowledge of:

- AWS Identity and Access Management (IAM) and AWS IAM Identity Center
- Route tables, security groups, and network ACLs
- Encryption keys and certificate management (for example, AWS Key Management Service [AWS KMS], AWS Certificate Manager [ACM])
- AWS security, identity, and compliance tools (for example, AWS CloudTrail, AWS Identity and Access Management Access Analyzer, AWS Security Hub, Amazon Inspector)

## Skills in:

- Evaluating cross-account access management
- Integrating with third-party identity providers
- Deploying encryption strategies for data at rest and data in transit
- Developing a strategy for centralized security event notifications and auditing


## Security pillar of the AWS Well-Architected Framework

- Security Foundations
- Identity and Access Management
- Data Protection


## What to know

- how to implement the principle of least privilege in AWS Organizations. For example, know the
  differences between `AdministratorAccess` and `PowerUserAccess`
- dive deeper into:
  - service control policies, 
  - IAM policies, 
  - resource policies, 
  - identity-based policies, 
  - bucket policies
- how to configure IAM policies and roles to support federated access
- how to use AWS IAM Identity Center to provide a single point of access for federated users
- how to use Amazon Cognito or IAM Identity Center to integrate with third-party identity providers for federated access
- how the AWS Security Token service generates temporary security credentials for federated users
- which AWS network and application protection services give you fine-grained protections at the host, network, and application-level boundaries for multiple AWS accounts across multiple Regions
- how to inspect and filter traffic to prevent unauthorized access and meet compliance requirements
- how to use AWS components like route tables to direct network traffic
- how to use `security groups` for controlling network access to your AWS resources
- how to use `network access control lists`, (or network `ACL`s), to provide stateless network control
- how to protect web applications from common attacks, like:
  - SQL injection or cross-site scripting, 
  - detecting and responding to targeted attacks such as distributed denial of service attacks (DDoS).
- Dive deeper into services such as:
  - `AWS Network Firewall` to provide stateful, managed,intrusion detection that scales automatically with your network traffic
  - `AWS Web Application Firewall` to control access to your HTTP and HTTPS content based on conditions that you specify, such as the IP addresses that requests originate from
- how `AWS Shield` provides automatic protection against **DDoS** attacks
- how `AWS Firewall Manager` simplifies your administration and maintenance tasks across multiple accounts
- encryption and certificate management.
  - Know the `AWS services` that by default, enforce encryption at rest, and encrypt your data automatically.
  - how to use `AWS Key Management Service` to create, store, and manage encryption keys
  - when to use `AWS CloudHSM` to satisfy compliance requirements and what levels both support
  - how to do certificate management and using `SSL TLS` certificates to secure network communications and establish the identity of resources
  - how to implement solution which provisions, manages, or deploys certificates for use with AWS services in AWS and your internal resources at scale
  - `AWS Certificate Manager` and how it integrates with AWS services, such as `Elastic Load Balancing` and `Amazon CloudFront`, to deploy SSL and TLS certificates on the service
  - how to using `Amazon API Gateway` to generate an `SSL/TLS` certificate for the gateway’s custom domain name
- compliance
  - how to configure and combine AWS services across multiple accounts for centralized security event notifications and auditing
  - how to monitor activity in your `AWS accounts` using `CloudTrail`
  - how to use `AWS Security Hub` to collect security data from across your accounts
  - `Amazon Inspector` to detect vulnerabilities
  - how to use `Amazon GuardDuty` to monitor your workloads for malicious activity.
  - how to use `AWS IAM Access Analyzer` to:
    - validate IAM policies
    - generate policies based on CloudTrail logs
    - identify resources in your accounts that are shared with an external entity

