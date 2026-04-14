# Q02: Cross-Region Backup and Restore for RDS

## Question

A company runs a mission-critical Amazon RDS for PostgreSQL database (Multi-AZ, 1 TB) in eu-west-1. Compliance requires that database backups must be stored in a **geographically separate Region** (eu-central-1) and the company must be able to restore the database in eu-central-1 within **2 hours** of declaring a disaster. The database handles 5,000 transactions per second.

The solutions architect must also ensure that backups are **encrypted with a customer-managed KMS key** and that the encryption key is available in the DR Region.

Which approach meets these requirements?

## Options

- **A.** Enable RDS automated backups with cross-Region automated backup replication to eu-central-1. Create a multi-Region KMS key (or replicate the KMS key to eu-central-1). During failover, restore the automated backup in eu-central-1.
- **B.** Take manual RDS snapshots daily and use AWS Lambda to copy them to eu-central-1 using the `copy-db-snapshot` API with the `--kms-key-id` parameter pointing to a KMS key in eu-central-1. Restore from the snapshot during failover.
- **C.** Set up an RDS cross-Region read replica in eu-central-1 encrypted with a KMS key in that Region. During failover, promote the read replica.
- **D.** Use AWS DMS with continuous replication to an RDS instance in eu-central-1. Encrypt the target instance with a KMS key in eu-central-1.

## Answers

### A. Cross-Region automated backup replication — ✅ Correct

RDS supports **automated backup replication** to a different Region. This feature continuously replicates automated backups (including transaction logs) to the target Region, enabling point-in-time restore in the DR Region. Combined with a **multi-Region KMS key** (or a separate key in eu-central-1 specified during replication setup), backups are encrypted at rest in both Regions. Restoring a 1 TB database from an automated backup typically completes within 1–2 hours, meeting the 2-hour RTO.

### C. Cross-Region read replica — ❌ Incorrect (over-provisioned)

A cross-Region read replica would meet the requirements (promotion takes minutes, RPO is seconds), but it requires running a **full RDS instance continuously** in eu-central-1. The question asks about backup and restore with a 2-hour RTO — this is achievable with cross-Region backup replication at a fraction of the cost. Running a read replica for DR when the RTO allows 2 hours of restore time is unnecessarily expensive.

### B. Manual snapshots with Lambda — ❌ Incorrect

Daily manual snapshots give an RPO of up to 24 hours, which may be too high for a mission-critical database doing 5,000 TPS. More importantly, this requires custom Lambda logic for snapshot management, error handling, retention, and cross-Region copy. RDS's built-in cross-Region backup replication (option A) is simpler and more reliable.

### D. DMS continuous replication — ❌ Incorrect

DMS continuous replication to a running RDS instance provides excellent RPO but is **over-provisioned** for a backup/restore requirement. It requires running and managing an additional RDS instance + DMS replication instance in eu-central-1 at significant cost. DMS is better suited for heterogeneous database migrations, not same-engine backup/restore scenarios.

## Recommendations

- **RDS cross-Region automated backup replication** is a relatively newer feature — know it exists. It provides point-in-time recovery capability in the DR Region without running a standby instance.
- **KMS key considerations for cross-Region:**
  - RDS snapshots are encrypted with the source Region's KMS key
  - When copying cross-Region, you must specify a KMS key in the **destination Region**
  - Use **multi-Region KMS keys** to simplify key management, or create separate keys
- **Cost comparison for cross-Region DB DR:**
  - Cheapest: Cross-Region backup replication (storage only)
  - Moderate: DMS replication + target instance
  - Expensive: Cross-Region read replica (full instance)
- Match the solution cost to the RTO requirement — don't pay for a read replica when backup/restore is sufficient.

## Relevant Links

- [RDS Cross-Region Automated Backups](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReplicateBackups.html)
- [Copying RDS Snapshots Cross-Region](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CopySnapshot.html)
- [KMS Multi-Region Keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html)
- [RDS Encryption at Rest](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)
