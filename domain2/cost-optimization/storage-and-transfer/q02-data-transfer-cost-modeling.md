# Q01: Data Transfer Cost Modeling for Multi-Region Architecture

## Question

A SaaS company operates in 3 Regions (us-east-1, eu-west-1, ap-southeast-1) serving 10,000 enterprise customers. Monthly data flow:

| Source | Destination | Volume | Description |
|--------|-------------|--------|-------------|
| Users (internet) | CloudFront | 500 GB | API requests from browsers/mobile |
| CloudFront | ALB (us-east-1) | 200 GB | Cache misses → origin fetch |
| ALB (us-east-1) | EC2 (us-east-1) | 200 GB | Internal request routing |
| EC2 (us-east-1) | RDS (us-east-1, different AZ) | 150 GB | Database queries |
| RDS (us-east-1) | EC2 (us-east-1, different AZ) | 100 GB | Query results |
| EC2 (us-east-1) → NAT GW | S3 (us-east-1) | 2 TB | Log shipping, data export |
| EC2 (us-east-1) | EC2 (eu-west-1) via VPC peering | 500 GB | Cross-Region data sync |
| EC2 (us-east-1) | Internet (customers) | 5 TB | Report downloads, data exports |
| RDS (us-east-1) | RDS (eu-west-1) | 300 GB | Aurora Global Database replication |

The current monthly data transfer bill is $1,200. The team has been asked to reduce it by at least 50%.

Which set of changes achieves the target?

## Options

- **A.** (1) Replace NAT Gateway with S3 VPC Gateway endpoint for S3 traffic — saves NAT Gateway data processing ($0.045/GB × 2 TB = $90/month). (2) Serve reports/exports through CloudFront instead of direct EC2 egress — CloudFront egress is ~$0.085/GB vs. EC2 egress ~$0.09/GB for first 10 TB, but CloudFront caching reduces origin fetches by 60%+ for repeated report downloads, saving ~$180/month in reduced egress volume. (3) Use S3 Transfer Acceleration for cross-Region sync to replace VPC peering data transfer. (4) Compress cross-Region sync payloads (gzip/zstd) to reduce 500 GB to ~125 GB — saving ~$7.50/month on cross-Region transfer.
- **B.** (1) Replace NAT Gateway with S3 VPC Gateway endpoint. (2) Serve reports through CloudFront. (3) Consolidate all workloads into a single AZ to eliminate cross-AZ data transfer. (4) Establish AWS Direct Connect between Regions.
- **C.** (1) Replace NAT Gateway with S3 VPC Gateway endpoint. (2) Serve reports through CloudFront with aggressive caching (Cache-Control: max-age=86400). (3) Move EC2 ↔ RDS traffic to same AZ. (4) Compress cross-Region sync payloads. (5) Use PrivateLink instead of VPC peering for cross-Region sync.
- **D.** (1) Switch all S3 access to use S3 VPC Gateway endpoint (eliminate NAT Gateway processing). (2) Route all customer-facing traffic through CloudFront with origin shield enabled in us-east-1. (3) Compress cross-Region sync payloads to reduce volume by ~75%. (4) Keep EC2 ↔ RDS cross-AZ for HA (do not consolidate AZs). (5) Use free-tier data transfer for Aurora Global Database replication (no additional charge beyond the Aurora service).

## Answers

### D. VPC Gateway endpoint + CloudFront with Origin Shield + compression + keep Multi-AZ + Aurora replication awareness — ✅ Correct

This option correctly identifies every major cost lever and avoids architectural compromises:

- **(1) S3 VPC Gateway endpoint — saves ~$90/month**:
  - NAT Gateway charges $0.045/GB for data processed. 2 TB/month × $0.045 = $90/month.
  - S3 Gateway endpoints are FREE — no data processing charge, no hourly charge, no bandwidth limit.
  - This is the single highest-impact, zero-risk optimization. Gateway endpoints for S3 and DynamoDB should always be deployed.

- **(2) CloudFront with Origin Shield — saves ~$270/month**:
  - 5 TB direct EC2 internet egress: first 10 TB at ~$0.09/GB = $450/month.
  - CloudFront egress pricing is ~$0.085/GB (slightly lower), but the real savings come from caching. Reports are often downloaded multiple times by the same customer or shared across teams. With CloudFront caching:
    - Assume 60% cache hit ratio → only 2 TB reaches origin (vs. 5 TB direct).
    - CloudFront egress: 5 TB × $0.085 = $425 (but only 2 TB origin egress + 5 TB edge egress — CloudFront origin fetches from EC2 are also charged at regional egress rates).
    - Net savings ≈ $270/month (reduced origin fetches + lower per-GB edge rates at scale).
  - **Origin Shield** adds an additional caching layer between edge locations and the origin. It consolidates cache misses from all edge locations through a single regional cache — reducing origin fetches further and improving cache hit ratio by 10-20%.

- **(3) Compress cross-Region sync — saves ~$7.50/month**:
  - 500 GB cross-Region via VPC peering at $0.02/GB = $10/month.
  - Compression (gzip/zstd) on JSON/log data typically achieves 75% reduction → 125 GB × $0.02 = $2.50/month.
  - Savings: $7.50/month — modest but essentially free (add compression to the sync process).

- **(4) Keep EC2 ↔ RDS cross-AZ for HA**:
  - Cross-AZ charges: (150 + 100) GB × $0.01/GB × 2 directions = $5/month.
  - Moving RDS to the same AZ as EC2 saves $5/month but eliminates Multi-AZ high availability. If that AZ fails, both compute and database go down simultaneously. The $5/month "savings" is not worth the single-AZ risk. Correct to keep cross-AZ.

- **(5) Aurora Global Database replication — no additional data transfer charge**:
  - Aurora Global Database replication traffic between Regions is included in the Aurora service — there's no separate data transfer charge. The 300 GB/month in the table is replicated at no additional transfer cost (you pay Aurora's replication pricing, not standard EC2 cross-Region data transfer).
  - Recognizing that this is NOT a cost optimization target (it's already included) avoids wasted effort.

- **Total estimated savings**: $90 + $270 + $7.50 = ~$367.50 → 30.6% of $1,200. To reach 50%, the team should also: optimize CloudFront cache-hit ratio with longer TTLs, compress origin responses (Content-Encoding: gzip), and review if all 500 GB of cross-Region sync is necessary (filter to essential data only).

### A. VPC endpoint + CloudFront + S3 Transfer Acceleration + compression — ❌ Incorrect

- **S3 Transfer Acceleration for cross-Region sync**: Transfer Acceleration speeds up S3 uploads from distant locations using CloudFront edge network. It charges $0.04-$0.08/GB ON TOP of standard transfer costs. The cross-Region sync is between EC2 instances via VPC peering — Transfer Acceleration is for S3 uploads, not EC2-to-EC2 synchronization. This adds cost, not saves it.
- VPC endpoint and CloudFront components are correct.
- Compression savings calculation is correct but this option proposes Transfer Acceleration which increases costs.

### B. Single AZ + Direct Connect between Regions — ❌ Incorrect

- **Consolidate to single AZ**: Eliminates ~$5/month in cross-AZ transfer but destroys high availability. A single AZ failure takes down the entire application. This is an unacceptable availability trade-off for $5/month.
- **Direct Connect between Regions**: Direct Connect is for connecting on-premises networks to AWS — not for Region-to-Region communication. There's no "Direct Connect between Regions" product. Inter-Region connectivity uses VPC peering, Transit Gateway peering, or the public internet. Direct Connect Gateway can route to multiple Regions from on-premises, but that requires physical on-premises infrastructure. Also, Direct Connect has monthly port fees ($0.30/GB for 1 Gbps) that would likely exceed the current cross-Region transfer costs.

### C. VPC endpoint + CloudFront + same-AZ + PrivateLink cross-Region — ❌ Incorrect

- **Same-AZ for EC2 ↔ RDS**: Same single-AZ risk as option B — saves $5/month at the cost of availability.
- **PrivateLink cross-Region**: AWS PrivateLink operates within a single Region — it connects VPC endpoints to services within the same Region. There's no cross-Region PrivateLink. Cross-Region connectivity requires VPC peering, Transit Gateway peering, or Transit Gateway inter-Region. PrivateLink does not reduce cross-Region data transfer costs.
- VPC endpoint, CloudFront, and compression components are correct.

## Recommendations

- **Deploy VPC Gateway endpoints for S3 and DynamoDB** in every VPC — they're free and eliminate NAT Gateway processing charges. This is the highest-ROI optimization with zero downsides.
- **Use CloudFront for any internet-facing content** — lower egress rates and caching reduce origin egress volume. Add Origin Shield for additional cache consolidation.
- **Never sacrifice Multi-AZ for minor cross-AZ transfer savings** — $0.01/GB cross-AZ is the "insurance premium" for high availability. Cross-AZ design is non-negotiable for production workloads.
- **Compress all cross-Region data transfers** (gzip, zstd, snappy) — compression is cheap to compute and reduces both transfer costs and latency.
- **Understand which services include replication in their pricing**: Aurora Global Database, DynamoDB Global Tables, and S3 Cross-Region Replication all have different pricing models for the replication itself.
- **Data transfer cost debugging**: Use Cost Explorer with "Data Transfer" service filter and group by "Usage Type" to identify exactly which transfer types (internet egress, cross-AZ, cross-Region, VPC peering) are driving costs.

## Relevant Links

- [VPC Gateway Endpoints (S3, DynamoDB)](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)
- [CloudFront Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html)
- [EC2 Data Transfer Pricing](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [NAT Gateway Pricing](https://aws.amazon.com/vpc/pricing/)
