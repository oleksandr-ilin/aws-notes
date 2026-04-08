Overview
Stream desktop applications from AWS to users via a managed streaming service.

Tasks
- Provision fleets and stacks, configure image builder, assign users and entitlements.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| VDI (Citrix) | Mature feature set | Higher ops and license cost |
| RDP on EC2 | Full control | Management overhead |

Limitations
- Network-dependent UX; image management is required for app updates.

Price info
- Per-stream instance + user/session pricing; depends on instance types used.

Network & multi-region
- Streams require stable network; deploy in-region nearest users or use edge connectivity.

Popular use cases
- Delivering Windows apps to BYOD devices, software demos, CAD/graphics apps with GPU instances.

When Not to Use
- Offline users or ultra-low-latency local apps.

DR strategy
- Keep image builder automation and store images in versioned repos; prepare recovery instances in a standby region.
