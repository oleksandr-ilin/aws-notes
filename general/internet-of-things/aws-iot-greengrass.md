Overview
Edge runtime for local compute, messaging and ML inference on devices (Greengrass v2).

Tasks
- Install Greengrass runtime, deploy components, and manage local deployments and permissions.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Edge containers | Familiar tooling | More platform work |
| Custom edge agents | Full control | Heavy ops burden |

Limitations
- Device resource constraints and component packaging; offline sync patterns must be handled.

Price info
- Software is free, costs come from device compute and management; some AWS features billed separately.

Network & multi-region
- Designed to run locally; coordinate with cloud for deployment artifacts and telemetry aggregation.

Popular use cases
- Local ML inference, low-latency control loops, and protocol translation at the edge.

When Not to Use
- Pure cloud workloads or when devices lack sufficient compute/storage.

DR strategy
- Keep component artifacts in versioned storage and test device redeployment and rollback procedures.
