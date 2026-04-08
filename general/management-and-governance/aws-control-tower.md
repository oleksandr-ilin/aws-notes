Overview
Opinionated landing zone automation for multi-account AWS environments with guardrails.

Tasks
- Set up Control Tower landing zone, configure organizational units, and apply guardrails.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom landing zone (Terraform/IaC) | Full control | More build effort

Limitations
- Opinionated templates may not fit all enterprise needs; customization is limited.

Price info
- No separate charge; underlying services billed.

Network & multi-region
- Multi-region support via underlying services; coordinate account baselines.

Popular use cases
- Rapid multi-account setup with best-practice guardrails for governance.

When Not to Use
- Highly custom enterprise architectures requiring bespoke account baselines.

DR strategy
- Keep landing zone configuration in IaC and test account provisioning/recovery.
