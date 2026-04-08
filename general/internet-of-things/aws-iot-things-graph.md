Overview
Visual workflow and orchestration service for IoT devices (deprecated / limited modern use).

Tasks
- Create flow diagrams connecting devices and cloud services for simple orchestration.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom orchestration with Step Functions | More control | More integration work |
| Greengrass components | Local orchestration | Edge-only scope |

Limitations
- Limited adoption and updates; evaluate viability before committing.

Price info
- Charged based on flows and executions (check current pricing for deprecation nuances).

Network & multi-region
- Regional; suited for local device orchestrations.

Popular use cases
- Simple visual orchestration for prototyping device interactions.

When Not to Use
- Long-term production workflows—prefer supported orchestration tools.

DR strategy
- Export flows and keep versions in IaC; have alternate orchestration plans.
