# Q03: VPC Endpoint Types — Gateway vs. Interface

## Question

A company has an application running on EC2 instances in a private subnet (no internet access). The application needs to access Amazon S3 to download configuration files and Amazon SQS to process messages. The security team requires that all traffic to AWS services remains on the AWS private network and never traverses the public internet or a NAT Gateway.

Which VPC endpoints should the solutions architect configure?

## Options

- **A.** Create a Gateway VPC Endpoint for S3 and a Gateway VPC Endpoint for SQS. Update the private subnet route tables to route traffic through both gateway endpoints.
- **B.** Create a Gateway VPC Endpoint for S3 and an Interface VPC Endpoint for SQS. Update the route table for S3 and use the private DNS name for SQS.
- **C.** Create Interface VPC Endpoints for both S3 and SQS. Use private DNS names for both services.
- **D.** Configure a NAT Gateway in a public subnet and route S3 and SQS traffic through it. Traffic from the NAT Gateway to AWS services stays on the AWS network.

## Answers

### B. Gateway Endpoint (S3) + Interface Endpoint (SQS) — ✅ Correct

AWS supports two types of VPC endpoints, and only specific services support each type:
- **Gateway VPC Endpoints** are supported for **S3 and DynamoDB only**. They are free (no hourly or data processing charge), added as a route table entry, and route traffic to S3/DynamoDB via the AWS private network.
- **Interface VPC Endpoints** (powered by AWS PrivateLink) are supported for **most other AWS services**, including SQS, SNS, KMS, CloudWatch, etc. They create an ENI in the subnet with a private IP address. With private DNS enabled, the standard SQS endpoint (`sqs.us-east-1.amazonaws.com`) resolves to the private ENI IP.

This combination ensures all traffic stays private at optimal cost (free S3 gateway + priced SQS interface endpoint).

### A. Two Gateway Endpoints — ❌ Incorrect

Gateway VPC Endpoints only support S3 and DynamoDB. SQS does not have a Gateway Endpoint option. Attempting to create a Gateway Endpoint for SQS would fail.

### C. Two Interface Endpoints — ❌ Incorrect

While an Interface Endpoint for S3 exists (it's a newer addition), Gateway Endpoints for S3 are **free** and should be preferred unless you need features specific to Interface Endpoints (e.g., access from on-premises via Direct Connect/VPN, or cross-Region access). Using an Interface Endpoint for S3 when a Gateway Endpoint suffices wastes money ($0.01/hr + $0.01/GB).

### D. NAT Gateway — ❌ Incorrect

NAT Gateway traffic to AWS services does traverse the public internet path (the traffic goes through the NAT Gateway to the service's public endpoints). While the physical path may stay within AWS infrastructure, it uses public IP addressing and is subject to NAT Gateway data processing charges ($0.045/GB). VPC endpoints keep traffic entirely on the private AWS network and are more cost-effective for high-volume service access.

## Recommendations

- **Always** create Gateway Endpoints for S3 and DynamoDB — they're free and reduce NAT Gateway costs.
- Use **Interface Endpoints** when you need: private DNS resolution, access from on-premises via DX/VPN, or the service doesn't support Gateway Endpoints.
- **Interface Endpoint pricing**: ~$0.01/hr per AZ + $0.01/GB processed. Create endpoints only in AZs where you have workloads.
- Enable **private DNS** on Interface Endpoints so applications use standard service URLs without code changes.
- S3 supports **both** endpoint types. Use Gateway (free, route-table-based) for EC2 access within VPC; use Interface for DX/VPN or cross-VPC access via PrivateLink.

## Relevant Links

- [Gateway VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
- [Interface VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html)
- [VPC Endpoint Types](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
- [VPC Endpoints Pricing](https://aws.amazon.com/privatelink/pricing/)
