# Amazon MQ

## Service Overview
Amazon MQ is a managed message broker service for Apache ActiveMQ and RabbitMQ, suitable for lift-and-shift of broker-based applications.

## Tasks this service can solve
- Lift-and-shift messaging for legacy apps using JMS/AMQP
- Broker-based workflows requiring topics, durable subscriptions
- Interoperability with existing message broker clients

## Alternatives
| Name | Type | Short description | Pro vs Amazon MQ | Cons vs Amazon MQ | Price comparison |
|---|---|---:|---|---|---|
| Amazon SQS | AWS | Simple queue service | Fully managed, serverless | Lacks broker semantics (JMS) | SQS typically cheaper for simple queues |
| Self-hosted ActiveMQ / RabbitMQ | OSS | Run brokers on EC2 or k8s | Full control and plugins | Ops burden | Ops cost vs managed broker fees |
| Amazon MQ (RabbitMQ) vs ActiveMQ | AWS | Broker engine choices | RabbitMQ better for AMQP/modern clients | ActiveMQ for JMS legacy | Pricing similar across engines with instance-hour and storage costs |

## Limitations
- Broker instance scaling and failover require careful planning; throughput depends on instance sizes.
- Not as horizontally scalable as distributed streaming systems like Kafka.

## Price info
- Charged per broker-instance-hour and storage; network transfer and snapshot costs may apply.

## Network & Multi-region considerations
- Brokers are regional and typically deployed within AZs for high availability using multi-AZ replication features.
- For multi-region messaging, use cross-region bridges or prefer global patterns with EventBridge/SNS.

## Popular use cases
- Migrating on-prem JMS applications to AWS
- Transactional messaging requiring broker features
- Interoperability with enterprise message broker ecosystems

## When Not to Use This Service
- Not a great fit for cloud-native serverless patterns that benefit from SQS/SNS or streaming systems like MSK. Choose SQS/SNS for simple durable queues and EventBridge for routing.

## DR strategy
- Deploy brokers in multi-AZ configurations and maintain automated backups of broker configurations. For cross-region resilience, set up bridging or mirror queues and ensure persistent messages are stored and replicated (e.g., via S3 backups or cross-region clustering where supported).
