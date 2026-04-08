# Amazon CodeGuru

## Service Overview
CodeGuru provides automated code review suggestions and application performance profiling.

## Tasks this service can solve
- Detect common code defects and performance hotspots
- Profile applications in production to identify CPU and latency issues

## Alternatives
| Name | Type | Short description | Pro vs CodeGuru | Cons vs CodeGuru | Price comparison |
|---|---|---:|---|---|---|
| SonarQube / SCA tools | OSS/Commercial | Static analysis and code quality | More rules and community plugins | Self-hosting or license fees | Varies; CodeGuru billed per-review/profiling

## Limitations
- Language and rule coverage is limited; may not catch all domain-specific issues.

## Price info
- Charged per code review and profiler sampling; check current pricing.

## Network & Multi-region considerations
- Service is regional; profile agents communicate with AWS endpoints—consider VPC endpoints for private networks.

## When Not to Use This Service
- Not ideal if you need deep, language-specific static analysis beyond CodeGuru's scope; use specialized tools.

## DR strategy
- Keep reviewer configuration and profiling instrumentation code in source control and automation to re-enable profiling after incidents.

## Popular use cases
- Code quality improvement and runtime performance tuning in critical services.
