# Q02: Service Endpoints for Private Service Integration

## Question

A healthcare company runs an application in a private subnet with no internet access. The application needs to:
- Upload patient documents to Amazon S3
- Write audit records to DynamoDB
- Invoke Lambda functions for document processing
- Pull container images from Amazon ECR
- Send notifications via Amazon SNS

The security team requires that none of this traffic traverses the public internet. The networking team wants to minimize endpoint costs where possible.

Which VPC endpoint configuration should the solutions architect implement?

## Options

- **A.** Create a Gateway VPC Endpoint for S3 and a Gateway VPC Endpoint for DynamoDB (both free). Create Interface VPC Endpoints (PrivateLink) for Lambda, ECR (both `ecr.api` and `ecr.dkr` plus S3 gateway for image layers), and SNS. Associate all endpoints with the private subnet's route table or security group as appropriate.
- **B.** Create Interface VPC Endpoints for all five services (S3, DynamoDB, Lambda, ECR, SNS). Apply security groups to all endpoints restricting access to the application's security group.
- **C.** Add a NAT Gateway in a public subnet and route all traffic from the private subnet through it. The NAT Gateway provides internet access, and all services are accessed via their public endpoints.
- **D.** Create a Transit Gateway with a VPN attachment to the internet. Route all AWS service traffic through the Transit Gateway. Use route table entries to direct traffic to each service's public IP ranges.

## Answers

### A. Gateway endpoints (S3, DynamoDB) + Interface endpoints (Lambda, ECR, SNS) — ✅ Correct

This is the optimal configuration:
- **Gateway Endpoint for S3**: Free, route-table-based. A prefix list entry is added to the subnet route table directing S3 traffic to the gateway endpoint. No per-hour or per-GB charges. Supports S3 bucket policies with `aws:sourceVpce` condition for endpoint-restricted access.
- **Gateway Endpoint for DynamoDB**: Also free and route-table-based. Same mechanism as S3 gateway endpoint.
- **Interface Endpoint for Lambda** (`lambda`): Creates an ENI in the subnet with a private IP. Lambda `Invoke` API calls resolve to the ENI's private IP via PrivateLink DNS. Charged per hour + per GB.
- **Interface Endpoints for ECR**: Requires **two endpoints**: `ecr.api` (for ECR API calls like authentication and manifest retrieval) and `ecr.dkr` (for Docker registry API / image pull). Image **layer data** is stored in S3, so the S3 Gateway Endpoint is also needed for complete ECR functionality.
- **Interface Endpoint for SNS**: Creates an ENI for SNS API calls. Publish/subscribe actions stay private.
- **Cost optimization**: Using free Gateway endpoints for S3 and DynamoDB saves the per-hour + per-GB costs that Interface endpoints would incur for those high-traffic services.

### B. Interface endpoints for all services — ❌ Functional but suboptimal

- Interface endpoints for S3 and DynamoDB **work** but cost money unnecessarily — Gateway endpoints provide the same private connectivity for free.
- Interface endpoints charge ~$0.01/hour per AZ + $0.01/GB processed. For high-volume S3 and DynamoDB traffic, this adds significant cost.
- This option satisfies the security requirement but violates the cost minimization requirement.

### C. NAT Gateway — ❌ Incorrect

- NAT Gateway allows **internet access** from private subnets. Traffic to AWS services traverses the public internet (via NAT Gateway → IGW → internet → service public endpoints).
- This directly violates the requirement that no traffic traverses the public internet.
- NAT Gateway costs: $0.045/hour + $0.045/GB processed — expensive for high-volume S3/DynamoDB traffic.
- The security team's no-internet requirement eliminates this option.

### D. Transit Gateway + VPN — ❌ Incorrect

- A VPN attachment to the internet introduces internet connectivity, violating the no-public-internet requirement.
- Transit Gateway is designed for VPC-to-VPC and VPC-to-on-premises connectivity, not for accessing AWS services.
- Routing to public IP ranges through a Transit Gateway still means traffic reaches the internet — just through a different path.
- This adds unnecessary cost and complexity compared to VPC endpoints.

## Recommendations

- **Always use Gateway Endpoints** for S3 and DynamoDB when available — they're free and route-table-based.
- **Interface Endpoints** are needed for all other AWS services. Common ones: Lambda, ECR, ECS, SNS, SQS, STS, KMS, CloudWatch Logs, Secrets Manager.
- **ECR requires 3 endpoints**: `ecr.api`, `ecr.dkr`, and the S3 Gateway endpoint. Missing any one causes image pull failures.
- **Enable Private DNS** on Interface endpoints (default) — this overrides the public DNS for the service, so existing application code works without changes.
- **Endpoint policies**: Both Gateway and Interface endpoints support IAM-like policies to restrict which actions/resources can be accessed through the endpoint — use these for defense in depth.
- **Multi-AZ**: Interface endpoints create one ENI per specified subnet/AZ. For HA, specify subnets in multiple AZs.

## Relevant Links

- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [Gateway Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
- [Interface Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html)
- [ECR VPC Endpoints](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html)
- [Endpoint Policies](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-access.html)
