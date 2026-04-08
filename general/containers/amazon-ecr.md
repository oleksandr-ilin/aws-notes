# Amazon ECR

## Service Overview
Amazon Elastic Container Registry (ECR) is a fully managed Docker container registry that stores, manages, and deploys container images.

## Tasks this service can solve
- Store and version container images securely
- Integration with CI/CD pipelines for image promotion
- Host private or public repositories for deployment

## Alternatives
| Name | Type | Short description | Pro vs ECR | Cons vs ECR | Price comparison |
|---|---|---:|---|---|---|
| Docker Hub | Commercial/OSS | Public and private container registry | Familiar ecosystem, large community | Rate limits and less integrated with AWS | Often comparable; ECR can be cheaper for AWS data transfer
| GitHub Container Registry | Commercial | Container registry integrated with GitHub | Tight CI/CD integration with GitHub Actions | External to AWS for some orgs | Comparable; choose based on platform
| Self-hosted registry (Harbor) | OSS | Registry with enterprise features | More control and customization | Operational overhead | Higher ops cost; ECR managed reduces ops

## Limitations
- Regional repositories by default; cross-region replication must be configured.

## Price info
- Charged for storage (per-GB) and data transfer for image pulls across regions or to the internet.

## Network & Multi-region considerations
- Use cross-region replication for resilience and faster pulls in other regions. Configure VPC endpoints (ECR API and ECR DKR) for private network access.

## When Not to Use This Service
- If you require a registry tightly integrated with non-AWS CI/CD or advanced on-prem policies, a self-hosted registry or third-party commercial registry may be preferable.

## DR strategy
- Enable cross-region replication and retain image tags and immutable manifests. Back up important images and repository configurations and store policies in source control.

## Popular use cases
- Storing images for ECS/EKS/Fargate deployments, CI/CD artifact hosting, and multi-account shared registries.
