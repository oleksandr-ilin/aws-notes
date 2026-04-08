Overview
Document OCR and structured data extraction service (forms, tables) for scanned documents.

Tasks
- Submit documents for analysis, parse returned structured data and integrate with workflows.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Tesseract + custom parsers | Cost control | Lower accuracy on complex docs |
| Third-party document extraction vendors | Industry features | Cost & integration |

Limitations
- Accuracy varies with document quality; complex layouts may need custom post-processing.

Price info
- Per-page pricing depending on feature and extraction complexity.

Network & multi-region
- Regional endpoints; keep privacy and residency constraints in mind for documents.

Popular use cases
- Invoice extraction, form processing, and automated data entry.

When Not to Use
- Highly-custom or noisy documents better handled by bespoke pipelines.

DR strategy
- Archive original documents and extraction outputs to versioned S3 buckets for reprocessing.
