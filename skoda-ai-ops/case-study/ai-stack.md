# AI stack

The stack is opinionated. Every model and every integration was picked for a specific job, and several obvious options were rejected on inspection. The reasoning matters more than the specific vendors, because the vendors will change; the routing logic should not.

## The model mix

Anthropic Claude is the primary reasoning model. It is the model both agents — the interactive desktop one (Cowork) and the scheduled headless one (OpenClaw) — run on. It is what drafts the blog posts, the newsletter issues, the social captions, the dealer hand-off emails, and most of the editorial decisions in between. The reasons are not exotic: it follows formatting and tone instructions reliably, it handles German prose better than the alternatives that were tested side by side, and the Cowork desktop experience makes the interactive workflow substantially better than running a model through a chat tab.

Image generation runs on Google Gemini's Nano Banana model. The trade-off here is cost-per-image versus quality, and at this volume — roughly thirty to fifty images per month between blog hero images, social photos, and PDF assets — Nano Banana is comfortably good enough at a fraction of the cost of the premium image models. The cases where it fails are predictable: complex multi-subject scenes, specific car-model accuracy beyond stock photography, anything requiring text rendering in the image. Those are handled by curating from existing photography rather than generating.

Video generation runs on Google Veo 3.1. The reason is not cost — Veo is the most expensive line item on the production side — but capability. Veo handles both 9:16 vertical (the Shorts and Reels format) and 16:9 cinematic with native synchronized audio, which means the engine note in a driving shot is generated in lockstep with the visual. The alternative is to source ambient road noise separately and edit it under the video, which adds production complexity that a one-person operation cannot afford. Veo is also the only model in the current generation that interpolates between a "before" still and an "after" still smoothly enough to use for a "dirty car → clean car" or "old config → new config" transition without obvious artifacts.

A local Ollama deployment runs on the VPS as a fallback. It is never the primary path. The reason is reliability under multi-step workflows: any task that loads several skills and calls them in sequence times out reliably on the local model in practice. A 30-second cloud draft beats a 5-minute local draft every time, and on a system where the operator's attention is the scarce resource, latency dominates cost.

## Why MCP is the integration plane

The Model Context Protocol is the integration layer that connects the agents to the rest of the system. Every external service the agents talk to is exposed as an MCP server: Firebase (project introspection, Firestore CRUD), Gmail (outbound email for dealer hand-off and newsletter), Airtable (CRM read/write), Gemini Image (generation), Veo (generation), a custom website-edit server (deploys), Excalidraw (diagram authoring), and on the desktop side, computer-use (controlling the operator's apps when needed).

The alternative — writing a bespoke client for each API — is what most ad-hoc agent systems start with and what they regret a year later. The bespoke approach means every tool call has its own auth flow, its own retry logic, its own argument schema, and its own bug surface. MCP centralizes that work into the server, and the agent gets a uniform tool-calling interface for everything.

The practical benefit shows up in the dev loop. Adding a new capability is "stand up an MCP server" rather than "write a tool, wire its auth, handle its errors, document its arguments, and pray the agent picks the right one." Most of the MCP servers in this system are off-the-shelf or one-file implementations; the custom ones (website deploys, the Preisvorschlag PDF builder) were written in an afternoon each.

There is one transport-layer subtlety worth documenting. Some MCP servers run as stdio processes spawned by the desktop client, and some run as HTTP/SSE servers consumed over the network. The same server software can sometimes be configured either way, and the choice matters: the Gemini Image MCP runs as a launchd-supervised SSE server on port 8901 for the Docker OpenClaw container to consume, and as a separately-launched stdio server for the desktop Cowork client. Trying to share a single stdio instance across both clients does not work, because stdio is one-to-one. The two-server pattern is documented in the change log.

## Why not a single foundation model end-to-end

The obvious counter-design is to pick one frontier model that does text, image, and video, and route everything through it. There are vendors offering that today. The reasons this system does not take that bet:

Image generation on the same model that does text reasoning is a worse trade-off on cost-per-output. The specialist image model is one to two orders of magnitude cheaper at the per-image volume this system operates at, and the quality is more than sufficient for editorial-grade automotive marketing.

Video generation on a generalist model is currently not viable at the quality bar required. Veo's native audio and aspect-ratio handling are not matched by the multi-modal models. This may change in the next twelve months; if it does, this is the part of the stack most likely to consolidate.

Vendor diversification is a side benefit rather than the primary driver. The text model could be any of three vendors and the system would not look dramatically different. The image model and video model are picked for capability, not strategically.

The single-model bet is also a single-point-of-failure bet. The Veo MCP has gone down twice in the last quarter (403 PERMISSION_DENIED on `veo-3.1-fast`, 404 on `veo-3.0`), and during those windows the system fell back to reusing existing video beds and overlaying text on them. If video, text, and image were all routed through one provider, an outage of that provider would stop the entire content engine; with the split, an outage of one degrades the system but does not stop it.

## What is not in the stack and why

No fine-tuning. The brand voice, language rules, and channel-specific tone are achievable with prompt instructions plus the publishing library's hard guardrails. Fine-tuning is on the table if the volume of corrections to the model output grows past where a few-shot prompt can fix, but at current volume it is not justified.

No vector database. The content corpus — past blog posts, past newsletters, past social — is on the order of hundreds of items, not hundreds of thousands. A vector store would be a yak-shave; in-context retrieval over the relevant subset is fine.

No agent framework (LangChain, CrewAI, AutoGen, et cetera). The agent abstraction in this system is just the LLM plus a set of MCP tools plus the publishing library, and the orchestration is whatever the agent decides to do under instruction. A framework would add a layer of abstraction without changing the behavior; for the scale of this system, the cost of the abstraction outweighs the benefit.

No custom prompt orchestration layer. The Anthropic API and the Cowork desktop client are the orchestration, and the skills loaded into both are the structured guidance. Prompts live in skill files, not in code, which keeps them readable by the operator without diving into a Python module.

No real-time streaming or websockets. The system is asynchronous by design — drafts are produced, posted to Telegram, approved or rejected, published or shelved. There is no use case where a real-time stream of model output to the operator would change the workflow, and the synchronous design is substantially easier to reason about when something goes wrong.

## What changes the stack

Three things would force a stack revision. A meaningful drop in latency for local-model multi-step workflows would let the Ollama deployment carry primary load and reduce cloud spend, though it would not change the design center. A generalist multi-modal model that matches Veo's quality for 9:16 video with native audio would collapse the image and video lines into one provider. And a regulatory change — for example, a stricter German interpretation of AI-disclosure rules around generated content — would force a watermarking or provenance layer into the publishing library.

None of these are imminent. The stack as it stands today is the right shape for the business as it stands today.

---

[← Orchestration design](orchestration-design.md) · [Up: README](../README.md) · [Next: Cost routing →](cost-routing.md)
