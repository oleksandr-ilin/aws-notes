# Amazon EKS Anywhere

## Service Overview
Amazon EKS Anywhere helps create and operate Kubernetes clusters on-premises and at the edge that are compatible with EKS.

## Tasks this service can solve
- Run Kubernetes clusters on-prem with EKS-compatible tooling
- Support air-gapped or offline cluster deployments

## Alternatives
| Name | Type | Short description | Pro vs EKS Anywhere | Cons vs EKS Anywhere | Price comparison |
|---|---|---:|---|---|---|
| Self-hosted k8s | OSS | Custom k8s installations | Full control | More ops & tooling | Higher ops cost; EKS Anywhere reduces some operational burden
| OpenShift | Commercial | Enterprise Kubernetes distro | Enterprise features and support | Licensing and complexity | Higher total cost typically

## Limitations
- Operational overhead for on-prem clusters remains; not a fully managed control plane.

## Price info
- Licensing/management fees may apply; underlying on-prem infra costs remain.

## Network & Multi-region considerations
- Manage secure connectivity and configuration drift between cloud EKS and on-prem clusters; use GitOps to keep manifests synchronized.

## When Not to Use This Service
- Not a fit if you prefer fully managed cloud control planes with minimal on-prem ops; prefer cloud EKS for simpler operations.

## DR strategy
- Maintain backup/restore procedures for cluster state and ETCD, and keep manifests and cluster configs in source control for rapid rebuild.

## Popular use cases
- Edge and regulated workloads, offline environments, and migrations requiring identical k8s APIs on-prem.
