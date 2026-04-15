# Task Statement 5: Design a Solution to Meet Performance Objectives

## Overview

This task statement focuses on designing architectures that meet performance requirements through appropriate technology selection, caching strategies, elastic scaling, and purpose-built services.

## Key Knowledge Areas

### Performance Monitoring Technologies
- **CloudWatch** — metrics, alarms, dashboards, anomaly detection, Contributor Insights, Application Insights
- **X-Ray** — distributed tracing, service maps, latency analysis, segment/subsegment drill-down
- **CloudWatch RUM / Synthetics** — real user monitoring and synthetic canaries for front-end performance
- **Enhanced Monitoring** — OS-level metrics for RDS (CPU, memory, file system, disk I/O per process)

### Storage Options
- **EBS** — gp3 (baseline 3,000 IOPS, 125 MB/s — scale independently), io2 Block Express (up to 256,000 IOPS, 4,000 MB/s), st1 (throughput-optimized HDD), sc1 (cold HDD)
- **EFS** — Elastic Throughput (auto-scales), Provisioned Throughput (guaranteed), Max I/O vs. General Purpose performance modes
- **S3** — 3,500 PUT + 5,500 GET requests/sec per prefix, multi-part upload, S3 Transfer Acceleration, S3 Express One Zone
- **FSx** — FSx for Lustre (HPC, ML training), FSx for NetApp ONTAP (enterprise NAS), FSx for Windows File Server

### Instance Families and Use Cases
- **General purpose** (M-family): balanced workloads, web servers, app servers
- **Compute optimized** (C-family): batch processing, ML inference, HPC, gaming servers
- **Memory optimized** (R/X-family): in-memory databases, real-time analytics, SAP HANA
- **Storage optimized** (I/D-family): data warehousing, distributed file systems, high sequential I/O
- **Accelerated computing** (P/G/Inf/Trn-family): ML training, GPU rendering, video encoding
- **Graviton** (suffix 'g'): up to 40% better price-performance for compatible workloads

### Purpose-Built Databases
- **DynamoDB** — key-value/document, single-digit ms, auto scaling, global tables
- **ElastiCache (Redis/Memcached)** — sub-millisecond in-memory caching
- **Neptune** — graph database for relationship-heavy queries
- **Timestream** — time-series workloads (IoT telemetry, DevOps metrics)
- **QLDB** — immutable, cryptographically verifiable ledger
- **Keyspaces** — managed Apache Cassandra for wide-column workloads
- **MemoryDB for Redis** — durable in-memory database with Redis API
- **DocumentDB** — managed MongoDB-compatible document database
- **OpenSearch** — full-text search, log analytics, SIEM

## Core Skills

1. **Large-scale architecture for varied access patterns** — read-heavy (caching), write-heavy (write-behind, partitioning), mixed (CQRS), real-time (streaming)
2. **Elastic architecture** — auto scaling groups, Application Auto Scaling, predictive scaling, Spot Instances, Lambda concurrency, Aurora Serverless
3. **Caching, buffering, and replicas** — CloudFront CDN, ElastiCache, DAX, API Gateway caching, read replicas, Aurora reader endpoints, SQS buffering
4. **Purpose-built service selection** — match the access pattern to the database engine (relational vs. key-value vs. graph vs. time-series vs. search)
5. **Right-sizing strategy** — Compute Optimizer, CloudWatch metrics analysis, cost-performance trade-offs, Graviton migration

## Study Approach

- Understand when to use each cache layer (CDN → API cache → application cache → database cache)
- Know EBS volume types and when to choose each (IOPS vs. throughput vs. cost)
- Practice matching workload patterns to instance families
- Be able to justify purpose-built database selection for a given access pattern
- Understand read replica promotion, cross-Region read replicas, and Aurora Global Database reader endpoints


## What to expect

This task statement aligns with the Performance Efficiency pillar of the AWS Well-Architected Framework. Though this task statement focuses mainly on the Selection focus area, it also includes aspects of Review, Monitoring, and Trade-offs.  Exam questions for this task statement require you to identify a solution that meets performance requirements, often while balancing cost and maintenance needs. You should also have a good understanding of the Cost Optimization and Operational Excellence pillars. Questions on the exam will test your ability to design an elastic architecture based on business objectives. You should be able to identify efficient scaling strategies that scale out to meet performance requirements. These strategies should also scale in as demand decreases in order to meet cost requirements. Evaluating scaling strategies requires you to know the scaling controls and configurations for compute, storage, database, and networking services. You should also know the capabilities of serverless options in these categories, and understand trade-offs between cost and decreased maintenance overhead. You should understand how to design a right-sizing strategy to ensure you meet performance requirements and save money.  For example, you should be familiar with Amazon EC2 instance families and their use cases, as well as right-sizing your storage AND database instances. Then you need to consider the other components. For EC2, what kind of volume storage has the right speed and data resiliency features to support the workload and how large of a volume do you need? If you are running High Performance Compute workloads, do you also need an Elastic Fabric Adaptor? In addition, make sure you understand how to configure monitoring tools to baseline the performance of your new solutions. Monitoring data verifies whether you have right-sized your components and supports ongoing usage monitoring to identify when resources become over-utilized or under-utilized.  Exam questions for this task statement also test your knowledge of applying design patterns, such as replicas, buffering, and caching, to meet performance objectives.  You might see options on the exam that describe using a replica design pattern to meet performance requirements. This pattern can put resources closer to where they are consumed to reduce latency such as creating a DynamoDB global table with replicas in Regions where demand is greatest.  A replica pattern can also create additional resources to handle demand and distribute load. For example,  You can use an Amazon RDS read replica to off-load some read traffic from the primary instance.  When evaluating a solution that uses the replica pattern, consider which services include a replication feature. You should also consider any additional cost and management these resources require. You should understand the buffering design pattern. For example, when you use a message queue as a buffer between applications, you need to identify when to use a managed messaging service, such as Amazon MQ for Apache ActiveMQ and RabbitMQ, or Amazon Simple Queue Service.  It is important to know the different queue types, their respective throughput and limits, options for message ordering, message payload size, message batching, and supported polling frequencies.You(opens in a new tab) should also be familiar with using a caching design pattern to meet performance requirements. For example, you should be able to recognize when to use CloudFront to reduce latency by caching frequently requested items closer to the user.  Your knowledge of CloudFront should include understanding how to set the correct time to live (TTL) on cached objects, how to configure CloudFront to automatically compress certain types of objects, and how to use CloudFront with Amazon Route 53 latency-based routing. When selecting a solution on the exam, you might need to determine when design patterns meet the stated requirements. For example, to optimize the response time of your databases, you can use database caching, read replicas, or a combination of the two. A database cache can only return results for queries that have already occurred, so it will only improve performance when traffic consists of many duplicate queries. A database cache also stores query results, which might make caching the wrong choice when working with frequently changing data. However, read replicas still need time to perform the query, which make them less performant than a cache. Multiple read replicas could also be required to match the number of requests per second that a single in-memory cache node can deliver. Requiring multiple read replicas can be more costly and creates more resources to manage. Finally, questions related to this task statement will test your ability to select services that support a variety of access patterns. For example, based on a given access pattern, you should be able to select the right database type. Where possible, identify the correct purpose-built database service, such as DynamoDB for key-value databases or Amazon Neptune for graph databases.   Ok, that’s all for task statement 5, Design a solution to meet performance objectives.