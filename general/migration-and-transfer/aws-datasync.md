Overview
AWS DataSync automates and accelerates online data transfer between on-prem storage and AWS.

Tasks
- Deploy DataSync agent, configure source and destination locations, run tasks and monitor throughput.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Rsync over VPN | Simple | Slower for very large datasets |
| Third-party transfer tools | Feature-rich | Additional cost

Limitations
- Requires agent for on-prem; throughput depends on network & agent sizing.

Price info
- Charged per-GB transferred and for agent usage where applicable.

Network & multi-region
- Typically region-targeted; plan bandwidth and consider parallel tasks for speed.

Popular use cases
- Large-scale dataset migrations, ongoing syncs for file systems.

When Not to Use
- Small ad-hoc file copies where manual methods are sufficient.

DR strategy
- Use DataSync for periodic bulk syncs as part of DR data seeding.
