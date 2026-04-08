# AWS X-Ray

## Service Overview
X-Ray provides tracing, service maps, and latency/profiling insights for distributed applications.

## Tasks this service can solve
- Trace requests across services and identify latency hotspots
- Visualize service maps and dependency graphs

## Alternatives
| Name | Type | Short description | Pro vs X-Ray | Cons vs X-Ray | Price comparison |
|---|---|---:|---|---|---|
| Jaeger / Zipkin | OSS | Open-source distributed tracing | Full control and community tools | Self-hosting and storage ops | OSS hosting cost vs X-Ray managed pricing
| Datadog APM | Commercial | Full observability platform | Rich UI and integrations | Costly licensing | Higher than X-Ray but full-featured

## Limitations
- Sampling and data retention limits; may miss low-frequency traces unless configured.

## Price info
- Charged for trace data ingested and retained; additional costs for integrated logs and insights.

## Network & Multi-region considerations
- Tracing is regional; aggregate traces to central observability pipelines if needed. Use service map data to correlate across accounts/regions using custom tools.

## When Not to Use This Service
- Not ideal if you require vendor-neutral tracing with full control over storage and retention; consider OSS tracing stacks.

## DR strategy
- Export critical trace data and retention configs; ensure instrumentation configs and sampling rules are in source control to re-enable after recovery.

## Popular use cases
- Debugging distributed microservices, latency investigations, and performance tuning.
