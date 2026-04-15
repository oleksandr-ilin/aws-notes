# Deployment Strategy

## Knowledge of:

- Infrastructure as code (IaC) (for example, AWS CloudFormation)
- Continuous integration/continuous delivery (CI/CD)
- Change management processes
- Configuration management tools (for example, AWS Systems Manager)

## Skills in:

- Determining an application or upgrade path for new services and features
- Selecting services to develop deployment strategies and implement appropriate rollback mechanisms
- Adopting managed services as needed to reduce infrastructure provisioning and patching overhead
- Making advanced technologies accessible by delegating complex development and deployment tasks to AWS


## What to understand

- How to use CloudFormation StackSets for multi-account, multi-Region deployments
- Nested stacks vs cross-stack references for modular IaC
- CloudFormation drift detection and change sets for safe updates
- Blue/green, canary, and rolling deployment strategies using CodeDeploy
- CodePipeline for orchestrating CI/CD across build, test, deploy stages
- Rollback mechanisms: CloudFormation rollback triggers, CodeDeploy auto-rollback on alarm
- Systems Manager for patch management, run commands, and configuration compliance
- AWS Proton and Service Catalog for delegating deployment complexity to developers
- Elastic Beanstalk and App Runner as managed deployment targets
- Feature flags and gradual rollout strategies


## Services to know

- `AWS CloudFormation` — IaC provisioning and stack management
- `AWS CodePipeline` — CI/CD orchestration
- `AWS CodeBuild` — Managed build service
- `AWS CodeDeploy` — Deployment automation with rollback
- `AWS Systems Manager` — Config management, patching, automation
- `AWS Proton` — Managed delivery for containerized/serverless
- `AWS Service Catalog` — Approved product governance
- `AWS Elastic Beanstalk` — PaaS deployment
- `AWS App Runner` — Simplified container deployment


## What to expect

first task statement from Domain 2 which is to Design a deployment strategy to meet business requirements. For this task statement, you need to have a good understanding of change management processes. When designing a new solution, you need to consider how you will deliver that solution to your customers. You also need to consider how you will update that solution once it’s in place. 

When considering possible answers on the exam, remember the AWS Well-Architected Framework Reliability pillar and its best practices for change management. One best practice is to deploy changes with automation. Another best practice is to deploy using immutable infrastructure. 

For questions that relate to this task statement, you are expected to be able to select services to develop deployment strategies and implement appropriate rollback mechanisms. These strategies will cover deploying and updating both infrastructure and applications.

Let’s talk about the deployment of infrastructure. You should be able to identify opportunities to adopt managed services which reduce infrastructure provisioning and patching overhead. Dive deeper into AWS CloudFormation and ensure you know how to implement infrastructure as code. You can use CloudFormation to adhere to the AWS Well-Architected Framework best practices. You can automate the provisioning and configuration of an entire stack of new or updated infrastructure. You can also deploy a new stack of resources instead of trying to update you production infrastructure. And once deployment is complete, you route production traffic to the updated stack. 

For the exam, you should have a deep understanding of core concepts of CloudFormation, such as templates, stacks, and change sets. You should also be familiar with CloudFormation best practices such as:

- Creating reusable infrastructure components
- Using nested stacks and StackSets 
- Using change sets
- And sharing StackSets across multiple accounts and Regions

Also, dive deeper into the DeletionPolicy options and how to retain a resource when deleting a stack. And integration with the AWS Cloud Development Kit.

 
Next, let’s discuss overhead related to infrastructure patching. When designing new solutions, you need to plan for ongoing maintenance. 

You might be asked questions about AWS Systems Manager. 

- Make sure you understand how to use Systems Manager to deploy security patches and software updates. 
- You should also know how to incorporate Systems Manager into the deployment of your new solutions. This could include using  AWS Secrets Manager to handle passwords for service accounts or Amazon RDS passwords for automated database deployments. 
- You might also use Run Command to facilitate server deployment and post-deployment server configurations. 
- You should also know when to plan for the installation of the Systems Manager agent.

Along with testing your understanding of how to deploy infrastructure on AWS, the exam will also verify your knowledge of application deployments. You will need to be able to determine an application or upgrade path for new services and features. These deployments should follow the AWS Well-Architected best practice for change management to deploy changes with automation. A common methodology that follows this best practice is continuous integration/continuous delivery or CI/CD. To review some of the AWS services that can build a CI/CD pipeline, let’s consider a basic example. 

Here we have a release pipeline built using AWS CodePipeline. CodePipeline automates the build, test, and release processes. This pipeline uses an AWS CodeCommit code repository as the source. It then uses AWS CodeBuild to build the application and initiate automated testing. After the build is approved, AWS CodeDeploy deploys the updated application to the compute resource that hosts the application code–such as Amazon EC2, AWS Lambda, or containers on AWS Fargate. CodeDeploy also routes production traffic to the newly deployed application. 

Finally, for this task statement you should know how to make advanced technologies accessible by delegating complex development and deployment tasks to AWS. 

For example, Amazon SageMaker is an end-to-end machine learning service that provides algorithms and frameworks for building and training machine learning models. It also manages the deployment of new and updated models for use in your applications.  