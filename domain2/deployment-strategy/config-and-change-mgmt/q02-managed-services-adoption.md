# Q02: Managed Services Adoption to Reduce Operational Overhead

## Question

A startup is building a web application. The development team has 3 engineers who spend 40% of their time managing infrastructure: provisioning EC2 instances, configuring NGINX, managing database backups, and applying OS patches. The application is a Node.js REST API with a PostgreSQL database and a React frontend. Requirements:
- Reduce operational overhead so developers focus on code, not infrastructure
- Support blue/green deployments for the API with automated rollback
- Database must have automated backups, point-in-time recovery, and multi-AZ failover
- Frontend must be served globally with low latency
- CI/CD pipeline must build, test, and deploy automatically on code push

Which architecture should the solutions architect recommend?

## Options

- **A.** Deploy the Node.js API on AWS App Runner from a container image in ECR. Use Amazon Aurora Serverless v2 (PostgreSQL-compatible) for the database with automated backups and multi-AZ. Host the React frontend on AWS Amplify Hosting with built-in CI/CD from GitHub. Use Amplify's preview environments for pull request testing.
- **B.** Deploy the API on EC2 instances behind an ALB with CodeDeploy blue/green deployments. Use RDS PostgreSQL Multi-AZ with automated backups. Host the frontend on S3 + CloudFront. Build a CodePipeline to orchestrate CodeBuild and CodeDeploy.
- **C.** Deploy the API on EKS with Helm charts. Use Amazon Aurora with provisioned capacity and read replicas. Host the frontend on ECS Fargate behind CloudFront. Use ArgoCD for GitOps-based deployment.
- **D.** Deploy the API on AWS Lambda behind API Gateway. Use DynamoDB for the database. Host the frontend on S3 + CloudFront. Use SAM (Serverless Application Model) for deployments.

## Answers

### A. App Runner + Aurora Serverless v2 + Amplify Hosting — ✅ Correct

This maximally reduces operational overhead for a small team:
- **App Runner**: Zero infrastructure management — push a container image or connect a GitHub repo, and App Runner handles load balancing, TLS, scaling, and deployment. No EC2 instances, no NGINX, no OS patching. Supports automatic deployments on code push.
- **Aurora Serverless v2**: Auto-scales database capacity (ACUs) based on demand. Includes automated backups, point-in-time recovery (up to 35 days), and multi-AZ by default. No instance sizing or scaling decisions needed.
- **Amplify Hosting**: Built-in CI/CD for frontend — detects pushes to connected repo branches, builds the React app, and deploys to a global CDN. Preview environments for PRs are included. No S3 bucket configuration or CloudFront distribution setup needed.
- **Total infra management**: Near zero — all three components are fully managed.

### B. EC2 + CodeDeploy + RDS + S3/CloudFront — ❌ Incorrect

This works but doesn't reduce operational overhead:
- EC2 instances still require OS patching, NGINX configuration, instance sizing, and security group management.
- CodePipeline + CodeDeploy + CodeBuild is a multi-service CI/CD setup that requires configuration and maintenance.
- S3 + CloudFront requires bucket policies, distribution configuration, and cache invalidation management.
- RDS Multi-AZ is good but still requires instance type selection and scaling decisions.
- This is better than raw infrastructure but still leaves significant ops burden for a 3-person team.

### C. EKS + Helm + Aurora + ECS — ❌ Incorrect

This is the *most* operationally complex option:
- EKS requires Kubernetes expertise: cluster management, node groups, Helm chart authoring, networking (VPC CNI), and RBAC.
- ArgoCD adds another tool to deploy, configure, and maintain.
- Aurora provisioned requires capacity planning.
- ECS Fargate for a static React frontend is over-engineered — Fargate runs containers, not static files.
- A 3-person startup team would spend more time on infrastructure, not less.

### D. Lambda + DynamoDB + S3/CloudFront — ❌ Incorrect

This is serverless and low-ops, but the technology choices don't fit:
- **DynamoDB** is a NoSQL key-value store — the application uses **PostgreSQL**. Migrating from relational to NoSQL requires schema redesign, query rewriting, and potential data model changes. This is not a deployment strategy question — it's a replatforming effort.
- Lambda has cold-start latency and 15-minute execution limits which may not suit all API patterns.
- The question asks about managed services to reduce overhead, not about re-architecting the data layer.

## Recommendations

- **App Runner** is ideal for teams who want container-based deployment without learning ECS/EKS. It auto-scales, handles TLS, and provides a deployment pipeline from ECR or GitHub.
- **Aurora Serverless v2** eliminates the biggest RDS management task: capacity planning. It scales smoothly from small to large workloads within the same cluster.
- **Amplify Hosting** is purpose-built for frontend frameworks (React, Next.js, Vue). It includes CDN, CI/CD, preview environments, and custom domain management in one service.
- For the Node.js API, consider whether **App Runner** or **Lambda + API Gateway** fits better — Lambda is more cost-effective for spiky/infrequent traffic; App Runner is simpler for always-on APIs.
- Use **AWS Proton** or **Service Catalog** if the team grows and you need to standardize deployment templates for multiple services.

## Relevant Links

- [AWS App Runner](https://docs.aws.amazon.com/apprunner/latest/dg/what-is-apprunner.html)
- [Aurora Serverless v2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- [Amplify Hosting](https://docs.aws.amazon.com/amplify/latest/userguide/welcome.html)
- [AWS Proton](https://docs.aws.amazon.com/proton/latest/userguide/Welcome.html)
