# Multi-account AWS environment

## Knowledge of:

- AWS Organizations and AWS Control Tower
- Multi-account event notifications
- AWS resource sharing across environments

## Skills in:

- Evaluating the most appropriate account structure for organizational requirements
- Recommending a strategy for central logging and event notifications
- Developing a multi-account governance model

## Understand how to use 

- AWS Organizations to create new accounts, 
- group them into Organizational Units (OUs) to simplify account management, 
- consolidated billing, 
- sharing resources, 
- apply service control policies to meet governance requirements.

## You should know

Q: if you do not want your Reserved Instance discount to be shared with other business units in your organization? 
A: you can turn off the reserved instance sharing on the management account for all member accounts.

how to use `AWS Resource Access Manager` to share resources across accounts.
AWS Organization -> Organizational Units -> Accounts



## AWS Control Tower
`AWS Control Tower` - This is a managed service that streamlines the process of setting up and governing a secure, multi-account, AWS environment.


### Know how
- create a `landing zone`, which serves as a starting point for an AWS multi-account environment
- the `Account Factory` feature is used to automate the provisioning of new accounts with pre-approved account configurations.
- use `Guardrails` enterprise-wide , to ensure governance rules for security, operations, and compliance are enforced
- use the `Dashboard` to continuously monitor your AWS accounts for compliance


## Understand

- the governance features in place that allow the accounts to perform their functions
- strategy for central logging and event notifications based on a given scenario
- how to configure services like CloudTrail, and Config, to provide alerts when a resource configuration is out of compliance with an organization’s policies and guidelines
- how these alerts can be sent to other services, like Amazon EventBridge, to provide automated responses such as remediation actions or notifications