# Amazon Timestream

## Service Overview
Amazon Timestream is a serverless time-series database optimized for storing, processing, and querying time-series data.

## Tasks this service can solve
- IoT telemetry storage and analytics
- Time-series metrics and monitoring data

## Alternatives
| Name | Type | Short description | Pro vs Timestream | Cons vs Timestream | Price comparison |
|---|---|---:|---|---|---|
| InfluxDB | OSS/Commercial | Time-series DB with rich query language | Mature time-series features | Requires ops or commercial license | Varies; Timestream is managed serverless |
| DynamoDB + Timestamps | AWS | Use key-value store with time-series patterns | Familiar serverless model | Less optimized for time-series queries | Cost varies by access patterns |

## Limitations
- Query expressiveness and retention policies are specific to the service; not a general-purpose relational DB.

## Price info
- Charged for writes, storage, and query processing.

## Network & Multi-region considerations
- Regional service; replicate ingestion or export data to S3 for cross-region analytics and DR.

## When Not to Use This Service
- Not suitable for general relational or wide table analytics; use Redshift or RDS for non-time-series use cases.

## DR strategy
- Export periodic snapshots or stream data to S3 and replicate across regions; maintain ingestion scripts and query definitions in source control.

## Popular use cases
- IoT telemetry pipelines, operational monitoring stores, and time-series analytics for sensors.
