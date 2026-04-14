# Q02: Achieving Near-Zero RPO for Multi-Database Architecture

## Question

A logistics company runs an application that uses **three different database engines**: Amazon RDS PostgreSQL for shipment tracking, Amazon DynamoDB for real-time GPS coordinates, and Amazon Redshift for analytics. The company needs to implement DR with an RPO of less than **1 minute** for all data stores. The DR Region is us-west-2 (primary is us-east-1).

Which combination achieves near-zero RPO for ALL three data stores? (Select THREE)

## Options

- **A.** Set up an RDS PostgreSQL cross-Region read replica in us-west-2.
- **B.** Take RDS automated snapshots every minute and copy to us-west-2.
- **C.** Enable DynamoDB Global Tables to replicate to us-west-2.
- **D.** Use DynamoDB Point-in-Time Recovery (PITR) with cross-Region export.
- **E.** Configure Redshift cross-Region snapshots with a 1-hour copy interval.
- **F.** Use Redshift cross-Region snapshot copy with the minimum interval, combined with Redshift Streaming Ingestion via Kinesis Data Streams with cross-Region replication.
- **G.** Migrate from Redshift to Aurora PostgreSQL with Global Database for unified near-zero RPO.

## Answers

### A. RDS cross-Region read replica — ✅ Correct

RDS cross-Region read replicas use asynchronous replication with typical lag of seconds. For PostgreSQL, replication lag is usually under 1 minute, meeting the RPO requirement. During failover, promote the replica.

### C. DynamoDB Global Tables — ✅ Correct

DynamoDB Global Tables provides multi-Region, multi-active replication with replication typically completing within **1 second**. This easily meets the sub-1-minute RPO for GPS coordinate data.

### F. Redshift with streaming ingestion — ✅ Correct

Standard Redshift cross-Region snapshots have a minimum interval of 1 hour and take time to copy, giving an RPO measured in hours — not minutes. To achieve near-zero RPO for analytics data, you need to combine snapshot copy (for base data) with **Redshift Streaming Ingestion from Kinesis Data Streams** replicated cross-Region. New data flows through Kinesis (which replicates in near-real-time) to both primary and DR Redshift clusters.

### B. RDS snapshots every minute — ❌ Incorrect

RDS automated snapshots cannot be configured at 1-minute intervals. Automated backups happen approximately every 5 minutes and are continuous, but cross-Region **copy** adds significant delay (minutes to hours depending on size). Cross-Region read replicas are the correct approach.

### D. DynamoDB PITR with export — ❌ Incorrect

PITR provides point-in-time recovery within the **same Region**. Exporting DynamoDB data to S3 and replicating it cross-Region introduces multi-minute delays. Global Tables (option C) is the purpose-built cross-Region solution.

### E. Redshift 1-hour snapshots — ❌ Incorrect

A 1-hour snapshot copy interval means the RPO is at minimum **1 hour plus copy time** — this violates the sub-1-minute RPO requirement. Redshift's minimum automatic snapshot interval is 8 hours; even manual snapshots at 1-hour intervals won't meet the requirement.

### G. Migrate from Redshift to Aurora — ❌ Incorrect

Migrating an analytics workload from Redshift (columnar) to Aurora (row-based OLTP) is architecturally inappropriate. Redshift is optimized for analytical queries; Aurora is not. This migration would degrade analytics performance significantly and is a disproportionate response.

## Recommendations

- Each AWS database service has a **different cross-Region replication mechanism**:
  - RDS: cross-Region read replicas
  - Aurora: Global Database
  - DynamoDB: Global Tables
  - Redshift: cross-Region snapshot copy (+ streaming for low RPO)
  - ElastiCache: Global Datastore
- The SAP exam often tests multi-database DR — know the replication method for each engine.
- **Redshift has the worst cross-Region RPO** of major AWS databases. For near-zero RPO, you must supplement snapshots with streaming replication.
- Don't confuse **PITR** (point-in-time recovery, same Region) with **cross-Region replication**.

## Relevant Links

- [RDS Cross-Region Read Replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html#USER_ReadRepl.XRgn)
- [DynamoDB Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)
- [Redshift Cross-Region Snapshots](https://docs.aws.amazon.com/redshift/latest/mgmt/managing-snapshots-console.html#snapshot-crossregion)
- [Redshift Streaming Ingestion](https://docs.aws.amazon.com/redshift/latest/dg/materialized-view-streaming-ingestion.html)
