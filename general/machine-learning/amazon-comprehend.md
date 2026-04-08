Overview
Managed NLP service for entity extraction, sentiment, classification and topic modelling.

Tasks
- Run batch or real-time text analysis; integrate with data pipelines and search.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| spaCy / custom models | Full control | More ops and labeling effort |
| Hugging Face models | Modern models | Hosting & tuning required |

Limitations
- Pre-trained model limits; custom model flexibility is limited compared to SageMaker.

Price info
- Per-unit or per-API call pricing depending on feature.

Network & multi-region
- Regional endpoints; replicate or route text to the nearest region to reduce latency.

Popular use cases
- Entity extraction, sentiment analysis, document classification.

When Not to Use
- Research or highly-custom models—use SageMaker and custom ML stacks.

DR strategy
- Archive inputs and outputs to S3 and version processing pipelines for replay.
