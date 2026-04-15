# Q05: Selecting AWS Regions and AZs Based on Latency Requirements

## Question

A global financial trading platform is expanding from us-east-1 to serve users in Europe and Asia-Pacific. The requirements are:
- European users must experience < 10 ms round-trip latency to the application tier
- The application processes market data feeds that originate from London and Tokyo exchanges
- A relational database must be available in all serving Regions with < 1 second replication lag
- If a Region becomes unavailable, users must be redirected to the next closest Region within 30 seconds
- The company must comply with EU data residency requirements for European customer PII

Which architecture should the solutions architect implement?

## Options

- **A.** Deploy the application in eu-west-2 (London), ap-northeast-1 (Tokyo), and keep us-east-1. Use Aurora Global Database with the primary in us-east-1 and read replicas in both Regions. Use Route 53 latency-based routing with health checks (TTL 10s). Place market data feed ingestion in the same Region as each exchange. Store EU PII exclusively in the eu-west-2 Aurora instance using application-level data partitioning.
- **B.** Deploy in all 3 Regions. Use DynamoDB Global Tables for sub-second replication. Use CloudFront with origin failover for automatic Region failover. Store EU PII in a separate DynamoDB table with a table-level resource policy restricting replication to EU only.
- **C.** Deploy only in us-east-1 and use AWS Global Accelerator with edge locations for latency reduction. Use RDS Multi-AZ for high availability. Process all market data feeds centrally. Rely on Global Accelerator's anycast routing for failover.
- **D.** Deploy in eu-west-2 and ap-northeast-1 only. Connect both Regions via Transit Gateway inter-Region peering. Use RDS with cross-Region read replicas. Use Route 53 weighted routing (50/50) for traffic distribution and manual DNS failover.

## Answers

### A. Three Regions with Aurora Global Database + latency-based routing — ✅ Correct

This architecture correctly addresses all requirements:
- **Region selection aligned to latency source**: eu-west-2 (London) is closest to London exchange feeds and European users — sub-10 ms within the region. ap-northeast-1 (Tokyo) serves APAC users and Tokyo exchange feeds locally.
- **Aurora Global Database**: Provides < 1 second replication from primary to secondary Regions (typical lag is ~200-500 ms). Supports planned and unplanned failover to promote a secondary cluster. Database is readable in all Regions immediately.
- **Route 53 latency-based routing with health checks**: Routes users to the Region with lowest latency. Health check failure triggers automatic rerouting within TTL (10s) + health check interval (~10-30s) — meeting the 30-second failover budget.
- **EU data residency**: Application-level partitioning ensures EU PII is written only to the eu-west-2 database. Aurora Global Database replicates all data, so additional app-level controls and encryption ensure compliance.
- **Co-located feed ingestion**: Processing London feed in eu-west-2 and Tokyo feed in ap-northeast-1 minimises ingestion latency.

### B. DynamoDB Global Tables + CloudFront — ❌ Partially incorrect

DynamoDB Global Tables do replicate sub-second and are multi-Region, but a relational database was specified — DynamoDB is not relational. CloudFront provides caching and content delivery but its origin failover is designed for static/dynamic content caching, not full application-tier failover control equivalent to DNS-based routing. DynamoDB table-level resource policies do not control replication geography — Global Tables replicate to all configured Regions.

### C. Single Region + Global Accelerator — ❌ Incorrect

Global Accelerator reduces network path latency by using AWS backbone, but it cannot eliminate the physical distance latency between London/Tokyo and us-east-1. Transatlantic round-trip is ~70-80 ms minimum — far above the 10 ms requirement for European users. RDS Multi-AZ is single-Region HA only, not cross-Region DR. Central feed processing adds unnecessary latency for London and Tokyo exchange data.

### D. Two Regions + weighted routing — ❌ Incorrect

Eliminates us-east-1, losing North American users. Weighted 50/50 routing ignores latency — a Tokyo user could be sent to London. Manual DNS failover cannot meet the 30-second redirect requirement. Transit Gateway peering connects VPC networks but does not solve database replication (RDS cross-Region read replicas have higher lag than Aurora Global Database and require manual promotion).

## Recommendations

- **Region selection** should be driven by: (1) proximity to end users, (2) proximity to data sources, (3) available services, and (4) regulatory requirements.
- **AZ selection within a Region**: Always use at least 2 AZs (preferably 3) for the application tier. AZs within a Region have < 2 ms latency between them.
- **Aurora Global Database** is preferred over RDS read replicas for multi-Region: it offers faster replication, managed failover, and write-forwarding.
- **Route 53 latency-based routing** uses a database of AWS Region IP latencies, not real user measurements — combine with health checks for reliability.
- For data residency: use application-level controls and consider AWS Regions in the required geography. Monitor with Config rules and SCPs to prevent resource creation outside approved Regions.

## Relevant Links

- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Route 53 Latency-Based Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html)
- [AWS Global Infrastructure - Regions](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/)
- [Global Accelerator vs CloudFront](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html)
