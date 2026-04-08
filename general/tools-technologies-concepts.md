# Key Tools, Technologies and Concepts

## Areas

- Compute
- Cost management
- Database
- Disaster recovery
- High availability
- Management and governance
- Microservices and component decoupling
- Migration and data transfer
- Networking, connectivity, and content delivery
- Security
- Serverless design principles
- Storage

## In-scope AWS services and features

### Analytics

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon Athena](analytics/amazon-athena.md) | Regional | Redshift Spectrum (pro: integrated, con: cost) | Serverless interactive SQL for data in S3 | Ad-hoc queries over S3 data | Low-latency OLTP lookups | Cost by data scanned, concurrency limits | $ per TB scanned; query costs apply
| [AWS Data Exchange](analytics/aws-data-exchange.md) | Regional / Global | Direct data sharing (pro: free, con: manual) | Marketplace for third-party datasets | Buying curated datasets | Private data sharing | Licensing & provider limits | Varies by dataset/subscription
| [AWS Data Pipeline](analytics/aws-data-pipeline.md) | Regional | AWS Glue (pro: serverless, con: learning) | Legacy ETL/orchestration for data workflows | Scheduled ETL with custom instances | Modern serverless ETL | Aging service, less integration | Instance + task runtimes
| [Amazon EMR](analytics/amazon-emr.md) | Regional | AWS Glue / Spark on EKS (pro: serverless) | Managed Hadoop/Spark clusters for big data | Large-scale batch processing | Small ad-hoc queries | Cluster startup time, tuning needed | EC2 + EMR per-instance pricing
| [AWS Glue](analytics/aws-glue.md) | Regional | EMR (pro: control, con: ops) | Serverless ETL, catalog and job authoring | ETL + metadata cataloging | Low-level cluster tuning | Cold start, job-time costs | DPUs per-second + catalog storage
| [Amazon Kinesis Data Analytics](analytics/amazon-kinesis-data-analytics.md) | Regional | MSK + Kafka Streams (pro: flexibility) | Real-time SQL/Apache Flink analytics on streams | Stream transforms & aggregations | Complex stateful streaming | State size and scaling limits | App-hour + resource charges
| [Amazon Kinesis Data Firehose](analytics/amazon-kinesis-data-firehose.md) | Regional | Kafka Connect (pro: control) | Ingest and deliver streaming data to targets | Simple reliable delivery to S3/Redshift/ES | Complex stream processing | Limited transform capabilities | Per-GB ingested + optional Lambda transform
| [Amazon Kinesis Data Streams](analytics/amazon-kinesis-data-streams.md) | Regional | MSK (pro: full kafka features) | Sharded streaming log for real-time apps | High-throughput stream ingestion | Simple queueing patterns | Shard management, retention limits | Shard-hour + PUT payload pricing
| [AWS Lake Formation](analytics/aws-lake-formation.md) | Regional | Glue + manual catalog (pro: control) | Simplifies building secure, governed data lakes | Governed access & ingest flows | Tiny datasets or simple needs | Complexity for small teams | Minimal service fees, uses underlying services
| [Amazon MSK](analytics/amazon-msk.md) | Regional | Self-managed Kafka / Kinesis (pro: managed) | Managed Apache Kafka clusters | Kafka-native streaming apps | Small simple streams | Broker ops, scaling considerations | Broker instance + storage billing
| [Amazon OpenSearch Service](analytics/amazon-opensearch-service.md) | Regional | OpenSearch on EC2 (pro: control) | Managed search and analytics (formerly ES) | Log search, analytics dashboards | Very cheap storage needs | Version lag and cluster limits | Instance-hour + storage + I/O
| [Amazon QuickSight](analytics/amazon-quicksight.md) | Regional | Tableau / Looker (pro: features) | SaaS BI and dashboarding service | Lightweight BI & dashboards | Enterprise feature-rich BI | Feature limitations in standard tiers | Per-user + session/usage pricing

### Application Integration

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon AppFlow](application-integration/amazon-appflow.md) | Regional | Custom ETL (pro: flexible) | Low-code SaaS-to-SaaS data flows | SaaS integrations (Salesforce, etc.) | Complex ETL needs | Connector limits | Per-flow + data processed
| [AWS AppSync](application-integration/aws-appsync.md) | Regional | API Gateway + custom GraphQL server | Managed GraphQL API with real-time/data sync | Mobile apps & real-time sync | Extremely custom GraphQL servers | Resolver limits & VTL complexity | Per-request + data transfer
| [Amazon EventBridge](application-integration/amazon-eventbridge.md) | Regional (supports global event bus) | SNS + SQS (pro: simpler) | Event routing and bus with filtering | Event-driven architectures | Very simple pub/sub only needs | Limits per account/throughput | Per-event published/ingested pricing
| [Amazon MQ](application-integration/amazon-mq.md) | Regional | Amazon SQS/SNS (pro: managed queues) | Managed ActiveMQ/RabbitMQ brokers | Lift-and-shift message brokers | Serverless message patterns | Broker management & scaling | Broker-hour + storage
| [Amazon SNS](application-integration/amazon-sns.md) | Regional | EventBridge (pro: routing) | Pub/sub messaging for notifications | Fan-out notifications | Complex routing/filtering | Message size limits | Per-request + delivery type pricing
| [Amazon SQS](application-integration/amazon-sqs.md) | Regional | Kafka / Kinesis (pro: ordering) | Durable queueing with at-least-once semantics | Decoupling microservices | True streaming semantics | Visibility timeout, FIFO limits | Per-request pricing
| [AWS Step Functions](application-integration/aws-step-functions.md) | Regional | Custom orchestrator (e.g., SWF) | Serverless workflows with state machines | Orchestrating serverless/microservices | Very low-latency step needs | State size and execution duration | Per-state transition + duration pricing

### Business Applications

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Alexa for Business](business-applications/alexa-for-business.md) | Regional | Third-party voice platforms | Voice-enabled meeting room and device management | Voice interactions in meeting rooms | Non-voice UIs | Limited device ecosystem | Per-device + usage costs (varies)
| [Amazon SES](business-applications/amazon-ses.md) | Regional | SendGrid / Mailgun (pro: features) | Email sending, receiving and reputation management | Transactional and bulk email | Complex campaign management | Deliverability tuning needed | Per-email + data transfer charges

### Blockchain

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon Managed Blockchain](blockchain/amazon-managed-blockchain.md) | Regional | Self-hosted blockchain (pro: control) | Managed Hyperledger Fabric & Ethereum nodes | Consortium blockchain use-cases | Simple databases | Network throughput and cost | Node-hour + transaction fees (varies)

### Cloud Financial Management

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS Budgets](cloud-financial-management/aws-budgets.md) | Regional | Custom tooling | Alerts and budgets for account spend | Enforcing cost guardrails | Granular forecasting | Manual setup for many accounts | Free tier + charges for notifications
| [AWS Cost and Usage Report](cloud-financial-management/aws-cost-and-usage-report.md) | Regional | Third-party billing tools | Detailed usage & billing data export | Detailed chargeback & analytics | Lightweight needs | Large file sizes & processing | Free to generate; storage/processing costs
| [AWS Cost Explorer](cloud-financial-management/aws-cost-explorer.md) | Regional | Third-party analytics | UI for cost trends, forecasting and rightsizing | Cost analysis & recommendations | Deep custom BI | Data granularity / lookback limits | Free + optional APIs
| [Savings Plans](cloud-financial-management/aws-savings-plans.md) | Global (account-level) | Reserved Instances (more rigid) | Committed usage discounts for compute | Predictable compute savings | Spiky unpredictable workloads | Commit commitment risk | Discount vs on-demand based on term

### Compute

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS App Runner](compute/aws-app-runner.md) | Regional | ECS/Fargate (pro: simpler, con: less control) | Fully managed service to run containerized web apps | Quickly deploy container web apps | Deep infra customization | Limited runtime control | Per-vCPU + memory pricing
| [AWS Auto Scaling](compute/aws-auto-scaling.md) | Regional | Manual scaling / ASG (pro: simple, con: less automation) | Adjusts capacity for EC2/ASG based on policies/metrics | Variable load handling | Not a replacement for architecture design | Policy complexity, cooldowns | No extra charge (pay for resources)
| [AWS Batch](compute/aws-batch.md) | Regional | ECS/EKS (pro: flexible scheduling, con: more setup) | Batch compute orchestration for large-scale jobs | HPC & batch workloads | Low-latency services | Job queue limits, instance provisioning delays | Pay for underlying compute (EC2/Fargate)
| [Amazon EC2](compute/amazon-ec2.md) | Regional (AZ instances) | Fargate (pro: serverless containers / con: less control) | Virtual servers with full OS control | Custom workloads, legacy apps | Serverless-first designs | Patching, scaling responsibility | Instance pricing (on-demand, reserved, spot)
| [Amazon EC2 Auto Scaling](compute/amazon-ec2-auto-scaling.md) | Regional | Manual scaling (pro: control, con: manual) | Scales EC2 instances across AZs via groups/policies | High availability, cost optimization | Single-instance setups | Requires proper health checks/policies | No extra charge (resource costs apply)
| [AWS Elastic Beanstalk](compute/aws-elastic-beanstalk.md) | Regional | EKS/ECS (pro: containers, con: more ops) | PaaS for deploying apps, manages infra lifecycle | Rapid app deployment | Highly custom infra | Limited deep infra control | Pay for underlying resources only
| [Amazon EKS](compute/amazon-eks.md) | Regional | Amazon ECS (pro: simpler, con: less k8s features) | Managed Kubernetes control plane | K8s workloads, portability | Very small teams unfamiliar with k8s | Control-plane fee, cluster ops | Control plane fee + worker instances
| [Elastic Load Balancing](compute/elastic-load-balancing.md) | Regional (per AZ) | Third-party LB (e.g., F5) (pro: features, con: cost) | Distributes traffic across targets (ALB/NLB/CLB) | Load distribution, HA | Very low-latency edge cases | Cost scales with usage, features vary by type | Per-hour + per-GB processed
| [AWS Fargate](compute/aws-fargate.md) | Regional | EC2 (pro: more control, con: infra to manage) | Serverless containers, no servers to manage | Containerized apps without infra | Workloads needing kernel access | Less host-level control, cold starts | vCPU & memory per-second pricing
| [AWS Lambda](compute/aws-lambda.md) | Regional | Fargate (pro: longer tasks, con: more infra) | Event-driven serverless functions | Short-lived event processing | Long-running processes (>15m) | Max 15m, limited ephemeral storage | Requests + duration + memory billing
| [Amazon Lightsail](compute/amazon-lightsail.md) | Regional | EC2 (pro: flexibility, con: complexity) | Simplified VM + managed resources with fixed pricing | Small apps, simple sites | Large production fleets | Limited advanced AWS features | Bundled monthly pricing
| [AWS Outposts](compute/aws-outposts.md) | On-prem / Local | Local Zones (pro: lower latency, con: hardware) | AWS-managed rack for on-prem workloads | Strict low-latency or data residency | Pure cloud-native apps | Procurement and footprint limits | Hardware + service billing
| [AWS Wavelength](compute/aws-wavelength.md) | Edge (carrier zones) | Local Zones (pro: broader coverage) | Ultra-low-latency compute at telco edge | 5G/mobile edge apps | General web apps | Very limited locations | vCPU/memory + data transfer fees

### Containers

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon ECR](containers/amazon-ecr.md) | Regional | Docker Hub (pro: public sharing) | Managed container image registry | Storing container images securely | Very large cross-cloud registries | Repository quotas & lifecycle rules | Per-GB storage + data transfer
| [Amazon ECS](containers/amazon-ecs.md) | Regional | EKS (pro: simpler) | Container orchestration for Docker containers | Simple containerized services | Complex k8s-native workloads | Fewer k8s features than EKS | No control plane fee; resource costs apply
| [Amazon ECS Anywhere](containers/amazon-ecs-anywhere.md) | On-prem / Regional | Self-managed ECS | Run ECS tasks on customer-managed infrastructure | Hybrid container workloads | Pure cloud-native provisioning | Connectivity & management overhead | ECS pricing + underlying infra
| [Amazon EKS](compute/amazon-eks.md) | Regional | ECS (pro: simplicity) | Managed Kubernetes control plane | K8s portability & ecosystem | Small teams without k8s skills | Control plane fee + cluster ops | Control plane fee + worker resources
| [Amazon EKS Anywhere](containers/amazon-eks-anywhere.md) | On-prem | Self-hosted k8s | Deploy EKS-compatible clusters on-prem | Offline/air-gapped k8s | Fully managed cloud convenience | Operational complexity | Software + infra costs
| [Amazon EKS Distro](containers/amazon-eks-distro.md) | Open-source | Other k8s distros | Upstream Kubernetes distribution matching EKS | Build-your-own cluster with EKS compatibility | Expectation of managed service | No managed control plane | Free distro, infra costs apply

### Database

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon Aurora](database/amazon-aurora.md) | Regional (multi-AZ) | Amazon RDS (pro: simplicity, con: lower scale) | MySQL/Postgres-compatible, distributed storage for OLTP | High-throughput relational workloads | Extremely low-cost single-node needs | Region-limited features, cost | Instance + storage; Aurora Serverless has ACU billing
| [Amazon Aurora Serverless](database/amazon-aurora-serverless.md) | Regional | Aurora provisioned (pro: predictable perf) | Autoscaling DB capacity (ACUs) for variable workloads | Spiky or infrequent DB usage | Heavy steady-state workloads | Cold-starts, connection management | ACU-based per-second billing
| [Amazon DocumentDB](database/amazon-documentdb.md) | Regional | MongoDB Atlas (pro: parity, con: external) | Managed MongoDB-compatible document DB | Mongo-style apps needing managed service | Cutting-edge Mongo features | Compatibility gaps, version lag | Instance + storage billing
| [Amazon DynamoDB](database/amazon-dynamodb.md) | Regional (supports global tables) | Amazon RDS / DocumentDB (pro: complex queries) | Serverless key-value & document DB, single-digit ms latency | Massive scale, predictable latency | Complex relational queries, ad-hoc analytics | Item size limits (400KB), cost for RCUs/WCUs | On-demand or provisioned capacity + storage
| [Amazon ElastiCache](database/amazon-elasticache.md) | Regional (AZ nodes) | Self-managed Redis (pro: control, con: ops) | Managed Redis/Memcached in-memory cache | Low-latency caching, session stores | Durable persistent storage | Node limits, failover considerations | Node-hour pricing
| [Amazon Keyspaces](database/amazon-keyspaces.md) | Regional | Self-managed Cassandra (pro: control, con: ops) | Managed Cassandra-compatible wide-column DB | Scale-out write-heavy workloads | Complex transactional queries | Cassandra query model limits | Capacity or on-demand pricing
| [Amazon Neptune](database/amazon-neptune.md) | Regional | JanusGraph on EC2 (pro: choice, con: ops) | Managed graph DB (Gremlin/SPARQL) | Graph queries & relationships | Simple relational data | Instance sizing & query patterns | Instance + storage pricing
| [Amazon RDS](database/amazon-rds.md) | Regional (multi-AZ) | Aurora (pro: performance, con: cost) | Managed relational DBs (MySQL, Postgres, etc.) | Lift-and-shift relational apps | Massive analytically scaled workloads | Vertical scaling limits | Instance + storage + I/O charges
| [Amazon Redshift](database/amazon-redshift.md) | Regional | Snowflake / BigQuery (pro: separation of storage/compute) | Columnar data warehouse for analytics | Large-scale analytics, BI | Small ad-hoc reporting | Concurrency and maintenance windows | Node-hours + storage; RA3 pricing options
| [Amazon Timestream](database/amazon-timestream.md) | Regional | InfluxDB (pro: OSS control) | Serverless time-series DB for metrics & telemetry | IoT/monitoring telemetry | General-purpose relational data | Query expressiveness limits | Ingest + storage pricing

### Developer Tools

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS Cloud9](developer-tools/aws-cloud9.md) | Regional | Local IDEs (VSCode) | Cloud IDE with dev environment pre-configured | Quick cloud dev environments | Offline dev workflows | Web-based latency | IDE included; pay for running instances
| [AWS CodeArtifact](developer-tools/aws-codeartifact.md) | Regional | Artifactory/Nexus | Hosted package repository (npm, pip, maven) | Secure package storage & sharing | Very small teams | Integration and retention policies | Per-GB stored + requests
| [AWS CodeBuild](developer-tools/aws-codebuild.md) | Regional | Jenkins (self-hosted) | Managed build service that runs CI builds | Serverless CI builds | Complex custom build servers | Build time limits & concurrency | Per-minute build compute pricing
| [AWS CodeCommit](developer-tools/aws-codecommit.md) | Regional | GitHub/GitLab | Managed Git hosting in AWS | Private repos with AWS integration | Rich social coding features | Fewer community integrations | Per-user storage limits apply
| [AWS CodeDeploy](developer-tools/aws-codedeploy.md) | Regional | Manual deploy scripts | Automates deployments to EC2/ECS/Lambda | Controlled, repeatable deploys | Very custom deployment flows | Agent/permissions required | No separate charge; resource usage applies
| [Amazon CodeGuru](developer-tools/amazon-codeguru.md) | Regional | SonarQube / SCA tools | Automated code reviews and profiling suggestions | Improve code quality & perf | Advanced static analysis needs | Language and rule coverage limits | Per-review + profiler charges
| [AWS CodePipeline](developer-tools/aws-codepipeline.md) | Regional | GitHub Actions (pro: community) | CI/CD orchestration service | Orchestrating pipelines across AWS | Complex multi-cloud pipelines | Integrations limit | Per-pipeline per-month pricing
| [AWS CodeStar](developer-tools/aws-codestar.md) | Regional | Custom project templates | Project management + dev tools starter kit | Kickstarting projects with CI/CD | Mature org tooling | Limited customization | Included in related service costs
| [AWS X-Ray](developer-tools/aws-xray.md) | Regional | Zipkin / Jaeger | Distributed tracing and service map | Performance debugging across services | Non-AWS tracing needs | Sampling and overhead | Trace storage + data processing

### End User Computing

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon AppStream 2.0](end-user-computing/amazon-appstream-2-0.md) | Regional | VDI solutions (e.g., Citrix) | Stream desktop applications from AWS | Delivering apps without local installs | Offline users | Network-dependent UX | Per-stream instance + user fees
| [Amazon WorkSpaces](end-user-computing/amazon-workspaces.md) | Regional | RDP on EC2 / VDI | Managed virtual desktops | Persistent user desktops | Very high-performance local needs | Session latency & disk limits | Per-user per-month or hourly pricing

### Frontend Web and Mobile

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS Amplify](frontend-web-and-mobile/aws-amplify.md) | Regional | Custom CI/CD + hosting | Framework & hosting for web/mobile apps | Rapid frontend iteration & hosting | Complex backend orchestration | Opinionated structure | Hosting + build + backend charges
| [Amazon API Gateway](frontend-web-and-mobile/amazon-api-gateway.md) | Regional | ALB + Lambda (pro: cost) | Managed API endpoint hosting with throttling & auth | Serverless APIs and throttling | Extremely high RPS at low cost | Payload size & latency | Per-request + data transfer
| [AWS Device Farm](frontend-web-and-mobile/aws-device-farm.md) | Regional | Local device labs | Test apps on real devices in cloud | Mobile app compatibility testing | On-prem device testing | Device availability & concurrency | Per-device-minute or remote access fees
| [Amazon Pinpoint](frontend-web-and-mobile/amazon-pinpoint.md) | Regional | 3rd-party messaging platforms | Customer engagement, analytics and messaging | Targeted campaigns and analytics | Extremely custom campaign tools | Feature set complexity | Per-message + endpoint charges

### Internet of Things

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS IoT Analytics](internet-of-things/aws-iot-analytics.md) | Regional | Kinesis + analytics (pro: control) | Analytics for IoT device data | Processing/time-series IoT data | Non-IoT analytics | Integration with IoT Core needed | Per-ingest + storage pricing
| [AWS IoT Core](internet-of-things/aws-iot-core.md) | Regional | MQTT brokers on EC2 | Device connectivity, messaging, registry | Secure device messaging & auth | Simple cloud-only devices | Scale and thing registry limits | Per-message + connection pricing
| [AWS IoT Device Defender](internet-of-things/aws-iot-device-defender.md) | Regional | Custom security tooling | Audit and monitor IoT fleets for security | Fleet security posture | Single-device troubleshooting | Rule & metric configuration | Per-device assessment & audit charges
| [AWS IoT Device Management](internet-of-things/aws-iot-device-management.md) | Regional | Custom device management | Fleet provisioning, OTA updates and management | Large device fleets | Small localized management | Device registry scale | Per-thing and management fees
| [AWS IoT Events](internet-of-things/aws-iot-events.md) | Regional | Custom event detection | Detect and respond to IoT events and patterns | Pattern detection across telemetry | Very complex event logic | Rule language limits | Per-detector-hour + input charges
| [AWS IoT Greengrass](internet-of-things/aws-iot-greengrass.md) | Regional / Edge | Edge containers | Local compute, messaging, ML inference at edge | Low-latency edge processing | Pure cloud-only apps | Device management complexity | Per-device software + resources
| [AWS IoT SiteWise](internet-of-things/aws-iot-sitewise.md) | Regional | Time-series DB + custom dashboards | Industrial equipment data modeling and visualization | Industrial telemetry and KPIs | Generic IoT telemetry | Modeling effort | Data ingestion + storage pricing
| [AWS IoT Things Graph](internet-of-things/aws-iot-things-graph.md) | Regional | Custom orchestration | Visual workflow for IoT devices | Orchestrating device interactions | Modern orchestration via services | Limited adoption & updates | Per-flow / per-device charges
| [AWS IoT 1-Click](internet-of-things/aws-iot-1-click.md) | Regional | Custom device triggers | Simple one-button device event triggers to AWS | Very simple device-triggered actions | Complex device logic | Limited device types | Per-trigger and device fees

### Machine Learning

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon Comprehend](machine-learning/amazon-comprehend.md) | Regional | spaCy / open-source NLP | Managed NLP (entity/sentiment) | Text analytics & entity extraction | Custom NLP research | Language support and model limits | Per-unit or per-API pricing
| [Amazon Forecast](machine-learning/amazon-forecast.md) | Regional | Prophet / in-house (pro: control) | Managed time-series forecasting service | Demand & resource forecasting | Research-level models | Data prep requirements | Per-forecast-hour + storage
| [Amazon Fraud Detector](machine-learning/amazon-fraud-detector.md) | Regional | Custom ML models | Managed fraud detection templates | Quick fraud model deployment | Highly custom fraud strategies | Limited model customization | Per-evaluation + training charges
| [Amazon Kendra](machine-learning/amazon-kendra.md) | Regional | Elasticsearch + custom search | Enterprise search with ML relevance | Natural language search across docs | Simple keyword search | Data connector limits | Per-query + capacity pricing
| [Amazon Lex](machine-learning/amazon-lex.md) | Regional | Dialogflow / custom bots | Conversational interfaces and chatbots | Simple conversational flows | Very advanced dialog requirements | Voice & channel integration limits | Per-request + text/voice pricing
| [Amazon Personalize](machine-learning/amazon-personalize.md) | Regional | Custom recommender systems | Personalization and recommendation as a service | Real-time recommendations | Highly specialized recommenders | Data preparation & feature limits | Per-campaign + inference pricing
| [Amazon Polly](machine-learning/amazon-polly.md) | Regional | Google TTS / open-source TTS | Text-to-speech voices and SSML | Generating speech from text | Ultra-natural neural voice needs | Voice selection & cost | Per-character or per-request pricing
| [Amazon Rekognition](machine-learning/amazon-rekognition.md) | Regional | OpenCV / custom vision | Image and video analysis (labels, faces) | Image moderation and detection | Deep custom models | Accuracy on edge cases | Per-image / per-minute video pricing
| [Amazon SageMaker](machine-learning/amazon-sagemaker.md) | Regional | Self-managed ML infra | Full ML lifecycle: train, tune, deploy | End-to-end ML pipelines | Small one-off models | Cost for heavy training | Instance-hour + training/inference costs
| [Amazon Textract](machine-learning/amazon-textract.md) | Regional | Tesseract + workflows | Extract text and structured data from documents | OCR and form extraction | Very noisy documents | Accuracy depends on input quality | Per-page pricing
| [Amazon Transcribe](machine-learning/amazon-transcribe.md) | Regional | Open-source ASR | Speech-to-text transcription | Transcribing audio to text | Very noisy audio | Accuracy & language support | Per-second transcribed pricing
| [Amazon Translate](machine-learning/amazon-translate.md) | Regional | Open-source MT | Neural machine translation service | Quick translations across many languages | Domain-specific jargon | Customization limits | Per-character pricing

### Management and Governance

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS CloudFormation](management-and-governance/aws-cloudformation.md) | Regional | Terraform (pro: multi-cloud, con: external state) | Declarative IaC for provisioning AWS resources | Repeatable infra provisioning | Complex cross-stack dependencies | Stack limits, template complexity | Free (pay for resources provisioned)
| [AWS CloudTrail](management-and-governance/aws-cloudtrail.md) | Global | Third-party logging (con: integration) | Audit trail of API activity and events | Security and compliance auditing | Real-time alerting (use CloudWatch/GuardDuty) | Large logs storage costs | Management events free; data events and insights billed
| [Amazon CloudWatch](management-and-governance/amazon-cloudwatch.md) | Regional | Prometheus + Grafana (open-source) | Monitoring, metrics, alarms and dashboards | Operational monitoring & alerting | Very large metric volumes | Cost for metrics, logs & dashboards | Metrics, logs ingestion and dashboard charges
| [Amazon CloudWatch Logs](management-and-governance/amazon-cloudwatch-logs.md) | Regional | ELK stack (self-managed) | Centralized log ingestion and retention | Application and infra logs | High-volume logs cost | Ingestion, retention and query costs | Per-GB ingestion + storage + queries
| [AWS CLI](management-and-governance/aws-cli.md) | Global | SDKs / web console | Command-line interface for AWS | Scripting and automation | GUI-oriented tasks | None | Free
| [AWS Compute Optimizer](management-and-governance/aws-compute-optimizer.md) | Regional | Cost Explorer recommendations | Rightsizing recommendations for compute | Cost and performance optimization | Custom workload nuance | Recommendation accuracy limits | Free recommendations (no direct charge)
| [AWS Config](management-and-governance/aws-config.md) | Regional | Custom inventory tools | Resource configuration tracking and compliance | Drift detection & compliance auditing | High-volume recording costs | Resource coverage and retention | Per-configuration-item + conformance packs
| [AWS Control Tower](management-and-governance/aws-control-tower.md) | Multi-region | Landing zone frameworks | Automated multi-account landing zone and guardrails | Quick enterprise account setup | Highly custom infra | Opinionated templates & lifecycle | No separate charge; underlying services billed
| [AWS License Manager](management-and-governance/aws-license-manager.md) | Regional | Manual license tracking | Manage and track software licenses across AWS | BYOL license management | Very complex licensing models | Vendor-specific license rules | Free
| [Amazon Managed Grafana](management-and-governance/amazon-managed-grafana.md) | Regional | Self-hosted Grafana | Managed Grafana with integrations to AWS data sources | Dashboards & visualization | Extreme plugin/custom needs | Plugin availability limits | Workspace + user pricing
| [Amazon Managed Service for Prometheus](management-and-governance/amazon-managed-service-for-prometheus.md) | Regional | Self-managed Prometheus | Managed Prometheus-compatible monitoring | Container metrics & observability | Very low-cost self-hosting | Ingest/storage limits | Ingest + storage pricing
| [AWS Management Console](management-and-governance/aws-management-console.md) | Global | AWS CLI / SDKs | Web-based management console for AWS | Human-friendly management | Automation at scale | Not scriptable | Free
| [AWS Organizations](management-and-governance/aws-organizations.md) | Global | Manual account structure | Centralized account management and SCPs | Multi-account governance | Very small orgs | SCP complexity | Free
| [AWS Personal Health Dashboard](management-and-governance/aws-personal-health-dashboard.md) | Global | Third-party status pages | Personalized alerts and guidance for AWS events | Incident awareness for account | General public status needs | Coverage for account-specific events | Free (detailed health with support plans)
| [AWS Proton](management-and-governance/aws-proton.md) | Regional | Custom templates & CD tools | Managed delivery for container and serverless services | Platform engineering and standardization | Small teams | Template authoring complexity | No separate charge; infra billed
| [AWS Service Catalog](management-and-governance/aws-service-catalog.md) | Regional | Terraform modules | Curated catalog of approved products | Governance and approved stacks | Rapid prototyping | Product lifecycle management | Free (underlying resources billed)
| [Service Quotas](management-and-governance/service-quotas.md) | Regional | Manual tracking | View and request increases for service quotas | Quota management & planning | Very small setups | Some quotas not requestable | Free
| [AWS Systems Manager](management-and-governance/aws-systems-manager.md) | Regional | Ansible / SSH | Ops tooling: patching, runbooks, inventory | Patching, automation & session manager | Non-AWS systems complexity | Agent requirement for some features | Some features free; others billed
| [AWS Trusted Advisor](management-and-governance/aws-trusted-advisor.md) | Global | Third-party advisors | Best-practice checks across cost, security, performance | Quick optimization checks | Deep custom audits | Full checks require support plan | Basic checks free; full checks need Business/Enterprise support
| [AWS Well-Architected Tool](management-and-governance/aws-well-architected-tool.md) | Regional | Third-party review frameworks | Run Well-Architected reviews and generate improvement plans | Architecture reviews & guidance | Enforcing changes | Advisory only | Free

### Media Services

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon Elastic Transcoder](media-services/amazon-elastic-transcoder.md) | Regional | MediaConvert (pro: advanced features) | Legacy media transcoding for simple workflows | Simple format conversions | High-quality broadcast workflows | Feature limitations vs MediaConvert | Per-minute + output resolution pricing
| [Amazon Kinesis Video Streams](media-services/amazon-kinesis-video-streams.md) | Regional | Self-managed media ingest | Ingest, store and analyze video streams | Real-time video analytics & playback | Bulk video archival | Retention and playback costs | Per-stream + storage + data transfer

### Migration and Transfer

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS Application Discovery Service](migration-and-transfer/aws-application-discovery-service.md) | Regional | Manual discovery tools | Discovers on-prem servers & app dependencies | Migration assessment & inventory | Very small inventories | Agent/scan coverage limits | Per-agent / per-scan pricing (varies)
| [AWS Application Migration Service](migration-and-transfer/aws-application-migration-service.md) | Regional | Manual migration (pro: control) | Server/VM replication & lift-and-shift migration | Fast rehosting of servers to AWS | Deep replatforming | Replication bandwidth & testing needs | Replication + staging resource costs
| [AWS Database Migration Service (DMS)](migration-and-transfer/aws-database-migration-service.md) | Regional | Custom ETL or dump/restore | Database migration with minimal downtime | Homogeneous & heterogeneous DB migrations | Very complex schema conversions | Data type compatibility & validation | Per-instance + data transfer
| [AWS DataSync](migration-and-transfer/aws-datasync.md) | Regional | Rsync / custom tools | Fast online data transfer between on-prem and AWS | Large-scale file transfers | Small ad-hoc copies | Throughput & agent provisioning | Per-GB transferred + agent costs
| [AWS Migration Hub](migration-and-transfer/aws-migration-hub.md) | Regional | Custom tracking spreadsheets | Central tracking and progress for migrations | Migration project visibility | Small one-off migrations | Integration with tools required | Free (service used for tracking)
| [AWS Schema Conversion Tool (SCT)](migration-and-transfer/aws-sct.md) | Regional | Manual schema conversion | Converts schema and code between DB engines | Assessing & converting schema for migrations | Complex stored-proc conversions | Not 100% automated for all features | Free tool; migration effort costs
| [AWS Snow Family](migration-and-transfer/aws-snow-family.md) | Edge / Regional | Online transfer (pro: faster setup) | Physical devices for large data transfer to AWS | Petabyte-scale offline transfer | Small data sizes | Logistics & import/export timing | Device rental + data transfer/import fees
| [AWS Transfer Family](migration-and-transfer/aws-transfer-family.md) | Regional | SFTP servers on EC2 | Managed SFTP/FTPS/FTP to S3/EFS | Managed file transfers and partner integrations | Highly custom transfer protocols | Protocol-specific limits | Per-hour endpoint + per-GB transfer

### Networking and Content Delivery

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [Amazon CloudFront](networking/amazon-cloudfront.md) | Global (edge) | API Gateway + S3 (pro: simplicity, con: no global CDN) | Global CDN for low-latency content delivery | Static + dynamic content at edge | Origin-heavy dynamic apps without caching | Cache invalidation complexity, regional edge limits | Data transfer + request pricing
| [AWS Direct Connect](networking/aws-direct-connect.md) | Location-based / Regional | VPN over internet (pro: cheaper, con: less stable) | Dedicated private network connection to AWS | High-throughput, predictable network | Small, ad-hoc connections | Requires colocations & setup time | Port/hour + data transfer pricing
| [Elastic Load Balancing](compute/elastic-load-balancing.md) | Regional (per AZ) | Self-managed LB (pro: control, con: ops) | ALB/NLB/CLB options to distribute traffic | HTTP routing, TLS termination, TCP load | Extremely cost-sensitive small apps | Features vary by type (ALB vs NLB) | Per-hour + per-GB processed
| [AWS Global Accelerator](networking/aws-global-accelerator.md) | Global | CloudFront (pro: CDN, con: caching) | Global static anycast IPs to improve global performance | Global apps needing static IPs | Simple regional apps | Limited routing control | Per-accelerator + data transfer fees
| [AWS PrivateLink](networking/aws-privatelink.md) | Regional | VPN/Transit Gateway (pro: simpler) | Private connectivity to services via ENIs | Secure service-to-service comms | Very large-scale network fabrics | Endpoint limits and costs | Endpoint-hour + data processing
| [Amazon Route 53](networking/amazon-route53.md) | Global (DNS) | Third-party DNS (e.g., Cloudflare) | Authoritative DNS, routing policies, health checks | DNS-based failover and routing | Private internal DNS only | DNS propagation, TTL behavior | Hosted zone + per-query charges
| [AWS Transit Gateway](networking/aws-transit-gateway.md) | Regional (supports inter-region peering) | VPC peering (pro: cheap for few VPCs) | Hub-and-spoke network connectivity between VPCs | Large multi-VPC network architectures | Very small deployments | Attachment limits, cost per GB | Attachment + data processing charges
| [Amazon VPC](networking/amazon-vpc.md) | Regional (AZs inside VPC) | Classic networking on-prem (pro: control) | Isolated virtual networks in a region | Network isolation, security boundaries | Global network topologies | Subnet design, IP limits | No direct charge (resource charges apply)
| [AWS VPN](networking/aws-vpn.md) | Regional | Direct Connect (pro: dedicated) | Encrypted site-to-site / client VPN connectivity | Quick secure tunnels to AWS | Extremely high throughput needs | Throughput & latency constraints | Per-connection + data transfer

### Security, Identity, and Compliance

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS Artifact](security/aws-artifact.md) | Global | Vendor attestation repos (pro: direct, con: manual) | Compliance and audit reports access | Compliance evidence collection | Not an enforcement tool | Documentation-only, no enforcement | Free to access (service dependent)
| [AWS Audit Manager](security/aws-audit-manager.md) | Regional | Third-party audit tools | Automates evidence collection for audits | Continuous audit readiness | Requires mapping to frameworks | Coverage varies by service | Charges based on assessments
| [AWS Certificate Manager](security/aws-certificate-manager.md) | Regional (public certs usable globally) | Third-party CAs (pro: options) | Provision and manage TLS certificates | TLS for AWS endpoints | Internal-only PKI needs | Public cert limits, regional endpoints | Free for public certs; ACM PCA paid
| [AWS CloudHSM](security/aws-cloudhsm.md) | Regional | AWS KMS (pro: managed keys, con: less control) | Dedicated HSMs for customer-managed keys | FIPS-level key storage & operations | Simple key management use-cases | Hardware management & cost | HSM cluster hourly billing
| [Amazon Cognito](security/amazon-cognito.md) | Regional | Auth0 / OIDC providers (pro: features) | Managed user auth, federation and user pools | Mobile & web app auth | Enterprise SSO-only needs | Limited advanced identity features | User/month + MAU pricing for some features
| [Amazon Detective](security/amazon-detective.md) | Regional | SIEM tools (pro: broader integrations) | Investigative security service with graph analysis | Threat investigation & root cause | Replacing SIEM entirely | Data ingestion costs, retention | Data processing & storage charges
| [AWS Directory Service](security/aws-directory-service.md) | Regional | Self-managed AD (pro: control) | Managed Microsoft AD and AD connectors | AD-dependent apps | Non-AD identity needs | Domain management constraints | Instance-hour + usage fees
| [AWS Firewall Manager](security/aws-firewall-manager.md) | Regional | Manual firewall management | Central policy management for firewall/WAF | Policy enforcement across accounts | Small single-account setups | Integration limits with services | Policy-management fees (varies)
| [Amazon GuardDuty](security/amazon-guardduty.md) | Regional | Third-party threat detection | Managed threat detection with findings | Continuous threat monitoring | Deep custom telemetry needs | Data volume-based costs | Per-GB or per-scan pricing
| [AWS IAM](security/aws-iam.md) | Global (account-level) | External IdPs (pro: SSO) | Identity & access management for AWS resources | Fine-grained permissions | Replacing with external IAM only | Human error risk with policies | Free (charged via other services)
| [Amazon Inspector](security/amazon-inspector.md) | Regional | Third-party scanners | Automated vulnerability scanning for EC2/ECR | Finding security issues | Full platform scanning needs | Coverage depends on resource types | Assessment run pricing
| [AWS KMS](security/aws-kms.md) | Regional (multi-region keys optional) | CloudHSM (pro: dedicated HW) | Managed encryption keys & cryptographic operations | Central key management | HSM-level compliance needs | API quotas, regional keys | Per-request + key storage charges
| [Amazon Macie](security/amazon-macie.md) | Regional | Custom DLP (pro: flexible) | Data loss prevention for S3, discovers sensitive data | Discover sensitive S3 data | Non-S3 data stores | S3-focused, false positives | Data discovery & classification pricing
| [AWS Network Firewall](security/aws-network-firewall.md) | Regional | Third-party NGFWs | Managed network firewall for VPCs | VPC-level perimeter controls | Complex inline inspection needs | Throughput limits per firewall | Endpoint-hour + data processing
| [AWS RAM](security/aws-ram.md) | Regional | Resource sharing via IAM | Share resources across accounts/orgs | Cross-account resource access | Simple single-account infra | Service support varies | No charge (resource charges apply)
| [AWS Secrets Manager](security/aws-secrets-manager.md) | Regional | Parameter Store (pro: cheaper) | Secure secret rotation, storage and retrieval | Application secrets with rotation | Extremely high-volume secrets | Cost per secret + API calls | Per-secret per-month + request charges
| [AWS Security Hub](security/aws-security-hub.md) | Regional | SIEMs (pro: deeper correlation) | Central view and standardization of security findings | Consolidated security posture | Single-service use | Dependence on integrated findings | Per-account service charges
| [AWS STS](security/aws-sts.md) | Global | Long-lived creds (pro: simplicity) | Token service for temporary credentials | Cross-account access, temporary creds | Persistent credential scenarios | Token duration limits | Free (used by other services)
| [AWS Shield](security/aws-shield.md) | Global (Standard) / Regional (Advanced) | Third-party DDoS mitigation | DDoS protection for AWS resources | Protects against network-layer attacks | Complete mitigation for huge attacks | Advanced has cost, Standard is free | Standard free; Advanced paid subscription
| [AWS Single Sign-On](security/aws-single-sign-on.md) | Regional | Third-party SSO | Centralized SSO across AWS accounts & apps | Multi-account access management | Very custom SSO flows | Integration limitations | Per-user or included in IAM Identity Center
| [AWS WAF](security/aws-waf.md) | Regional / CloudFront (global) | Third-party WAFs | Web application firewall to block common exploits | Protecting web apps at edge or regional | Deep custom parsing | Rule capacity and cost | Request + rule charges

### Storage

| Name | Availability | Alternatives (main pro / cons) | Short description | Good for | Bad for | Main limitations | Cost info |
|---|---|---|---|---|---|---|---|
| [AWS Backup](storage/aws-backup.md) | Regional | Custom scripts + lifecycle (pro: flexible, con: manual) | Centralized backup orchestration for AWS services | Centralized restore and lifecycle | Niche services may need custom tooling | Not all services fully supported | Pricing based on backup storage & restores
| [Amazon EBS](storage/amazon-ebs.md) | AZ (volume attached to AZ) | EFS (pro: shared filesystem) | Block storage for EC2, low-latency | Databases, single-instance block storage | Shared filesystem needs | Tied to AZ, snapshot-based cross-AZ | Per-GB-month + provisioned IOPS charges
| [AWS Elastic Disaster Recovery](storage/aws-elastic-disaster-recovery.md) | Regional | CloudEndure / third-party DR | Orchestrated recovery of servers to AWS | Rapid recovery & minimal RTO | Simple backup-only needs | Requires replication setup | Replication + storage + compute costs
| [Amazon EFS](storage/amazon-efs.md) | Regional (mount targets in AZs) | FSx (pro: performance options) | Fully-managed NFS file system for Linux | Shared file storage across instances | Single-instance block perf | Performance modes, throughput limits | Per-GB-month + throughput (if provisioned)
| [Amazon FSx](storage/amazon-fsx.md) | Regional | EFS / self-managed (pro: choice) | Managed high-performance file systems (Lustre, Windows) | High-performance or Windows workloads | Small-scale simple file needs | Specific OS/filesystem feature limits | Per-GB + throughput/IOPS pricing
| [Amazon S3](storage/amazon-s3.md) | Regional (global namespace) | Azure Blob / GCS | Object storage, highly durable & available | Static hosting, backups, big data | Low-latency block storage | Consistency semantics, request costs | Per-GB-month + requests + data transfer
| [Amazon S3 Glacier](storage/amazon-s3-glacier.md) | Regional | S3 Standard (pro: immediate) | Low-cost archival storage class | Long-term cold storage | Frequent-access data | Retrieval latency and fees | Low storage cost, retrieval pricing applies
| [AWS Storage Gateway](storage/aws-storage-gateway.md) | On-prem / Hybrid | Direct uploads to S3 (pro: simple) | Hybrid storage appliance for on-prem to AWS | Migrate on-prem backups to S3 | Pure cloud-native apps | Appliance management & network dependency | VM/hour + storage transfer
