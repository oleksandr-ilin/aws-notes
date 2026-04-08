Overview
Curated catalog of approved products and stacks for self-service provisioning by teams.

Tasks
- Define products, portfolios, and constraints; grant access to end users.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Terraform modules + CI/CD | Flexible & code-driven | Requires governance

Limitations
- Product lifecycle management and updates require careful planning.

Price info
- No separate charge; underlying resource costs apply.

Network & multi-region
- Regional; products may provision resources across regions depending on templates.

Popular use cases
- Self-service infra for dev teams with guardrails.

When Not to Use
- Very ad-hoc provisioning where centralized products slow experimentation.

DR strategy
- Version product templates and maintain rollback artifacts in version control.
