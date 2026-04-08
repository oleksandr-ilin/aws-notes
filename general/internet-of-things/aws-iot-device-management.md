Overview
Fleet management service for provisioning, organizing, and performing OTA updates across devices.

Tasks
- Register devices, run OTA updates, provision fleets, and monitor device health.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom provisioning services | Full control | Higher maintenance |
| Third‑party device management | Richer features | Cost & vendor lock-in |

Limitations
- Scale and device-type support vary; provisioning methods must be implemented per device.

Price info
- Charged per device and for management operations.

Network & multi-region
- Regional service; plan for devices across geographies with local gateways where needed.

Popular use cases
- OTA firmware updates, large-scale provisioning and fleet inventory.

When Not to Use
- Very simple or single-device scenarios.

DR strategy
- Keep firmware/artifact repositories versioned and store device state exports for recovery.
