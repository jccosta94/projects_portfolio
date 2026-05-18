# Model selection

> **Status:** outline draft.

## Routing by job
- **Reasoning / drafting:** Anthropic Claude. Primary choice for any text that goes to the public.
- **Image gen:** Gemini Nano Banana. Cheap and good enough for editorial-quality automotive photography.
- **Video:** Veo 3.1. Picked for native audio and aspect-ratio handling, not for cost.
- **Fallback for text:** Ollama-hosted models. Reliability backup only — never primary.

## Why local models are not the primary path
- Multi-step skill workflows (16 marketing skills, each with a sub-task tree) time out on local hosts in practice.
- A 30-second cloud draft beats a 5-minute local draft every time.
- The cost saving from local is not large enough to justify the latency.

## Why no single foundation model end-to-end
- Image, text, and video have different price/quality/latency curves.
- Vendor diversification is a side benefit, not the primary driver.

## What would change this
- A meaningful improvement in local-model latency on multi-step workflows.
- A unified model that handles 9:16 video with synchronized audio at Veo quality, for less.
