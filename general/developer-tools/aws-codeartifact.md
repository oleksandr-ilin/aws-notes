# AWS CodeArtifact

## Service Overview
CodeArtifact is a managed artifact repository for npm, pip, Maven, and other package formats.

## Tasks this service can solve
- Host private packages, dependency proxying, and secure distribution
- Integrate with CI/CD for artifact promotion

## Alternatives
| Name | Type | Short description | Pro vs CodeArtifact | Cons vs CodeArtifact | Price comparison |
|---|---|---:|---|---|---|
| Artifactory / Nexus | Commercial/OSS | Full-featured repo managers | More enterprise features | More ops and license cost | Often more expensive; CodeArtifact simpler for AWS-centric teams |

## Limitations
- Feature set smaller than full enterprise artifact managers; retention and promotion workflows need integration.

## Price info
- Charged for storage and requests.

## Network & Multi-region considerations
- Repositories are regional; use replication or multi-region CI configurations for global teams.

## When Not to Use This Service
- Not ideal if you need advanced enterprise features (replication, fine-grained promotion) not provided by CodeArtifact.

## DR strategy
- Backup repository metadata and ensure CI pipelines can rebuild artifacts; store critical artifacts in cross-region S3 for recovery.

## Popular use cases
- Private package hosting, dependency proxying, and secure internal package distribution.
