# Networking


## Knowledge of:

- AWS Global Infrastructure
- AWS networking concepts (for example, Amazon Virtual Private Cloud [Amazon VPC], AWS Direct Connect, AWS VPN, transitive routing, AWS container services)
- Hybrid DNS concepts (for example, Amazon Route 53 Resolver, on-premises DNS integration)
- Network segmentation (for example, subnetting, IP addressing, connectivity among VPCs)
- Network traffic monitoring

## Skills in:

- Evaluating connectivity options for multiple VPCs
- Evaluating connectivity options for on-premises, co-location, and cloud integration
- Selecting AWS Regions and Availability Zones based on network and latency requirements
- Troubleshooting traffic flows by using AWS tools
- Using service endpoints for service integrations

    Using service endpoints for service integrations

## Types

- **Amazon Virtual Private Cloud (Amazon VPC)**
- **AWS Direct Connect**
- **AWS Client VPN**
- **AWS Site-to-Site VPN**
- **AWS Transit Gateway**
- **AWS PrivateLink**


## What to understand

you should understand how to separate your network segments based on given requirements. For example, you should know how to use segmentation to: 

- Slow down a cyber attack by creating multiple layers of defense
- Restrict access to data and network resources by creating permissions boundaries
- make your infrastructure more resilient by limiting the scope of an outage.

Understand how to handle DNS routing with 
- `Amazon Route 53`
- `Route 53 Resolver`. 

You should also understand how to manage 
- cross-Region
- hybrid connections with transit gateway. 

Lastly, know how to use service endpoints and AWS PrivateLink to secure traffic by keeping it on the AWS network.


## You should know

- how to aggregate usage data from `Amazon CloudWatch`, `AWS CloudTrail`, and `VPC flow logs` across accounts. 
- how to use `Amazon VPC Traffic Mirroring` to replicate network traffic for content inspection, threat monitoring, and troubleshooting. 
- how to configure `Amazon Inspector` to scan for security vulnerabilities across accounts in a multi-account structure.

## You should be comfortable with

- intra-system connectivity
- inter-system connectivity
- public IP address management
- private IP address management
- domain name resolution.

### Example 1
What could you implement to meet requirements for applications to access updates and patches from the internet through a specific third-party using their URL and denying all other outbound connections? 

- Security groups, 
- network ACLs, 
- NAT Gateway, 
- forward web proxy server in your VPC to manage outbound access using URL-based rules. 