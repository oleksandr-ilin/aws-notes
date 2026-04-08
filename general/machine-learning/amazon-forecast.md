Overview
Managed time-series forecasting service that automates model building and tuning.

Tasks
- Prepare datasets, train predictors, and deploy forecasts for business planning.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Prophet / in-house models | Control & interpretability | More maintenance |
| SageMaker Autopilot | Greater flexibility | Higher cost & complexity |

Limitations
- Requires careful feature engineering; not suitable for all forecasting problems.

Price info
- Per-forecast-hour + storage; depends on training complexity.

Network & multi-region
- Regional service; store datasets in S3 and manage cross-region replication if needed.

Popular use cases
- Demand forecasting, capacity planning, financial projections.

When Not to Use
- Research-grade or highly-custom forecasting models.

DR strategy
- Keep historical datasets and trained predictor artifacts in S3 with versioning.
