Overview
Core managed service for securely connecting, authenticating, and messaging IoT devices at scale.

Tasks
- Register devices, manage certificates, route messages to apps/streams, and set policies.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Self-managed MQTT brokers | Full control | Scalability & ops burden |
| Third-party IoT platforms | Vendor features | Integration & potential cost |

Limitations
- Device registry and connection scaling considerations; integration points required for downstream processing.

Price info
- Charged per message and connection time depending on features used.

Network & multi-region
- Regional endpoints; design for device geo-distribution and local gateway aggregation.

Popular use cases
- Device telemetry ingestion, command/control, and secure device authentication.

When Not to Use
- Very small fleets that can use simple MQTT brokers or direct cloud ingestion.

DR strategy
- Back up device registry exports and certificate inventories; test device reprovisioning workflows.
