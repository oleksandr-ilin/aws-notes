Overview
Image and video analysis service for label detection, face recognition, moderation, and more.

Tasks
- Send images/videos for analysis, tune collections for face recognition, and handle results.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| OpenCV / custom models | Full control | Ops & model training |
| Third-party vision APIs | Specialized features | Cost & lock-in |

Limitations
- Accuracy varies by use case; privacy and compliance concerns for face recognition.

Price info
- Per-image / per-minute video pricing depending on feature set.

Network & multi-region
- Regional endpoints; consider data residency and processing locality.

Popular use cases
- Content moderation, metadata extraction, and surveillance analytics.

When Not to Use
- Highly specialized vision tasks where custom models outperform managed APIs.

DR strategy
- Archive media and analysis results to S3 and retain metadata for audits.
