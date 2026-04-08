Overview
Customer engagement service for analytics-driven messaging (email, SMS, push) and campaigns.

Tasks
- Configure channels, define segments, create campaigns and analyze results.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Third‑party messaging platforms | Rich feature sets | Cost & integration |
| SES + custom workflows | More control | More orchestration work |

Limitations
- Campaign complexity and per‑message costs; limits on throughput per channel.

Price info
- Per-message + endpoint charges; channel-dependent pricing.

Network & multi-region
- Regional service; integrate with multi-region data pipelines for global campaigns.

Popular use cases
- Targeted marketing, transactional messaging, and usage analytics.

When Not to Use
- Very high-volume SMS where carrier relationships are needed—use specialized providers.

DR strategy
- Archive campaign definitions and analytics to S3; use IaC to re-create segments.
