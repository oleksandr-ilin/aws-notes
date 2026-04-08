# AWS CodePipeline

## Service Overview
CodePipeline is a managed CI/CD orchestration service that automates stages from source to deploy.

## Tasks this service can solve
- Orchestrate CI/CD pipelines across AWS services
- Enforce approvals, quality gates, and automated deploys

## Alternatives
| Name | Type | Short description | Pro vs CodePipeline | Cons vs CodePipeline | Price comparison |
|---|---|---:|---|---|---|
| GitHub Actions / GitLab CI | Commercial/OSS | CI/CD pipelines with broad ecosystem | Rich marketplace and actions | External dependency or management | Varies; CodePipeline billed per pipeline/action usage

## Limitations
- Integrations outside AWS may require custom actions; pipeline templates can be verbose.

## Price info
- Charged per active pipeline per month (check current pricing); underlying service costs apply.

## Network & Multi-region considerations
- Pipelines are regional; use cross-region artifact stores or duplicate pipelines for multi-region deployments.

## When Not to Use This Service
- If you already use a sophisticated CI/CD platform with many integrations, you may prefer to continue using it.

## DR strategy
- Store pipeline definitions as code (CloudFormation/CodePipeline YAML) and keep pipeline templates and artifact storage replication scripts for recovery.

## Popular use cases
- End-to-end AWS CI/CD for microservices and infrastructure pipelines.
