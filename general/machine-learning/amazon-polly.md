Overview
Text-to-speech service offering multiple voices and SSML support for speech generation.

Tasks
- Generate speech from text, manage voice selection, and integrate with streaming or file outputs.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Open-source TTS | Cost control | Quality & maintenance |
| Google TTS | Different voices & languages | Vendor choice |

Limitations
- Voice selection and regional support; high-volume cost considerations.

Price info
- Per-character or per-request pricing depending on usage.

Network & multi-region
- Regional endpoints; route requests to nearest region for lower latency.

Popular use cases
- IVR systems, accessibility features, and content narration.

When Not to Use
- Ultra-high-fidelity neural voices where specialized vendors may be preferable.

DR strategy
- Archive input text and generated audio to S3 and store voice settings in IaC.
