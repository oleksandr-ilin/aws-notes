# AWS Lambda

## Service Overview
AWS Lambda is a serverless compute service that runs code in response to events and automatically manages the compute resources required.

## Tasks this service can solve
- Event-driven processing (S3 uploads, DynamoDB streams, Kinesis)
- Lightweight APIs (via API Gateway)
- Scheduled jobs and cron-like tasks
- Glue code for integrations and webhooks

## Alternatives
| Name | Type | Short description | Pro vs Lambda | Cons vs Lambda | Price comparison |
|---|---|---:|---|---|---|
| AWS Fargate | AWS | Serverless containers for longer-lived processes | Better for long-running tasks; more control | More operational surface and config | Generally higher for sustained workloads; Lambda cheaper for bursty short jobs |
| Amazon EC2 | AWS | Full VMs with OS control | Full control, custom runtimes | Must manage instances, patching | Higher baseline costs for always-on workloads |
| Google Cloud Functions | GCP | GCP serverless functions | Similar developer experience in GCP | Different integrations; vendor lock-in | Comparable per-invocation costs |
| OpenFaaS | OSS | Open-source serverless on Kubernetes | Avoid vendor lock-in | Requires k8s operations | Ops cost for infra; no per-invocation fees |

## Limitations
- Maximum execution time (15 minutes). 
- Limited ephemeral /tmp storage (default 512 MB, can be increased in newer runtimes). 
- Cold starts for infrequently used functions (mitigated with provisioned concurrency).
- Concurrent execution limits per account/region (soft limits adjustable).

## Price info
- Billed per-request and per-duration (GB-seconds) plus free tier. Additional costs for provisioned concurrency and associated resources (API Gateway, VPC ENIs, etc.).

## Network & Multi-region considerations
- Lambda functions are regional. For multi-region resilience, deploy functions into multiple regions and use DNS failover (Route 53) or regional API endpoints. 
- When running in a VPC, each cold start may attach ENIs which add latency and resource limits — consider using VPC-enabled Lambda with ENI warmers or use RDS proxies where applicable.
- Integrations: works natively with many AWS event sources (S3, SNS, SQS, EventBridge, DynamoDB). For cross-region events, use EventBridge or replicate events through SNS/global topics.

## Popular use cases
- Image processing on S3 uploads
- Lightweight REST APIs behind API Gateway
- Scheduled cron jobs and cleanup tasks
- Real-time stream processing for small workloads

## When Not to Use This Service
- Not a fit for long-running processes (>15 minutes), heavy CPU/GPU workloads, or tasks needing OS/kernel access. Use Amazon EC2, ECS/Fargate, or specialized GPU instances instead.

## DR strategy
- Deploy functions in multiple regions and replicate event sources (S3 cross-region replication, multi-region EventBridge). Persist critical state in durable stores (DynamoDB global tables, S3). Use versioning and aliases with automated health checks and Route 53 failover for API endpoints.
