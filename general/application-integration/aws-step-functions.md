# AWS Step Functions

## Service Overview
AWS Step Functions is a serverless orchestration service for building resilient workflows as state machines with visual monitoring.

## Tasks this service can solve
- Orchestrating complex serverless workflows
- Long-running business processes with human approvals
- Error handling and retries across distributed steps

## Alternatives
| Name | Type | Short description | Pro vs Step Functions | Cons vs Step Functions | Price comparison |
|---|---|---:|---|---|---|
| Custom orchestrator (Lambda + SQS) | AWS | Homegrown orchestration using functions and queues | No per-state billing | More orchestration code to maintain | Can be cheaper but more engineering cost |
| Temporal / Cadence | OSS/Commercial | Durable workflow engines | Rich programming models and long-running workflows | Self-hosting or commercial cost | Operational cost vs per-state Step Functions billing |
| AWS SWF (legacy) | AWS | Older workflow service | Low-level control | Legacy and more complex | Typically replaced by Step Functions |

## Limitations
- Per-state transition and duration pricing can be expensive for very chatty workflows.
- State size and input limits exist; long-running executions have soft limits.

## Price info
- Charged per state transition and for express workflows by duration/requests; Standard and Express workflow types have different pricing models.

## Network & Multi-region considerations
- Step Functions are regional. For multi-region resilience, run workflows in multiple regions and coordinate via cross-region events or API gateways.
- Integrates naturally with Lambda, ECS, SNS, SQS, and API Gateway.

## Popular use cases
- Order processing pipelines with retries and compensating actions
- ETL orchestration and long-running batch processes
- Business process automation with human-in-the-loop steps

## When Not to Use This Service
- Not ideal for extremely chatty or fine-grained state transitions due to per-state costs; consider event-driven or custom orchestrators when cost or latency is critical.

## DR strategy
- Use multi-region workflow deployment for critical processes, persist durable state in DynamoDB or S3, and keep workflow definitions in source control. Recreate or replay workflow state in another region when necessary.
