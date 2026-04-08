Overview
Declarative IaC for provisioning AWS resources with templates and stacks.

Tasks
- Author templates, create/update stacks, manage drift and change sets.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Terraform | Multi-cloud & mature ecosystem | External state management |
| Pulumi | Code-first IaC | Learning curve for SDKs |

Limitations
- Stack limits, template complexity, and cross-stack dependencies.

Price info
- No service charge; pay for provisioned resources.

Network & multi-region
- Regional stacks but some resources are global; design cross-region provisioning carefully.

Popular use cases
- Repeatable infra provisioning, baseline account setups, and automated deployments.

When Not to Use
- If you need multi-cloud abstractions—consider Terraform or Pulumi.

DR strategy
- Store templates in version control and test stack recovery procedures in a sandbox account.
