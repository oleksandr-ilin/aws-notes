Overview
Managed personalization/recommendation service that automates feature extraction and model training.

Tasks
- Ingest user-item interaction data, train campaigns, and serve real-time recommendations.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Custom recommenders on SageMaker | Flexibility | More engineering & ops |
| Open-source recommenders | Cost control | Integration work |

Limitations
- Data modeling constraints and cold-start challenges; tuning required for specific domains.

Price info
- Per-campaign + inference costs depending on traffic.

Network & multi-region
- Regional service; manage datasets and replication for global users.

Popular use cases
- E-commerce recommendations, content personalization, and real-time suggestions.

When Not to Use
- Very unique recommendation logic requiring specialized models—use SageMaker.

DR strategy
- Archive training datasets and exported model artifacts; version campaign configurations.
