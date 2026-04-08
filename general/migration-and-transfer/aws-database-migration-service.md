Overview
AWS Database Migration Service (DMS) helps migrate databases to AWS with minimal downtime.

Tasks
- Configure replication instances, endpoints, and replication tasks.
- Validate data and cut over once replication lag is acceptable.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Dump & restore | Simplicity | Longer downtime |
| Custom ETL | Flexibility | More work and testing

Limitations
- Complex schema or proprietary features may require manual conversion.

Price info
- Pay for replication instance-hours and storage; data transfer charges may apply.

Network & multi-region
- Supports cross-region migrations; plan network bandwidth and VPN/Direct Connect accordingly.

Popular use cases
- Homogeneous and heterogeneous DB migrations with minimal downtime.

When Not to Use
- If a full refactor/replatform is intended during migration.

DR strategy
- Use continuous replication for near-DR scenarios and validate failover procedures.
