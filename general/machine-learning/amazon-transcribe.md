Overview
Speech-to-text transcription service for audio/video with multiple languages.

Tasks
- Submit audio for batch or streaming transcription, process results, and integrate into pipelines.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Open-source ASR (Kaldi, Vosk) | Cost control | Operational maintenance |
| Third-party ASR providers | Different language support | Vendor differences |

Limitations
- Accuracy varies by audio quality and domain-specific vocabulary; speaker diarization costs extra.

Price info
- Per-second or per-minute transcription pricing.

Network & multi-region
- Regional endpoints; for low-latency streaming route to the nearest region.

Popular use cases
- Call transcription, subtitles, meeting notes.

When Not to Use
- Extremely noisy audio or languages not supported—consider custom ASR models.

DR strategy
- Archive audio and transcripts to S3 for reprocessing and audits.
