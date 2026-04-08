Overview
AWS Schema Conversion Tool (SCT) converts database schema and code between engines to reduce manual effort in migrations.

Tasks
- Analyze source schema, run assessments, and convert schema and procedural code.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Manual conversion | Full control | Time-consuming |
| Third-party conversion services | Expert assistance | Cost

Limitations
- Not all stored procedures or proprietary features convert automatically; manual adjustments often required.

Price info
- The tool itself is free; migration effort and testing drive cost.

Network & multi-region
- Runs from local/EC2 environments; conversion artifacts can be moved between regions.

Popular use cases
- Heterogeneous database migrations (e.g., Oracle → Aurora/Postgres).

When Not to Use
- Simple homogeneous migrations where no schema changes are needed.

DR strategy
- Use SCT reports to document schema drift and include in DR runbooks for restores.
