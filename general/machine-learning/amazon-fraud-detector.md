Overview
Managed service for building fraud detection models using pre-built templates and event-based training.

Tasks
- Define events, train models, evaluate, and deploy detectors integrated with real-time workflows.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom ML pipelines on SageMaker | Full control | More expertise & ops |
| Third-party fraud platforms | Domain features | Cost & integration |

Limitations
- Template-based approach may be limiting for highly novel fraud patterns.

Price info
- Per-evaluation + training charges depending on usage.

Network & multi-region
- Regional; route events to the region hosting detectors for low-latency decisions.

Popular use cases
- Transaction scoring, account takeover detection, and real-time risk decisions.

When Not to Use
- Non-standard fraud patterns needing custom feature engineering—use SageMaker.

DR strategy
- Archive training events and model artifacts in S3; version detector configs for recovery.
