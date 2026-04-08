Overview
Ops tooling for patching, runbooks, inventory, Session Manager, and automation across fleets.

Tasks
- Configure inventory, runbooks, patch baselines, and use Session Manager for secure access.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Ansible / SSH | Familiar tooling | More network exposure & ops

Limitations
- Some features require agents; cross-account setup can be non-trivial.

Price info
- Some features free; certain advanced features may be billed.

Network & multi-region
- Regional; centralize ops via aggregator accounts where helpful.

Popular use cases
- Patching, automation, remote access, and operational runbooks.

When Not to Use
- Small fleets where manual SSH access is simpler.

DR strategy
- Store runbooks and automation documents in version control and test restore tasks.
