Overview
Conversational AI service for building chatbots and voice assistants with intent recognition.

Tasks
- Define intents, slot types, build flows, integrate with channels and Lambda for fulfillment.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Dialogflow / Rasa | Different features & deployment models | Integration differences |
| Custom NLU on SageMaker | Full control | Higher ops cost |

Limitations
- Complexity for very advanced dialog flows; channel integration work required.

Price info
- Per-request + speech/voice charges for voice interactions.

Network & multi-region
- Regional; route traffic to nearest region for latency-sensitive interactions.

Popular use cases
- Customer support chatbots, IVR systems, and command interfaces.

When Not to Use
- Deeply-custom dialog managers or specialized multimodal assistants—use custom stacks.

DR strategy
- Keep bot definitions and training data in version control and IaC for quick rebuilds.
