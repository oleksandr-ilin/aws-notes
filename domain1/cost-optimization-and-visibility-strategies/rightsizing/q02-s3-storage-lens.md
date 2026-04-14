# Q02: S3 Storage Lens for Cross-Account Storage Optimization

## Question

A healthcare organization stores petabytes of data across 25 AWS accounts, including medical images in Amazon S3, archived records, and compliance data. The storage costs have grown 40% year-over-year. The cloud team needs organization-wide visibility into S3 storage patterns to identify optimization opportunities such as: objects that should be transitioned to cheaper storage classes, incomplete multipart uploads consuming storage, buckets without lifecycle policies, and accounts with disproportionate storage growth.

Which approach should the solutions architect recommend?

## Options

- **A.** Enable Amazon S3 Storage Lens with an organization-level dashboard and advanced metrics. Use the dashboard to identify optimization opportunities across all accounts, including storage class distribution, lifecycle rule coverage, and incomplete multipart uploads.
- **B.** Enable S3 server access logging on all buckets across all accounts. Aggregate logs in a central S3 bucket, and use Amazon Athena to analyze access patterns and storage utilization.
- **C.** Use Amazon S3 Inventory reports for each bucket across all accounts. Aggregate the inventory data in a central S3 bucket and build custom reports using Amazon QuickSight.
- **D.** Use AWS Trusted Advisor to identify S3 buckets with cost optimization opportunities. Review recommendations account-by-account and implement changes.

## Answers

### A. S3 Storage Lens with advanced metrics — ✅ Correct

Amazon S3 Storage Lens provides organization-wide visibility into S3 usage across all accounts with:
- **Organization-level dashboard** covering all 25 accounts from a single view.
- **Advanced metrics** showing storage class distribution, incomplete multipart upload bytes, lifecycle rule coverage, and replication status.
- **Contextual recommendations** based on storage patterns (e.g., "enable lifecycle rules for X buckets").
- **Trend analysis** showing growth rates per account, bucket, and prefix over time.
- Pre-built **metric groupings** for cost optimization, data protection, and access management.
This directly addresses all the requirements with minimal setup.

### B. S3 access logging + Athena — ❌ Incorrect

S3 server access logs capture API requests but do not provide storage class distribution, lifecycle rule coverage, or incomplete multipart upload analysis. Building analytical queries for storage optimization from raw access logs would require significant custom development and wouldn't give the high-level dashboard view needed for organizational visibility.

### C. S3 Inventory + QuickSight — ❌ Incorrect

S3 Inventory provides per-bucket object listings (including storage class and size), which could be aggregated for analysis. However, setting up inventory for every bucket across 25 accounts, aggregating the data, and building custom QuickSight dashboards is substantial effort. Storage Lens provides all of this out of the box as a managed feature.

### D. Trusted Advisor — ❌ Incorrect

Trusted Advisor has limited S3 cost checks (e.g., buckets with logging disabled) but doesn't provide storage class analysis, lifecycle coverage, multipart upload tracking, or organization-wide storage trends. It's not designed for the deep S3 storage optimization analysis described here.

## Recommendations

- **S3 Storage Lens free metrics** include 28+ usage metrics with 14-day retention. **Advanced metrics** add 35+ additional metrics with 15-month retention and contextual recommendations.
- Enable **advanced metrics and recommendations** for the best optimization insights — the cost is minimal relative to petabytes of storage.
- Use Storage Lens to identify buckets lacking **S3 Intelligent-Tiering** or lifecycle policies — transitioning infrequently accessed data can yield significant savings.
- Monitor **incomplete multipart uploads** — they consume storage silently. Enable **AbortIncompleteMultipartUpload** lifecycle rules.
- Export Storage Lens metrics to S3 for integration with custom analytics pipelines.

## Relevant Links

- [Amazon S3 Storage Lens](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens.html)
- [S3 Storage Lens Metrics](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens_metrics_glossary.html)
- [S3 Intelligent-Tiering](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intelligent-tiering.html)
- [S3 Lifecycle Management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
