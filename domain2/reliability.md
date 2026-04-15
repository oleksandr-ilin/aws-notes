# Reliability

## Knowledge of:

- AWS global infrastructure
- AWS storage services and replication strategies (for example Amazon S3, Amazon RDS, Amazon ElastiCache)
- Multi-AZ and multi-Region architectures
- Auto scaling policies and events
- Application integration (for example, Amazon SNS, Amazon SQS, AWS Step Functions)
- Service quotas and limits

## Skills in:

- Designing highly available application environments based on business requirements
- Leveraging advanced techniques to design for failure and ensure seamless system recoverability
- Implementing loosely coupled dependencies
- Operating and maintaining high-availability architectures (for example, application failovers, database failovers)
- Leveraging AWS managed services for high availability
- Implementing DNS routing policies (for example, Route 53 latency-based routing, geolocation routing, simple routing)


## What to understand

- Multi-AZ designs for all tiers: compute (ALB + ASG across AZs), database (RDS Multi-AZ, Aurora), caching (ElastiCache Multi-AZ)
- SQS dead-letter queues, retry strategies, and idempotent processing
- Step Functions for orchestrating long-running workflows with error handling
- SNS fan-out for event-driven decoupling
- Auto Scaling policies: target tracking, step, scheduled, and predictive
- Route 53 routing policies: failover, latency, weighted, geolocation, multivalue answer
- Service quotas planning and request strategies
- Circuit breaker patterns in distributed systems
- Storage replication: S3 CRR, RDS cross-Region replicas, ElastiCache Global Datastore


## Services to know

- `Amazon SQS` — message queuing and decoupling
- `Amazon SNS` — pub/sub event notifications
- `AWS Step Functions` — workflow orchestration
- `Amazon Route 53` — DNS routing policies
- `EC2 Auto Scaling` — capacity management
- `Amazon RDS` — Multi-AZ, read replicas
- `Amazon Aurora` — Global Database, Multi-AZ
- `Amazon ElastiCache` — Multi-AZ, Global Datastore
- `Amazon S3` — Cross-Region Replication
- `Service Quotas` — limit management


## What to expect

This task statement also relates to the last task statement for this domain and they both pertain to the Reliability pillar of the Well-Architected Framework. However, last task statement focused on disaster recovery, or reacting to an outage. Questions for task statement 4 about reliability focus on preventing and isolating failures. Ensure you can design and implement highly available or fault-tolerant application environments based on business requirements. Unlike RPO and RTO, which define the amount of allowable downtime and data loss, availability requirements are expressed as uptime. Dive deeper to ensure you know how the AWS global infrastructure can help ensure  high availability by using multi-Region and multi-AZ deployments. In addition to supporting failover, a multi-Region deployment can reduce latency by placing resources closer to your customers. This increases the reliability of your application and can prevent issues like packet loss and message timeouts.You(opens in a new tab) should also know how to configure Amazon Route 53 to deliver stable and reliable performance using latency-based routing, geolocation routing, and failover routing. Optimizing each component of your workload is key to delivering solutions that are highly available and deliver reliable performance. Components should be loosely coupled to limit the impact of failures, scalable to meet demand and maintain stability, and highly available.First, we’ll talk about implementing loosely coupled dependencies. You should be familiar with synchronous decoupling, such as using an Application Load Balancer to distribute traffic between Amazon EC2 instances, or using API Gateway to route API calls between different Lambda functions. You should also be familiar with asynchronous decoupling, such as connecting Amazon Simple Queue Service with a worker application using a polling model, or using Amazon Simple Notification Service topics to fan out messages to multiple subscriber applications. Additionally, you’ll need to understand how to use serverless tools in AWS to build and improve decoupling mechanisms. What are the ways you can deploy Amazon SQS, Amazon API Gateway, Amazon DynamoDB, and the many other services within the serverless toolkit? Understanding how to implement these tools will help you to build flexibility and decoupling capabilities into your architectures, and it will also help in determining which responses will work and will work best.For the exam, you should be able to evaluate solutions that create additional resources using auto scaling to absorb increased demand.This includes capturing the correct metrics to determine when to perform a scaling action, for example, configuring your load balancers to perform health checks on instances in your EC2 Auto Scaling groups and setting CloudWatch Alarms to monitor CPU utilization.Next(opens in a new tab), you need an understanding of scaling actions for different AWS services, for example, adjusting EC2 instance counts in an EC2 Auto Scaling group, adjusting Amazon ECS task count in an ECS service, or modifying the provisioned throughput of an Amazon DynamoDB table. However, when selecting any scaling action on the exam, you need to consider resource constraints, such as the capacity of a storage class or the maximum amount of traffic over a specified type of network connection. You also need to consider service quotas and limits. Quotas limit the number of resources you can provision and request rates on APIs. You should be able to recognize component configurations that support stability and deliver high availability.For example, you should know the different Amazon RDS multi-AZ deployment types, such as multi-AZ database instances, multi-AZ database clusters, and read replicas. You should also know which database engines support each deployment type, and when to use each deployment type. Though they all support failover, the standby instance in a multi-AZ database instance deployment doesn’t support stability by taking some of the read traffic load off the primary database instance. You should understand how to cache common query results with Amazon ElastiCache to support database stability. When evaluating highly available storage solutions, consider how a service replicates your data to multiple locations. For example, for Amazon S3 know how to enable Cross-Region Replication to duplicate your files to different Regions. Also understand when to use AWS Global Accelerator with Amazon S3 Multi-Region Access Points to provide the lowest latency access to your replicated data.Familiarize yourself with AWS shared file systems such as Amazon Elastic File System and Amazon FSx to provide multi-AZ access to your data. File storage is also decoupled from your compute resources so a compute outage does not affect your storage.Ok, so that’s all for this video. Remember, if you were unfamiliar with any of the ideas or services covered in this video, consult the AWS documentation, and find opportunities for hands-on practice.