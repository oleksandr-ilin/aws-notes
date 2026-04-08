# Amazon EKS Distro

## Service Overview
Amazon EKS Distro (EKS-D) is the open-source distribution of Kubernetes used by EKS, enabling users to run identical Kubernetes builds anywhere.

## Tasks this service can solve
- Build Kubernetes clusters with the same tested binaries as EKS
- Ensure compatibility between on-prem and EKS cloud clusters

## Alternatives
| Name | Type | Short description | Pro vs EKS-D | Cons vs EKS-D | Price comparison |
|---|---|---:|---|---|---|
| Vanilla upstream Kubernetes | OSS | Community Kubernetes releases | More rapid upstream features | Less tested across AWS integrations | Free but ops cost applies
| Commercial distros (Rancher, OpenShift) | Commercial | Supported Kubernetes distributions | Enterprise tooling & support | Licensing cost | Higher due to support/licenses

## Limitations
- EKS-D provides binaries but not a managed control plane; users still operate clusters.

## Price info
- EKS-D itself is free; operating clusters has infra and ops costs.

## Network & Multi-region considerations
- Use EKS-D to ensure consistent versions across regions and environments; manage networking and CNI choices per environment.

## When Not to Use This Service
- Not needed if you use managed EKS control plane and prefer not to manage cluster lifecycle yourself.

## DR strategy
- Keep cluster install manifests, bootstrapping scripts, and versioned binaries in artifact storage; test cluster recreation and restore procedures.

## Popular use cases
- Consistent k8s builds for air-gapped environments and reproducible cluster builds.
