Overview
Legacy media transcoding service for simple media format transformations.

Tasks
- Submit source media, configure presets, and retrieve transcoded outputs.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| AWS Elemental MediaConvert | Advanced features & codecs | More configuration complexity |
| Self-hosted FFmpeg pipelines | Full control | Operational overhead |

Limitations
- Limited modern codec support and features compared to MediaConvert; mostly for simple tasks.

Price info
- Charged per-minute of output and resolution tier.

Network & multi-region
- Regional service; store inputs/outputs in S3 and replicate for cross-region workflows.

Popular use cases
- Simple format conversion and light-weight transcoding pipelines.

When Not to Use
- High-quality broadcast or feature-rich transcoding—use MediaConvert or Elemental solutions.

DR strategy
- Keep source media in versioned S3 buckets and pipeline definitions in IaC for recovery.
