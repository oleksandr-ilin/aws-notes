Overview
Resource configuration tracking and compliance auditing for AWS resources.

Tasks
- Enable recording, create rules, and evaluate resource compliance over time.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom inventory tools | Tailored data | More maintenance

Limitations
- Recording all resources can be noisy and expensive; rule coverage varies.

Price info
- Per-configuration-item and conformance packs fees may apply.

Network & multi-region
- Regional; use aggregator accounts for centralized compliance views.

Popular use cases
- Drift detection, compliance audits, and configuration history.

When Not to Use
- Very small accounts where manual tracking suffices.

DR strategy
- Archive configuration snapshots and remediation runbooks in version control.
