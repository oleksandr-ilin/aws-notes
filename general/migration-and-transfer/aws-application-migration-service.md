Overview
Server/VM replication and orchestration service for lift-and-shift migrations to AWS.

Tasks
- Configure replication, run test cutovers, and orchestrate final migration cutover.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Manual image conversion & import | Control | Time-consuming |
| Third-party migration tools | Feature-rich | Cost & integration

Limitations
- Bandwidth and replication requirements; some OS/app specifics require tailoring.

Price info
- Replication and staging resource costs; recovery compute billed separately.

Network & multi-region
- Replication target is a region; plan for staging account and network connectivity.

Popular use cases
- Large-scale rehosting of VMs with minimal downtime.

When Not to Use
- Complex replatforming that benefits from refactoring rather than lift-and-shift.

DR strategy
- Use migration workflows also for DR runbooks; keep replication configs and tests automated.
