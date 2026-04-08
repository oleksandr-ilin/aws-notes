# Reliable and Resilient Architecture

## Knowledge of:

- Recovery time objectives (RTOs) and recovery point objectives (RPOs)
- Disaster recovery strategies (for example, using AWS Elastic Disaster Recovery, pilot light, warm standby, and multi-site)
- Data backup and restoration

## Skills in:

- Designing disaster recovery solutions based on RTO and RPO requirements
- Implementing architectures to automatically recover from failure
- Developing the optimal architecture by considering scale-up and scale-out options
- Designing an effective backup and restoration strategy


## should be comfortable with

- balancing traffic between Regions using the different Amazon Route 53 routing policies, and Amazon Elastic Load Balancing with Amazon EC2 and Auto Scaling groups
- how to reduce your workload’s latency by using services like Amazon CloudFront to cache content closer to the end user, and AWS Global Accelerator to direct traffic to the closest edge location


## disaster recovery strategies

Disaster recovery focuses on one-time recovery objectives in response to natural disasters, large-scale technical failures, and so on. 

### Ensure you know 

- which services to use to support your recovery point objective (or RPO), 
- which is the acceptable amount of data loss measured in time, and recovery time objective (or RTO), 
- which is the maximum amount of downtime tolerated due to a failure

### familiar with

the following disaster recovery strategies and how to replicate your infrastructure across Regions to provide a resilient architecture

- Multi-site (active/active)
- warm standby
- pilot light.


### based on given requirements

- RTO, 
- RPO
- effort
- cost


## Strategy for replicating your workload components and backups

- `AWS Elastic Disaster Recovery` to replicate your compute and volume storage resources to a disaster recovery Region, 
- `AWS Backups` to centrally manage and automate your backup processes across AWS accounts