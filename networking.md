# Networking

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