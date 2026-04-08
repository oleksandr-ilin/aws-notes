# AWS CodeBuild

## Service Overview
CodeBuild is a fully managed build service that runs CI builds in isolated build environments.

## Tasks this service can solve
- Run build/test pipelines, produce artifacts, and integrate with CodePipeline
- Scale build concurrency without managing build servers

## Alternatives
| Name | Type | Short description | Pro vs CodeBuild | Cons vs CodeBuild | Price comparison |
|---|---|---:|---|---|---|
| Jenkins / GitHub Actions | OSS/Commercial | CI solutions with broad ecosystems | More plugin ecosystem and custom runners | More ops (Jenkins) or vendor lock-in (GH Actions) | Varies; CodeBuild billed per build minute

## Limitations
- Build environment customizations limited to provided images or custom images; concurrency limits may apply.

## Price info
- Charged per build-minute by compute type.

## Network & Multi-region considerations
- Build projects are regional; use region-specific runners for regional resource access and low-latency artifact pushes.

## When Not to Use This Service
- Not ideal if you require highly customized build agents with persistent state; consider self-hosted runners.

## DR strategy
- Keep buildspecs and build image definitions in source control and CI pipeline templates in IaC to recreate build infrastructure.

## Popular use cases
- CI for serverless apps, container image builds, and test automation.
