# Q02: AWS PrivateLink vs. VPC Peering for Service Access

## Question

A SaaS provider operates a multi-tenant API service in their AWS account behind a Network Load Balancer. They need to provide access to this API to 50 customer AWS accounts. The requirements are:
- Customers access the API using private IPs (no internet exposure)
- The SaaS provider must not expose their VPC CIDR range to customers
- Customers' VPCs have overlapping CIDR ranges (many use 10.0.0.0/16)
- The connection must be initiated from the customer's VPC, not the provider's
- The solution must scale to hundreds of customers without networking complexity

Which connectivity approach should the solutions architect recommend?

## Options

- **A.** Create VPC Peering connections between the SaaS provider's VPC and each customer's VPC. Use security groups on the NLB to restrict access to specific customer CIDR ranges.
- **B.** Create an AWS PrivateLink VPC Endpoint Service backed by the Network Load Balancer. Have each customer create an interface VPC endpoint in their VPC that connects to the endpoint service. Approve connection requests from authorized customer accounts.
- **C.** Deploy the API in a Transit Gateway and share the Transit Gateway via RAM with customer accounts. Each customer attaches their VPC to the shared Transit Gateway.
- **D.** Set up AWS Site-to-Site VPN connections between the SaaS provider's VPC and each customer's VPC. Use VPN routing to provide private connectivity.

## Answers

### B. AWS PrivateLink (VPC Endpoint Service) — ✅ Correct

AWS PrivateLink is purpose-built for this use case:
- **Private connectivity**: Customers access the API via a private IP address (ENI) in their own VPC — no internet exposure.
- **No CIDR exposure**: The provider's VPC CIDR is never shared with customers. Customers see only the endpoint ENI in their VPC.
- **Overlapping CIDRs**: PrivateLink works regardless of VPC CIDR overlap — each customer's interface endpoint gets an IP from their own VPC CIDR.
- **Unidirectional**: Connections are initiated from the customer's VPC endpoint to the provider's NLB — the provider cannot initiate connections into customer VPCs.
- **Scalable**: The provider creates one endpoint service; each customer independently creates their own endpoint. No per-customer networking configuration needed on the provider side.
- **Approval workflow**: The provider can auto-approve or manually approve connection requests.

### A. VPC Peering — ❌ Incorrect

VPC Peering requires non-overlapping CIDR ranges, which is impossible with 50 customers using overlapping CIDRs. It also exposes the provider's entire VPC to each customer (bidirectional routing). Managing 50+ peering connections with security groups is complex and doesn't scale to hundreds of customers.

### C. Transit Gateway sharing — ❌ Incorrect

A shared Transit Gateway would expose routing between customer VPCs (unless complex route table segmentation is implemented), creating potential cross-customer access. Overlapping CIDRs would cause routing conflicts. Transit Gateway is designed for network connectivity, not service endpoint access. Sharing a TGW with external customers also raises significant security concerns.

### D. Site-to-Site VPN — ❌ Incorrect

VPN connections require IP address negotiation, IPSec tunnel management, and customer gateway configuration for each of 50+ customers. VPN has bandwidth limitations (1.25 Gbps per tunnel) and introduces the same overlapping CIDR problem. The operational overhead of managing 50+ VPN tunnels is prohibitive.

## Recommendations

- **PrivateLink** is the AWS-recommended approach for SaaS providers offering private API access to customers.
- PrivateLink requires a **Network Load Balancer** or **Gateway Load Balancer** as the backend — Application Load Balancers are not directly supported.
- Enable **private DNS name** on the endpoint service so customers can use a friendly DNS name (e.g., `api.saasprovider.com`) that resolves to the private endpoint IP.
- Consider **PrivateLink for AWS Marketplace SaaS** — AWS Marketplace supports PrivateLink-based SaaS subscriptions.
- PrivateLink is also how AWS services provide private endpoints (e.g., `com.amazonaws.us-east-1.s3` interface endpoint).

## Relevant Links

- [AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html)
- [VPC Endpoint Services](https://docs.aws.amazon.com/vpc/latest/privatelink/endpoint-services-overview.html)
- [PrivateLink vs. VPC Peering](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
- [Interface VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html)
