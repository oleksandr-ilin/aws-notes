Overview
Simple one-button device event service that triggers AWS Lambda functions for single-action devices.

Tasks
- Register devices, map triggers to Lambda functions, and configure endpoints.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom webhook receivers | Flexible | More ops work |
| MQTT triggers to backend | Scalable | More plumbing required |

Limitations
- Very simple trigger model; limited device types supported.

Price info
- Charged per-device and per-trigger depending on usage.

Network & multi-region
- Regional service; design for device locality and Lambda endpoints.

Popular use cases
- Panic buttons, attendance buttons, simple remote triggers.

When Not to Use
- Complex device logic or multi-step workflows.

DR strategy
- Keep trigger mappings and Lambda functions under version control and test emergency rebindings.
