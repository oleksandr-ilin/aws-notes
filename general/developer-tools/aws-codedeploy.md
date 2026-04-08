# AWS CodeDeploy

## Service Overview
CodeDeploy automates application deployments to EC2, Lambda, and on-premises instances with hooks and deployment strategies.

## Tasks this service can solve
- Automated blue/green and rolling deployments
- Safe automated rollbacks and lifecycle hooks

## Alternatives
| Name | Type | Short description | Pro vs CodeDeploy | Cons vs CodeDeploy | Price comparison |
|---|---|---:|---|---|---|
| Custom deployment pipelines (Ansible, scripts) | OSS/Custom | Custom deploy automation | Full control | More maintenance | Ops cost vs managed CodeDeploy
| Third-party CD tools | Commercial | Feature-rich deployment tooling | Advanced deployment strategies and UI | License costs | Varies; may be higher

## Limitations
- Integrations and deployment strategies are opinionated; advanced scenarios may require custom scripting.

## Price info
- No separate charge; pay for underlying resources and any CodePipeline integrations.

## Network & Multi-region considerations
- Configure per-region deployments and use multi-region pipelines for global delivery; ensure artifact storage accessibility in target regions.

## When Not to Use This Service
- Not ideal if you already have mature non-AWS deployment orchestration tightly coupled to other ecosystems.

## DR strategy
- Keep deployment templates, hooks, and rollback playbooks in source control; test restore and rollback steps regularly.

## Popular use cases
- Blue/green deployments for web services, Lambda versioned deployments, and automated rollouts.
