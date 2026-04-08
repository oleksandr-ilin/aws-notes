# AWS Cloud9

## Service Overview
Cloud9 is a browser-based IDE with preconfigured development environments hosted in AWS.

## Tasks this service can solve
- Quick cloud-based development environments
- Pair-programming and shared terminals
- Cloud-native onboarding for developers

## Alternatives
| Name | Type | Short description | Pro vs Cloud9 | Cons vs Cloud9 | Price comparison |
|---|---|---:|---|---|---|
| VS Code / Gitpod | OSS/Commercial | Local or cloud IDEs | Richer local extensions and offline support | Less integrated with AWS by default | Varies; Cloud9 charges only for hosted instance resources |

## Limitations
- Web-based latency; dependent on network connectivity and underlying EC2 instance costs.

## Price info
- IDE usage is free; you pay for underlying EC2 instance hours and storage.

## Network & Multi-region considerations
- Environments are regional; use per-region environments for locality and access to regional resources.

## When Not to Use This Service
- Not ideal for offline-first workflows or when local dev environments are mandated for compliance.

## DR strategy
- Store environment setup scripts, dotfiles, and workspace configuration in source control; recreate environments via automation if needed.

## Popular use cases
- Onboarding new developers, quick experiments, and cloud-native demos.
