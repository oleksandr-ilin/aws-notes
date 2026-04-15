# Security Controls

## Knowledge of:

- IAM
- Route tables, security groups, and network ACLs
- Encryption options for data at rest and data in transit
- AWS service endpoints
- Credential management services
- AWS managed security services (for example, AWS Shield, AWS WAF, Amazon GuardDuty, AWS Security Hub)

## Skills in:

- Specifying IAM users and IAM roles that adhere to the principle of least privilege access
- Specifying inbound and outbound network flows by using security group rules and network ACL rules
- Developing attack mitigation strategies for large-scale web applications
- Developing encryption strategies for data at rest and data in transit
- Specifying service endpoints for service integrations
- Developing strategies for patch management to remain compliant with organizational standards


## What to understand

- How to design IAM policies with least privilege, using conditions, resource ARNs, and permission boundaries
- Security group (stateful) vs NACL (stateless) behavior and layering
- WAF + Shield + CloudFront for DDoS and application-layer attack mitigation
- Encryption at rest: KMS, CloudHSM, service-specific encryption (S3 SSE, RDS, EBS)
- Encryption in transit: TLS termination at ALB/CloudFront, end-to-end TLS, certificate management with ACM
- VPC endpoints (Gateway vs Interface) to keep traffic off the public internet
- Secrets Manager vs Parameter Store for credential management
- GuardDuty + Security Hub + Inspector for managed threat detection and compliance
- Patch management with Systems Manager Patch Manager


## Services to know

- `AWS IAM` — users, roles, policies, permission boundaries
- `AWS WAF` — web application firewall
- `AWS Shield` — DDoS protection
- `Amazon GuardDuty` — threat detection
- `AWS Security Hub` — centralized security findings
- `Amazon Inspector` — vulnerability scanning
- `AWS KMS` — managed encryption keys
- `AWS CloudHSM` — dedicated hardware security modules
- `AWS Certificate Manager` — TLS certificates
- `AWS Secrets Manager` — secret rotation and storage
- `AWS Systems Manager Parameter Store` — config and secret storage
- `AWS PrivateLink / VPC Endpoints` — private service connectivity

## What to expect

One of the most important considerations when designing a new solution is security, which brings us to the third task statement in domain 2: Determine security controls based on requirements.  To prepare for questions related to this task statement, review the AWS Well-Architected Framework Security pillar. Exam questions will touch on all focus areas in the security pillar, but dive deeper into Identity and Access Management, Infrastructure Protection, and Data Protection.  When answering questions, pay close attention to the requirements. And be familiar with each of the following services and how they could be used together to provide defense-in-depth: Amazon GuardDuty for threat detection and security visibility, and Amazon Security Hub for security best practice checks, aggregated alerts, and automated remediation supportYou might also be asked to consider managed security services for attack mitigation for large-scale web applications. AWS WAF to protect web applications and APIs from common exploits and bots, and AWS Shield for managed distributed denial of service (DDoS) protection. Ensure you know what OSI layer of protection that AWS security services offer. When assessing the design of a new solution on the exam, you might be asked to evaluate an identity and access management strategy or configuration. This requires skills in specifying IAM users and IAM roles that adhere to the principle of least privilege access and meet stated requirements.  You should also be familiar with credential management and how each service supports best practices around credential rotation and auditing. When requirements allow, know how to configure IAM for use with temporary credentials instead of long-term credentials. Next, be familiar with how AWS Secrets Manager facilitates credential rotation, controls access to secrets, and works with monitoring and logging services to support oversight and audits. Make sure you understand how to use Config to create Config Rules to automate credential auditing and to initiate remediation actions.What if you need to ensure that all Amazon EC2 instances are using an approved AMI in production, but allow developers to use non-approved AMIs. What solution would you implement? Config is a great choice here because it also evaluates your AWS resource configurations, snapshots, and retrieves historical configs too. In addition to identity and access management, you also need to consider infrastructure protection when designing new solutions. When evaluating solutions on the exam, consider control methodologies, such as defense in depth.  You will need to be able to take a given set of requirements, apply them within a layered defense strategy, and identify the appropriate route table, security group, and network access control list (network ACL) configurations.  You should also be able to recognize opportunities to protect your network traffic by specifying service endpoints for service integrations. This includes recognizing the different configuration requirements of interface endpoints and gateway endpoints. It also includes knowing when to use VPC endpoints with AWS PrivateLink to securely connect your VPCs, AWS services, or on-premises applications.   Protecting the infrastructure of your newly deployed solutions also requires developing strategies for patch management to remain compliant with organizational standards.  For the exam, you should be familiar with Patch Manager, a capability of AWS Systems Manager, and when it is the correct solution for patching operating systems, and applications. Alternately, instead of updating servers in-place, you could update an Amazon EC2 instance’s Amazon Machine Image (AMI) and deploy the updated server as new.  When architecting new solutions, you need to have a plan for protecting data at rest and data in transit. Be sure to review encryption strategies.  For questions relating to encrypting data at rest, review the following: Identify which key management solution will meet the stated requirements. Do the requirements allow you to use AWS KMS to create, store, and rotate encryption keys? If they require you to create your own encryption keys, do you need to store them in AWS CloudHSM? Understand how to query AWS CloudTrail to audit the use of your encryption keys. Be prepared to answer questions about configuring your AWS services to encrypt all stored data by default. If an exam question requires automated validation and remediation, consider Config Rules to verify that newly created resources adhere to your encryption policies.For questions about encrypting data in transit, ensure you know how to: enforce encryption in transit by using VPN connections and defining HTTPS endpoints in your network components, such as load balancers and CloudFront distributions. Know how to secure network communications by configuring network components to work with TLS certificates. You can use a managed service like AWS Certificate Manager to provision, manage, and deploy public and private TLS certificates, And know when to use VPC endpoints and AWS PrivateLink, and how they protect traffic between VPCs or on-premises locations. Also, dive deeper into how service requests should be authenticated and securely processed.  A key service here is Amazon API Gateway. Ensure you know how to integrate Lambda proxy and Cognito along with DynamoDB to store the metadata? And know how to add autoscaling. That’s all for task statement 3, Determine security controls based on requirements. Be sure to take note of any services or strategies that you might have been unfamiliar with. 