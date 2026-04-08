# Cost optimization and visibility strategies

## Knowledge of:

- AWS cost and usage monitoring tools (for example, AWS Trusted Advisor, AWS Pricing Calculator, AWS Cost Explorer, AWS Budgets)
- AWS purchasing options (for example, Reserved Instances, Savings Plans, Spot Instances)
- AWS rightsizing visibility tools (for example, AWS Compute Optimizer, Amazon Simple Storage Service [Amazon S3] Storage Lens)

## Skills in:

- Monitoring cost and usage with AWS tools
- Developing an effective tagging strategy that maps costs to business units
- Understanding how purchasing options affect cost and performance


## Understand 

choose between VPC Peering and transit gateway when creating multi-VPC and multi-Region environments

With transit gateway, you are charged by the hour for the number of connections and the amount of data transferred over these connections. However, it simplifies management and reduces the number of VPN connections required. The lower operational overhead of transit gateway might yield benefits and cost savings that can outweigh its additional cost of data processing

when to choose Direct Connect or Site-to-Site VPN for a hybrid infrastructure. Direct Connect has lower data transfer costs, but requires you to set up a cross-connect in a Direct Connect location, which takes time to set up and incurs additional cost. Alternatively, Site-to-Site VPN can be active in a matter of minutes, but charges a higher data transfer rate. 

where and when to use VPC endpoints to control costs by routing traffic within AWS. 

How to reduce egress traffic over the Internet, THROUGH:
- a NAT gateway,
- a VPN connection, 
- Direct Connect.


how to reduce costs using CloudFront to cache data at edge locations, compared to routing every request back to a VPC

## how to manage costs and billing for a multi-account structure.

how to implement AWS Organizations and consolidated billing, to combine the charges for all AWS accounts

realize cost savings by combining usage across all accounts to share 
- volume pricing discounts,
- Reserved Instance discounts, 
- Savings Plans

## how to choose and manage the appropriate purchase options for all workloads

how to select the right Savings Plan for your compute resources based on current usage and whether you expect that usage to  change over time.

developing an effective tagging strategy that maps
- resource costs to workloads, 
- individual organizations, 
- product owners 
to track AWS usage and costs.

## understand
- how to use `AWS Compute Optimizer` to right-size a multi-account, multi-Region, workload.
- how `AWS Cost Explorer` helps you visualize and manage your AWS costs and usage across AWS accounts at daily or monthly granularity
- how to use `Amazon Athena` and `Amazon QuickSight` to create custom dashboards and provide deeper analysis of cost and usage throughout an organization
- how to use `AWS Budgets` in a multi-account environment to track your resource usage and cost
- configuring `AWS Budget Actions` to automatically respond to cost and usage statuses in your accounts.