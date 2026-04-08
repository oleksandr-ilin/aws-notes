# AWS Cost Anomaly Detection

## Service Overview
Cost Anomaly Detection (part of Cost Management) uses machine learning to detect unusual spend patterns and alert teams.

## Tasks this service can solve
- Detect and alert on sudden unexpected cost spikes
- Automate notifications to teams for remediation

## Alternatives
| Name | Type | Short description | Pro vs Anomaly Detection | Cons vs Anomaly Detection | Price comparison |
|---|---|---:|---|---|---|
| Third-party monitoring (Cloudability, CloudHealth) | Commercial | Broader FinOps monitoring and alerts | More features and multi-cloud support | Cost and integration | Higher licensing costs but more features |
| Custom threshold alerts on Cost Explorer | AWS | Manual thresholds and alerts | Simpler and no ML | Less sensitive to true anomalies | Potentially cheaper but less accurate

## Limitations
- ML models need time to learn normal patterns; may produce false positives initially.

## Price info
- Often included within Cost Management features; check current AWS pricing for anomaly detection alerts.

## Network & Multi-region considerations
- Works across consolidated billing; ensure alerting targets and SNS topics are configured for global teams.

## When Not to Use This Service
- If you require deep multi-cloud anomaly detection, consider commercial FinOps providers instead.

## DR strategy
- Keep alert configurations and notification runbooks in source control; ensure on-call rotations and remediation playbooks are replicated across teams and regions.

## Popular use cases
- Detecting runaway development environments, compromised credentials causing resource spam, or billing surprises after deployments.
