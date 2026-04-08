Overview
Full ML platform for training, tuning, deploying and managing models at scale.

Tasks
- Prepare datasets, train models with built-in algorithms or custom containers, deploy endpoints, and monitor.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Self-managed ML infra | Full control | Heavy ops & scaling concerns |
| Other cloud ML platforms | Different features | Migration effort |

Limitations
- Cost for heavy training; requires ML expertise to optimize and manage workflows.

Price info
- Instance-hour + training/inference charges and additional storage & data transfer costs.

Network & multi-region
- Regional; manage model artifact replication for global inference and training data locality.

Popular use cases
- Custom model training, hyperparameter tuning, batch and real-time inference.

When Not to Use
- Very small models that are cheaper to run as serverless functions.

DR strategy
- Store training artifacts, checkpoints and model images in versioned S3 buckets and backup endpoint configs.
