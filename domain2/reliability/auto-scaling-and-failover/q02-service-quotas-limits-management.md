# Q02: Service Quotas and Limits Management at Scale

## Question

A company runs a large-scale IoT platform processing 500,000 device messages per second. During a product launch, the platform experienced cascading failures:
1. Lambda concurrent executions hit the default 1,000 limit → messages queued in SQS → SQS messages exceeded retention
2. EC2 Auto Scaling couldn't launch instances — the account hit the on-demand vCPU limit for m5 family (default 5× vCPUs in the Region)
3. API Gateway throttled at 10,000 requests/second (account-level soft limit) → mobile apps received 429 errors
4. The team filed manual quota increase requests via Support Center — but the requests took 3 days to process, causing extended outage

The team needs a proactive system to prevent quota-related failures. Which approach is most effective?

## Options

- **A.** Use AWS Service Quotas service to: (1) request quota increases for Lambda, EC2, and API Gateway BEFORE the launch (proactive). (2) Create CloudWatch alarms on service quota utilization metrics (e.g., alarm when Lambda concurrent executions reach 80% of the quota). (3) Use AWS Service Quotas request templates to pre-configure quota increases for new Regions. (4) Integrate Service Quotas with AWS Organizations to apply quota increase requests across all accounts via a quota request template. (5) Use Trusted Advisor to identify quotas approaching limits.
- **B.** Design the architecture to stay within default quotas: reduce Lambda concurrency by increasing batch sizes, use fewer but larger EC2 instances, and implement client-side throttling for API Gateway. If quotas are hit, scale horizontally across multiple AWS accounts.
- **C.** Create a custom quota monitoring system: Lambda functions that call `GetServiceQuota` and `GetAWSDefaultServiceQuota` APIs hourly, store results in DynamoDB, and send alerts via SNS. Use AWS Support API to automatically submit quota increase requests when utilization exceeds 70%.
- **D.** Pre-request maximum possible quotas for all services at account creation. Set Lambda reserved concurrency to the maximum quota value. Set API Gateway account-level throttle to the maximum possible. Over-provision EC2 capacity reservations for peak.

## Answers

### A. Service Quotas proactive increases + CloudWatch alarms + request templates + Trusted Advisor — ✅ Correct

This is the comprehensive, AWS-native approach to quota management:

- **(1) Proactive quota increase requests**:
  - AWS Service Quotas allows requesting increases through the console, CLI, or API — no Support ticket needed for most services. Requests are typically processed within minutes to hours (not days).
  - **Before a launch**: Calculate expected usage (500K messages/sec × average Lambda duration = required concurrent executions), then request quota increases to 2× expected peak:
    - Lambda concurrent executions: default 1,000 → request 50,000+
    - EC2 on-demand vCPUs: default varies → request based on instance type and count
    - API Gateway: default 10,000 RPS → request 50,000+
  - Process requests 1-2 weeks before launch — some quotas require AWS review.

- **(2) CloudWatch alarms on quota utilization**:
  - Service Quotas publishes utilization metrics to CloudWatch. You can create alarms like:
    - `ServiceQuota.UtilizationPercentage` for Lambda concurrent executions > 80% → SNS alert
    - `ServiceQuota.UtilizationPercentage` for EC2 on-demand vCPUs > 75% → auto-request increase
  - **Proactive alerting**: The team sees warnings BEFORE hitting limits — preventing the cascading failure pattern described in the scenario.
  - Not all services publish quota utilization metrics — for those that don't, use custom CloudWatch metrics via Lambda.

- **(3) Request templates for new Regions**:
  - Service Quotas request templates allow pre-configuring quota increase requests that are automatically submitted when you enable a new Region or create a new account. This ensures new Regions don't launch with default quotas that are too low.
  - Useful when expanding to new Regions for the IoT platform.

- **(4) Organization-level quota management**:
  - In AWS Organizations, the management account or delegated administrator can view and request quotas across all accounts. This provides centralized visibility into quota usage across the organization.
  - Quota request templates can be applied at the Organization level — when a new account is created via AWS Control Tower, it automatically gets the pre-configured quotas.

- **(5) Trusted Advisor service limit checks**:
  - Trusted Advisor's "Service Limits" category checks 50+ common service quotas and alerts when usage exceeds 80%. This provides a safety net for quotas that the team might forget to monitor via CloudWatch alarms.
  - Trusted Advisor checks are refreshed periodically — use them as a complement to real-time CloudWatch alarms, not a replacement.

### B. Architect within default quotas + multi-account — ❌ Incorrect

- **Design within default quotas**: Default quotas are conservative starting points — they're NOT engineering targets. A platform processing 500K messages/sec cannot fit within 1,000 Lambda concurrent executions. Increasing batch size reduces invocations but increases per-invocation duration and memory, potentially hitting other limits (15-minute timeout, 10 GB memory).
- **Fewer larger instances**: Using fewer, larger instances (e.g., m5.24xlarge instead of m5.xlarge) reduces instance count but increases blast radius — a single instance failure takes down more capacity. This trades reliability for quota avoidance.
- **Multi-account horizontal scaling**: Splitting the workload across multiple AWS accounts adds enormous operational complexity: cross-account networking, data partitioning, monitoring aggregation, deployment pipelines per account. Requesting quota increases is orders of magnitude simpler.
- This approach optimizes for the wrong constraint (quotas instead of architecture).

### C. Custom quota monitoring system — ❌ Incorrect

- **Custom Lambda polling**: Re-implements what Service Quotas + CloudWatch integration already provides natively. Custom code requires: error handling, API rate limiting (Service Quotas API has its own throttle), DynamoDB table management, and maintenance as new services are added.
- **AWS Support API for automatic quota requests**: The Support API (`CreateCase`) can submit quota increase requests, but: (1) it requires a Business or Enterprise Support plan ($100+/month), (2) automated requests may trigger abuse detection, (3) quota increases are not instantaneous — they may take hours even via API.
- **Hourly polling is too infrequent**: At 500K messages/sec, Lambda concurrent executions can spike from 50% to 100% in minutes. Hourly checks miss rapid spikes. CloudWatch alarms with 1-minute resolution are superior.
- The approach works but is over-engineered given that native tools exist.

### D. Pre-request maximum quotas + over-provision — ❌ Incorrect

- **Maximum possible quotas for all services**: AWS doesn't grant unlimited quotas. Each service has hard limits (cannot be increased) and soft limits (can be increased via request). Pre-requesting maximums for services you don't use triggers unnecessary AWS review and may be denied.
- **Lambda reserved concurrency at maximum**: Reserved concurrency for one function limits concurrency for ALL OTHER functions in the account. If you reserve 50,000 for the IoT function, other functions can only use the remaining allocation. This is a resource contention problem, not a reliability improvement.
- **API Gateway maximum throttle**: Setting the account-level throttle higher doesn't help if the API's route-level throttle, stage throttle, or usage plan throttle is still at defaults. Throttling is applied at the most restrictive level.
- **EC2 capacity reservations**: Capacity Reservations guarantee that instances are available when needed (useful for DR), but you pay for them whether used or not. Over-provisioning "just in case" is wasteful — targeted reservations for peak are better.
- This "set everything to max" approach wastes money and doesn't address the root cause (lack of monitoring and proactive management).

## Recommendations

- **Identify your account's top 10 most-used service quotas** and create CloudWatch alarms at 70% and 90% utilization thresholds. Common critical quotas: Lambda concurrent executions, EC2 vCPUs per instance family, EBS volume count, VPC ENIs, API Gateway RPS, ELB target group targets.
- **Request quota increases 2 weeks before any major launch** — some increases require AWS team review (especially high-value quotas like GPU instances or large EBS volume counts).
- **Know the difference between soft limits (adjustable) and hard limits (not adjustable)**:
  - Soft: Lambda concurrent executions, EC2 on-demand vCPUs, S3 bucket count
  - Hard: Lambda 15-minute timeout, S3 5 TB max object size, DynamoDB 400 KB max item size, CloudFormation 500 resources per stack
- **Service Quotas request templates** should be standard in your AWS landing zone configuration — every new account gets production-appropriate quotas from day one.
- **Trusted Advisor + Service Quotas + CloudWatch = complete quota monitoring**: Trusted Advisor for periodic broad checks, Service Quotas for centralized management, CloudWatch for real-time alerting.

## Relevant Links

- [AWS Service Quotas](https://docs.aws.amazon.com/servicequotas/latest/userguide/intro.html)
- [Service Quotas CloudWatch Metrics](https://docs.aws.amazon.com/servicequotas/latest/userguide/configure-cloudwatch.html)
- [Service Quotas Request Templates](https://docs.aws.amazon.com/servicequotas/latest/userguide/organization-templates.html)
- [Trusted Advisor Service Limits](https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor-check-reference.html)
- [Lambda Concurrency Quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
