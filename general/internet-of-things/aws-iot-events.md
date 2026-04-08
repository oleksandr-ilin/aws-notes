Overview
Event detection and response service for IoT telemetry and state-based event patterns.

Tasks
- Define detectors, inputs and actions to trigger on detected patterns in telemetry streams.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom rules engine | Customizable | More ops and maintenance |
| Use Lambda/Step Functions | Flexible | More plumbing required |

Limitations
- Best for predefined pattern detection; complex logic may require external processing.

Price info
- Charged per detector-hour and input processing.

Network & multi-region
- Regional; aggregate detections centrally for cross-region fleets.

Popular use cases
- Alarm detection, anomaly alerts, and automated device actions.

When Not to Use
- Arbitrary complex analytics or heavy ML—use analytics pipelines instead.

DR strategy
- Version detector definitions and ensure actions (SNS/Lambda) have recovery workflows.
