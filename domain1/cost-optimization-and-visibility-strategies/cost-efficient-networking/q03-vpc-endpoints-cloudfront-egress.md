# Q03: VPC Endpoints and CloudFront for Egress Cost Reduction

## Question

A video streaming company serves content stored in Amazon S3 to millions of users globally. The application backend runs on EC2 instances in a private subnet and frequently reads from S3 and writes to Amazon DynamoDB. The company's monthly AWS bill shows three major cost components: $15,000 in S3 data transfer (egress to internet), $8,000 in NAT Gateway data processing charges (for EC2→S3 and EC2→DynamoDB traffic), and $3,000 in NAT Gateway hourly charges.

Which combination of changes should the solutions architect recommend to reduce costs? **(Select TWO.)**

## Options

- **A.** Configure Amazon CloudFront with the S3 bucket as the origin. Serve video content to end users through CloudFront instead of directly from S3. Use Origin Access Control (OAC) to restrict direct S3 access.
- **B.** Create S3 Gateway VPC Endpoints and DynamoDB Gateway VPC Endpoints in the VPC. Update route tables for the private subnets to route S3 and DynamoDB traffic through the gateway endpoints instead of the NAT Gateway.
- **C.** Move the EC2 instances to a public subnet with Elastic IP addresses. This eliminates the need for NAT Gateway entirely.
- **D.** Enable S3 Transfer Acceleration to speed up content delivery to global users and reduce data transfer costs.
- **E.** Replace NAT Gateway with a NAT instance (t3.micro) to reduce the per-GB data processing charge.

## Answers

### A. CloudFront for content delivery — ✅ Correct

CloudFront data transfer to the internet is significantly cheaper than S3 direct egress ($0.085/GB vs. $0.09/GB for the first 10 TB, with deeper discounts at higher tiers). More importantly, CloudFront caches content at edge locations, reducing the number of requests to the S3 origin and significantly reducing origin data transfer. For video streaming, cache hit ratios of 80-95% are common, which could reduce the $15,000 S3 egress cost by 80% or more. Additionally, data transfer from S3 to CloudFront is **free**.

### B. Gateway VPC Endpoints for S3 and DynamoDB — ✅ Correct

Gateway VPC Endpoints for S3 and DynamoDB are **free** — no hourly charge and no per-GB data processing fee. By routing EC2→S3 and EC2→DynamoDB traffic through Gateway Endpoints instead of the NAT Gateway, the company eliminates both the $8,000 in NAT Gateway data processing charges and reduces NAT Gateway usage (potentially reducing or eliminating the $3,000 hourly charges if no other traffic routes through the NAT Gateway).

### C. Public subnet with Elastic IPs — ❌ Incorrect

Moving EC2 instances to public subnets violates security best practices by directly exposing them to the internet. It would also still incur S3 and DynamoDB data transfer charges (over the public internet). Additionally, public IPv4 addresses now cost $0.005/hr each. This doesn't solve the cost problem and introduces security risks.

### D. S3 Transfer Acceleration — ❌ Incorrect

Transfer Acceleration speeds up uploads to S3 by routing through CloudFront edge locations, but it **increases** costs (additional per-GB charge on top of standard transfer pricing). It's designed for upload acceleration, not for reducing content delivery costs. CloudFront is the correct solution for serving content to end users.

### E. NAT instance to replace NAT Gateway — ❌ Incorrect

A NAT instance eliminates the per-GB processing fee but introduces operational overhead (managing the instance, patching, HA configuration). A t3.micro would also bottleneck at ~5 Gbps max network performance. The better solution is eliminating the traffic that flows through the NAT Gateway entirely using VPC Endpoints.

## Recommendations

- **Gateway VPC Endpoints** (S3, DynamoDB) are free and should be enabled in every VPC — there's no reason not to use them.
- **Interface VPC Endpoints** (for other services) have hourly and data processing charges but still avoid NAT Gateway fees and keep traffic private.
- **CloudFront** reduces egress costs through caching + cheaper edge transfer rates. S3→CloudFront transfer is free.
- Audit NAT Gateway usage with **VPC Flow Logs** to identify what traffic flows through it — much of it may be eliminable with VPC Endpoints.
- Consider using **S3 Intelligent-Tiering** alongside CloudFront to further optimize storage costs for the video content.

## Relevant Links

- [VPC Gateway Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
- [Amazon CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/)
- [NAT Gateway Pricing](https://aws.amazon.com/vpc/pricing/)
- [CloudFront Origin Access Control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
