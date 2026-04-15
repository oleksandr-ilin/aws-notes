# Q04: Container Networking with Amazon EKS VPC CNI

## Question

A company runs a microservices platform on Amazon EKS in a VPC with CIDR 10.0.0.0/16. The cluster hosts 200 pods per node across 50 nodes. The application requires:
- Every pod must have a routable VPC IP address so legacy services in other VPCs (connected via Transit Gateway) can reach pods directly by IP
- The current VPC CIDR is running out of available IP addresses
- Pods must be able to connect to an Amazon RDS instance in a private subnet using security group rules applied at the pod level
- Network policies must enforce east-west traffic control between namespaces

Which combination of networking features should the solutions architect configure?

## Options

- **A.** Use the Amazon VPC CNI plugin with custom networking to assign pod IPs from a secondary CIDR (100.64.0.0/16). Enable Security Groups for Pods to apply RDS security group rules at the pod level. Install a Calico network policy engine for namespace-level east-west traffic control.
- **B.** Switch to an overlay network CNI (e.g., Weave or Flannel) that uses its own IP range and does not consume VPC IPs. Use Kubernetes NetworkPolicy resources with the default CNI. Add NACLs on the RDS subnet to allow traffic from the pod overlay range.
- **C.** Use the Amazon VPC CNI plugin with prefix delegation to increase IP density per ENI. Assign pods dedicated ENIs via the trunk interface. Use only security groups on the worker node EC2 instances for all access control.
- **D.** Use AWS App Mesh to manage all service-to-service communication. Place pods in a private subnet with a NAT Gateway for outbound access. Use IAM roles for pods to authenticate to RDS instead of security group rules.

## Answers

### A. VPC CNI custom networking + Security Groups for Pods + Calico — ✅ Correct

This addresses every requirement:
- **VPC CNI custom networking**: Assigns pod ENIs from a secondary CIDR (e.g., 100.64.0.0/16) while keeping node primary ENIs on the original CIDR. Pods still get routable VPC IPs that are reachable from peered/TGW-connected VPCs — solving the IP exhaustion problem without losing direct pod addressability.
- **Security Groups for Pods**: Associates specific security groups with individual pods via `SecurityGroupPolicy` CRD. This lets the RDS security group's inbound rule reference the pod's security group — fine-grained, pod-level network access control that matches how non-containerised EC2 workloads integrate with RDS.
- **Calico network policy engine**: Provides Kubernetes NetworkPolicy enforcement (which VPC CNI alone does not enforce). Calico policies can restrict east-west traffic between namespaces using label selectors.

### B. Overlay network CNI — ❌ Incorrect

Overlay CNIs (Weave, Flannel) encapsulate pod traffic — pods do **not** get routable VPC IPs. Legacy services in other VPCs cannot reach pods directly by their overlay-internal IPs via Transit Gateway. NACLs cannot reference overlay ranges that don't exist in the VPC routing table. This fundamentally breaks the routable-IP requirement.

### C. VPC CNI with prefix delegation only — ❌ Partially addresses the problem

Prefix delegation increases IP density (assigning /28 prefixes to ENIs) but still consumes IPs from the existing CIDR — it doesn't solve exhaustion if the CIDR itself is too small. Applying security groups only at the node level means all pods on a node share the same security group rules — you cannot differentiate access to RDS per pod. This fails the pod-level security group requirement.

### D. App Mesh + NAT Gateway — ❌ Incorrect

App Mesh manages service mesh routing (Envoy sidecars) but does not assign VPC IPs to pods or provide security group integration. NAT Gateway handles outbound traffic and does not help with inbound direct-IP access from other VPCs. IAM roles for pods handle authentication (e.g., to AWS APIs), not network-level security group rules for RDS inbound access.

## Recommendations

- **VPC CNI custom networking** keeps pods routable on VPC IPs while using a secondary CIDR to extend address space — critical for brownfield VPCs running out of IPs.
- **Prefix delegation** (available alongside custom networking) further increases IP density: each ENI slot holds a /28 prefix (16 IPs) instead of a single IP.
- **Security Groups for Pods** requires nodes to use Nitro-based instance types (trunk ENI support). Node-level security groups still apply in addition to pod-level ones.
- **Calico** is the recommended network policy engine for EKS — Amazon VPC CNI supports it natively via the `aws-node` DaemonSet configuration.
- For IPv6 clusters, VPC CNI assigns each pod a globally unique IPv6 address, eliminating the CIDR exhaustion issue entirely.

## Relevant Links

- [Amazon VPC CNI Plugin](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [Custom Networking](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html)
- [Security Groups for Pods](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)
- [Prefix Delegation](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)
- [Calico Network Policy on EKS](https://docs.aws.amazon.com/eks/latest/userguide/calico.html)
