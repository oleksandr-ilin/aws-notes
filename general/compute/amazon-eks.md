# Amazon EKS

## Service Overview
Amazon EKS is a managed Kubernetes control plane that simplifies running Kubernetes on AWS while allowing use of standard Kubernetes APIs and tooling.

## Tasks this service can solve
- Container orchestration for microservices
- Workload portability across environments
- Complex deployment patterns (canaries, blue/green)
- Stateful and stateless workloads with k8s operators

## Alternatives
| Name | Type | Short description | Pro vs EKS | Cons vs EKS | Price comparison |
|---|---|---:|---|---|---|
| Amazon ECS | AWS | AWS-native container orchestrator | Simpler to operate, tighter AWS integration | Less k8s ecosystem/portability | Often lower operational cost for simple apps |
| GKE | GCP | Managed Kubernetes on GCP | Strong autoscaling and multi-zone features | Different integrations | Comparable; GKE has its own control-plane pricing |
| Self-hosted k8s | OSS | Kubernetes on EC2 / on-prem | Full control | Significant ops overhead | Higher ops cost unless using managed control plane |
| OpenShift | Commercial | Kubernetes distro with platform features | Enterprise features and support | More opinionated and costly | Generally higher licensing/ops cost |

## Limitations
- Control plane fee per cluster plus costs for worker nodes; cluster ops and upgrade complexity.
- Complex networking plugins and CNI choices can affect scale and performance.

## Price info
- Control plane hourly fee plus worker node EC2 or Fargate charges. Additional costs for EBS, load balancers, and networking.

## Network & Multi-region considerations
- Clusters are regional. For high availability, distribute worker nodes across multiple AZs within a region.
- For multi-region resilience, operate multiple clusters per region and use global traffic managers (Route 53, service mesh federation) or CI/CD to sync deployments.
- Integrate with AWS VPC CNI, AWS Load Balancer Controller, and IAM for service accounts.

## Popular use cases
- Microservices architectures with Kubernetes-native tools
- Porting existing k8s workloads to AWS
- Running operator-driven databases and stateful services

## When Not to Use This Service
- Not recommended for very small teams unfamiliar with Kubernetes or simple container workloads where ECS/Fargate would reduce operational overhead.

## DR strategy
- Run worker nodes across multiple AZs and deploy separate clusters per region for region-level failover. Use declarative CI/CD to re-create clusters and GitOps to restore desired state. For stateful workloads, use cross-region replication (e.g., RDS Aurora Global DB, distributed storage) and backup strategies.
