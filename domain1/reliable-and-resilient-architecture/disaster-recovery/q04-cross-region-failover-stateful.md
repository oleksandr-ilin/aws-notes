# Q04: Cross-Region Failover for Stateful Application

## Question

A media streaming company runs a stateful application in **eu-west-1**. The application stores user session data in Amazon DynamoDB, video metadata in Amazon Aurora MySQL, and video files in Amazon S3. The company needs to implement DR in **eu-central-1** with an RTO of **30 minutes** and RPO of **5 minutes**. During failover, users must be automatically redirected to the DR Region without manual DNS changes.

Which combination of services and configurations should the solutions architect use? (Select THREE)

## Options

- **A.** Configure DynamoDB Global Tables for cross-Region session replication.
- **B.** Use DynamoDB Streams with AWS Lambda to manually copy session items to eu-central-1.
- **C.** Set up Aurora Global Database with a secondary cluster in eu-central-1.
- **D.** Take Aurora automated snapshots every 5 minutes and copy them to eu-central-1.
- **E.** Enable S3 Cross-Region Replication (CRR) for the video files bucket.
- **F.** Use Amazon Route 53 health checks with failover routing policy, pointing to regional endpoints.
- **G.** Use Amazon CloudFront with origin failover group to handle Region failures automatically.

## Answers

### A. DynamoDB Global Tables — ✅ Correct

DynamoDB Global Tables provides fully managed, multi-Region, multi-active replication with sub-second replication lag. This meets the 5-minute RPO for session data and requires no custom code or Lambda functions.

### C. Aurora Global Database — ✅ Correct

Aurora Global Database replicates data from the primary cluster to secondary clusters in other Regions with typical replication lag under 1 second. During failover, the secondary cluster can be promoted to a standalone writer in under 1 minute, meeting both the 5-minute RPO and 30-minute RTO.

### F. Route 53 failover routing — ✅ Correct

Route 53 failover routing with health checks automatically redirects traffic to the DR Region when the primary Region's health check fails. This provides automatic DNS failover without manual intervention, as required by the question.

### B. DynamoDB Streams with Lambda — ❌ Incorrect

While this could work, it's a custom-built, self-managed replication solution. DynamoDB Global Tables (option A) provides this functionality natively with better reliability and no operational overhead. Building custom replication introduces failure points and maintenance burden.

### D. Aurora snapshots every 5 minutes — ❌ Incorrect

Aurora automated backups run approximately every 5 minutes, but **cross-Region snapshot copy** is a manual/scheduled process that introduces additional delay. The snapshot restore process also takes significant time (potentially exceeding the 30-minute RTO for large databases). Aurora Global Database is the correct approach for low RPO/RTO.

### E. S3 Cross-Region Replication — ❌ Incorrect (not selected)

While S3 CRR is a valid service, the question asks to select THREE answers, and S3 CRR alone doesn't address session data, database, or DNS failover. The three critical components are: session replication (A), database replication (C), and automatic failover routing (F). S3 CRR could be added as a fourth consideration but isn't one of the three most critical choices.

### G. CloudFront origin failover — ❌ Incorrect

CloudFront origin failover works at the **distribution level** for specific origin errors (5xx, 4xx) but doesn't provide Region-level failover for the entire application architecture. It doesn't handle database failover or session migration. Route 53 failover routing is the correct approach for Region-level DR.

## Recommendations

- For **stateful cross-Region applications**, the typical pattern is: DynamoDB Global Tables (session state) + Aurora Global Database (relational data) + Route 53 failover (DNS).
- **DynamoDB Global Tables** vs. Streams+Lambda: always prefer the managed service over custom replication unless you have very specific transformation requirements.
- **Aurora Global Database** promotion takes ~1 minute — much faster than restoring from snapshots.
- **Route 53 health checks** should monitor the **application endpoint**, not just the infrastructure, to detect application-level failures.
- When a question says "automatically redirected," this is a strong hint for Route 53 failover or latency-based routing.

## Relevant Links

- [DynamoDB Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Route 53 Failover Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
- [S3 Cross-Region Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
