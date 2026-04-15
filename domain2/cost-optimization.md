# Task Statement 6: Determine a Cost Optimization Strategy

## Overview

This task statement focuses on identifying cost-effective architectures through pricing model selection, storage tiering, data transfer cost modeling, right-sizing, and expenditure controls.

## Key Knowledge Areas

### Cost and Usage Monitoring Tools
- **Cost Explorer** — visualize spending trends, filter by service/tag/account, forecast future costs, identify RI/SP coverage gaps
- **AWS Budgets** — dollar/usage/RI/SP budgets with threshold alerts and auto-actions (apply SCP, stop EC2, notify)
- **Trusted Advisor** — cost optimization checks (idle RDS, underutilized EC2, unassociated EIPs, idle load balancers)
- **Pricing Calculator** — estimate costs before deployment, compare configurations
- **Cost Anomaly Detection** — ML-based anomaly detection on spending with root cause analysis
- **CUR (Cost and Usage Report)** — most granular billing data, Athena/QuickSight integration for custom analytics

### Pricing Models
- **On-Demand** — pay per hour/second, no commitment, highest unit cost
- **Reserved Instances** — 1-year or 3-year, Standard (exchange same family) vs. Convertible (exchange any), All/Partial/No Upfront
- **Savings Plans** — Compute SP (any Region, family, OS, tenancy), EC2 Instance SP (specific family + Region), SageMaker SP
- **Spot Instances** — up to 90% discount, 2-minute interruption notice, Spot Fleet, capacity-optimized allocation
- **Dedicated Hosts** — physical server, BYOL (bring your own license), per-host billing

### Storage Tiering
- **S3 Standard** → **S3 Intelligent-Tiering** (auto-moves objects) → **S3 Standard-IA** (infrequent, per-GB retrieval) → **S3 One Zone-IA** → **S3 Glacier Instant Retrieval** → **S3 Glacier Flexible Retrieval** → **S3 Glacier Deep Archive**
- **EBS** — gp3 vs. gp2 migration (gp3 is 20% cheaper at baseline), io2 vs. io1, st1/sc1 for sequential workloads
- **EFS** — Standard vs. Infrequent Access (lifecycle policies), One Zone for non-critical data

### Data Transfer Costs
- **Inbound** — free (into AWS from internet)
- **Same AZ** — free (private IP), charged (public/Elastic IP)
- **Cross-AZ** — $0.01/GB each direction (within Region)
- **Cross-Region** — $0.02/GB (varies by Region pair)
- **Internet egress** — tiered ($0.09/GB first 10 TB, decreasing)
- **VPC endpoints** — Gateway endpoints (S3, DynamoDB) free data transfer vs. Interface endpoints ($0.01/GB processed)
- **CloudFront** — lower egress rates than direct internet, Regional edge caches reduce origin fetches
- **Direct Connect** — lower data transfer rates than internet egress

### Managed Service Offerings
- Replace self-managed infrastructure with managed services to reduce operational overhead
- **Aurora Serverless v2** vs. provisioned Aurora — pay per ACU for variable workloads
- **Lambda** vs. always-on EC2 — cost-effective for sporadic/event-driven workloads
- **Fargate** vs. EC2 for ECS/EKS — no cluster management, per-vCPU-second billing
- **DynamoDB on-demand** vs. provisioned — on-demand for unpredictable traffic, provisioned + auto scaling for steady

## Core Skills

1. **Right-sizing infrastructure** — Compute Optimizer recommendations, CloudWatch CPU/memory utilization analysis, downsizing over-provisioned instances
2. **Pricing model selection** — RI/SP for steady-state, Spot for fault-tolerant, on-demand for spiky/short-lived
3. **Data transfer cost modeling** — minimize cross-AZ/cross-Region transfers, use VPC Gateway endpoints, consolidate with CloudFront
4. **Expenditure and usage awareness** — tagging strategy, AWS Organizations consolidated billing, Budgets + alerts + auto-remediation, SCP-based guardrails

## Study Approach

- Memorize data transfer cost tiers (free inbound, cross-AZ cost, cross-Region cost, egress tiers)
- Understand RI vs. Savings Plan flexibility trade-offs
- Know S3 storage class characteristics and lifecycle transition rules
- Practice calculating total cost of ownership (compute + storage + data transfer + licensing)
- Understand when managed services are cheaper despite higher unit cost (reduced ops overhead)


## What to expect

This task statement aligns to the Cost Optimization pillar of the AWS Well-Architected Framework and focuses on the following focus areas: Expenditure and Usage Awareness, and Cost-Effective Resources.  Balancing cost with performance and reliability requirements is important. Dive deeper into the Performance Efficiency pillar and the Reliability pillar. Exam questions for this task statement test your familiarity with costs related to AWS services, and how you include cost controls and monitoring into new solutions. Let’s start with costs related to AWS services. Based on requirements, you should be able to identify opportunities to select and right-size infrastructure for cost-effective resources. You should also be able to identify the best pricing models, or combination of pricing models for a solution. If your compute and database resource usage requirements are not yet known or intermittent, you can consider on-demand instances or spot Instances. When expected usage is known and expected to persist over a one-year or three-year period, consider Savings Plans for compute services. You can choose the EC2 Instance Savings Plan, the Compute Savings Plan, which include Amazon EC2 instances, AWS Lambda, and AWS Fargate, or the Amazon SageMaker Savings Plan for SageMaker Machine Learning instances. Alternatively, for predictable and persistent database usage, you can consider Reserved Instances, which are available for Amazon RDS, Amazon Redshift, Amazon ElastiCache, and Amazon OpenSearch Service. You also need to be familiar with storage pricing. You might see questions that test your knowledge of Amazon S3 costs. This can include selecting the most cost-effective storage class to meet storage capacity and access pattern requirements. It can also include identifying the correct lifecycle policy that moves objects between storage classes to maintain cost optimization as usage changes. You should also be familiar with how data retrieval, versioning, and replication requirements impact overall costs. You might also need to know costs associated with Amazon EBS. If a solution terminates EC2 instances, does it also delete the EBS volume? If a solution creates EBS snapshots, does it also archive or delete older EBS snapshots? You might need to determine when a managed service can deliver more cost savings than a self-managed option. For example, using Cognito to authenticate users instead of building your own authentication tool can save development, hosting, and maintenance costs. In addition to the cost of the components, make sure you understand data transfer modeling and reducing costs related to data transfer. Costs for data transfer are based on the source, the destination, and the amount of traffic.  Familiarize yourself with the costs of transferring data between AWS and on-premises data centers. This can be any combination of the following: the hourly charge for an Site-to-Site VPN connection and the cost of transferring data, the hourly charge of an Direct Connect port and the cost of transferring data. and the number of attachments on a Transit Gateway and the cost of transferring data.Next(opens in a new tab), be familiar with the cost of data transfer within your workloads. To avoid the costs of routing traffic over the internet, you can connect to AWS services and workloads in other AWS Regions from within AWS by using VPC endpoints.  Data transfer within an Availability Zone is free, but may not be free across Availability Zones. For example, there is no charge for data synchronization between two Amazon RDS DB instances in separate Availability Zones. However, ingress and egress data transfer between two Amazon EC2 instances across these same Availability Zones does incur charges. Finally, you should understand how to prepare your new solutions for ongoing cost management. This includes implementing cost controls and configuring cost and usage monitoring tools. Some tools for cost control that might be on the exam include resource tagging strategies, and IAM policies that restrict the creation of new resources.  When evaluating options for cost and usage visibility on the exam, be familiar with the following tools and services. You can use AWS Pricing Calculator to estimate the price of an AWS solution before building it out. You can use Cost Explorer dive into aggregate cost data by services and so on. Trusted Advisor runs checks on your infrastructure and makes recommendations, including cost optimizations. You can configure CloudWatch alarms to react to changes in a Trusted Advisor check, such as an updated resource or a service quota that is reached. 