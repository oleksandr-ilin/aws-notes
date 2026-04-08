Overview
AWS Outposts extends AWS infrastructure and services to on-premises locations for low-latency or data residency needs.

Tasks
- Order Outpost rack, plan network connectivity, deploy workloads via supported AWS services.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Local virtualization | Full control | No managed AWS services

Limitations
- Hardware procurement lead times and footprint constraints.

Price info
- Hardware + service billing and ongoing management fees.

Network & multi-region
- Acts like a regional extension; plan connectivity to the parent AWS region.

Popular use cases
- Low-latency telecom, regulated data residency, or local processing needs.

When Not to Use
- General cloud-native workloads without latency/residency constraints.

DR strategy
- Mirror configurations to a cloud region and automate failback procedures.
