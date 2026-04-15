# Business Continuity

## Knowledge of:

- AWS global infrastructure
- AWS networking concepts (for example, Route 53, routing methods)
- RTOs and RPOs
- Disaster recovery scenarios (for example, backup and restore, pilot light, warm standby, multi-site)
- Disaster recovery solutions on AWS

## Skills in:

- Configuring disaster recovery solutions
- Configuring data and database replication
- Performing disaster recovery testing
- Architecting a backup solution that is automated, is cost-effective, and supports business continuity across multiple Availability Zones and/or AWS Regions
- Designing an architecture that provides application and infrastructure availability in the event of a disruption
- Leveraging processes and components for centralized monitoring to proactively recover from system failures


## What to understand

- How each DR strategy (backup/restore, pilot light, warm standby, multi-site) maps to RTO/RPO targets and cost
- Aurora Global Database and DynamoDB Global Tables for cross-Region data replication
- AWS Elastic Disaster Recovery (DRS) for continuous block-level replication
- Automated backup with AWS Backup across services and accounts
- Route 53 health checks, failover routing, and DNS TTL considerations
- Centralized monitoring with CloudWatch cross-account dashboards and composite alarms
- AWS Fault Injection Simulator (FIS) for DR testing and chaos engineering
- Application-level failover vs infrastructure-level failover
- S3 Cross-Region Replication for data continuity
- Multi-AZ vs Multi-Region availability tradeoffs


## Services to know

- `Amazon Route 53` — DNS routing, health checks, failover
- `Aurora Global Database` — cross-Region database replication
- `DynamoDB Global Tables` — multi-Region NoSQL replication
- `AWS Elastic Disaster Recovery` — continuous server replication
- `AWS Backup` — centralized backup management
- `Amazon CloudWatch` — monitoring, alarms, composite alarms
- `AWS Fault Injection Simulator` — controlled failure injection
- `Amazon S3` — Cross-Region Replication (CRR)
- `AWS CloudFormation` — infrastructure recovery via IaC


## What to expect

he second task statement from Domain 2 Design a solution to ensure business continuity. This refers specifically to an entire system’s ability to recover from a disaster.  For an overview of important concepts related to business continuity, review the Well-Architected Framework’s Reliability pillar and focus on the failure management and its best practices.  Questions in Domain 2 often present requirements and ask you to select a solution that meets those requirements, such as recovery time objective or RTO, which is the maximum amount of tolerable downtime due to a failure, and recovery point objective or RPO, which is the acceptable amount of data loss measured in time. You should know how to design an architecture that provides application and infrastructure availability in the event of a disruption. For example, you should know how to use multiple Availability Zones or Regions for fault isolation to limit the impact of an outage, and to support failover operations in the event of a failure in one Availability Zone or Region. You should also be able to identify the appropriate disaster recovery scenario based on requirements, such as RTO, RPO, and cost. Multi-site which is active-active, warm standby, pilot light, and backup and restore recovery strategies each have trade-offs between costs and RTO/RPO. For example, the multi-site/active-active strategy has a fast recovery time, but has a higher cost. Warm standby has a slightly slower recovery time, but is more cost effective. Pilot light is even more cost effective, but requires a longer recovery time. And backup and restore is the most cost effective, but requires a tolerance for higher RPO/RTO.   What if your company runs a data warehouse on Redshift and it must remain available even if an entire Region goes down. And you need to provide a RPO of 24 hours and RTO of 1 hour. What solution would you implement? Would you configure automatic snapshots and use cross-region snapshot to copy and replicate the production cluster to a disaster recovery region? Or change the DNS endpoint or use cross-Region replication? Dive deeper into snapshots and how you can enable automatic copies to another Region. The exam also tests your ability to architect a backup solution that is automated, cost effective, and supports availability across multiple Regions or Availability Zones. Your solution should balance the required RPO/RTO, data resiliency, and costs of data storage, and data transfer. In addition, know how to configure a disaster recovery, or DR, solution to quickly reestablish access to your systems after an outage. A disaster recovery solution should be able to: 

- replicate production resources for a failover environment,
- integrate with monitoring tools to detect outages, 
- redirect traffic to the failover environment, 
- and support disaster recovery testing.

We mentioned this in an earlier lesson, but dive deeper into AWS Elastic Disaster Recovery which performs ongoing replication of the servers and block storage in a source location, to a recovery Region in AWS. You can also perform testing against the recovery Region to verify that the replication is performing properly. You also want to be familiar with the different AWS database replication capabilities. For example, with Amazon RDS you can replicate your database to additional Availability Zones.  

- You can create a multi-AZ instance, 
- a multi-AZ cluster, 
- or read replicas, 

but which type you can create is limited by the database engine you are using. Some of these database copies can accept read traffic; others cannot. In the event of an outage in a primary database instance, copies can be promoted to primary, but read replicas must be restarted first, which takes longer. This requires a longer RTO.  Another topic in this task statement includes data replication services. For example, you should be familiar with AWS DataSync and AWS Storage Gateway and understand when one or the other meets the stated requirements. Are you being asked to replicate files, volumes, or tape backups? How does the frequency of each service’s scheduled replication actions affect RPO?  You will also need to identify the correct traffic routing configurations for a disaster recovery solution. This includes the correct Amazon Route 53 settings for DNS routing, such as configuring health checks to recognize outages, choosing the correct DNS routing method that will meet failover requirements, and setting the appropriate time to live (TTL) on DNS records that need to be changed rapidly in the event of a failover.  It also includes network components that facilitate failover such as load balancers. You should familiarize yourself with how using static or dynamic routing impacts the configurations required to reroute traffic during an event. Finally, when designing a disaster recovery solution, use processes and components for centralized monitoring to proactively recover from system failures. For example, you should be able to identify the correct CloudWatch configurations for monitoring system health and CloudWatch alarms, which can initiate an automated response. Consider how you could use a service like EventBridge to automate a response to an outage using AWS Lambda, or Amazon Simple Notification Service to alert the appropriate team of the failure.  OK, that’s all for Task statement 2, Design a solution to ensure business continuity. Remember, if you are unfamiliar with any of the services that were mentioned in this topic, make sure to note them for additional study.