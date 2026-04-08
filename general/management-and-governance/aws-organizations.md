Overview
Centralized account management and organizational control across multiple AWS accounts.

Tasks
- Create organizational units, apply service control policies (SCPs), and manage accounts.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Manual account structure | Simpler | Hard to scale

Limitations
- SCPs can be complex; careful planning required to avoid lockouts.

Price info
- Free; underlying services billed.

Network & multi-region
- Global control plane for org-level policies and account management.

Popular use cases
- Multi-account governance, centralized billing, and control plane policies.

When Not to Use
- Single-account projects; organizations add complexity.

DR strategy
- Backup org structure and SCPs as IaC; document account recovery processes.
