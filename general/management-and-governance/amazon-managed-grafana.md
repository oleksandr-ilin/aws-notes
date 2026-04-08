Overview
Managed Grafana service integrated with AWS data sources for dashboards and visualization.

Tasks
- Create workspaces, connect data sources (CloudWatch, Prometheus), and build dashboards.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Self-hosted Grafana | Full plugin support | Ops & maintenance

Limitations
- Plugin and customization limits compared to self-hosted Grafana.

Price info
- Workspace and user pricing depending on plan.

Network & multi-region
- Regional; connect to data sources across accounts/regions securely.

Popular use cases
- Centralized operational dashboards with AWS-managed data sources.

When Not to Use
- Extremely custom dashboarding needs requiring unsupported plugins.

DR strategy
- Export dashboard JSON and store in version control; backup workspace configs.
