# Architecture evolution

The current system is the third architecture. Each version was a response to a specific failure mode of the one before it. Documenting all three is more useful than only showing the final state — the dead ends explain the choices that survived.

## V1 — Single agent, prompt-based guardrails

The first version was the obvious one. An LLM with a long system prompt, a few tool calls, and instructions to publish to social channels when content was ready. Everything lived inside the agent: tone of voice, language rules, compliance phrases, the channel-specific formatting, the schedule. The operator gave high-level direction and the agent figured the rest out.

This worked for about a week. Three failure modes killed it.

The first was inconsistency. The same instruction produced subtly different outputs on different days. Prices were sometimes hedged with "ab", sometimes not. Compound German words occasionally got truncated. The brand voice drifted between posts.

The second was the compliance problem. A prompt instruction not to use "Festpreis" or "garantierter Preis" is honored most of the time, but most of the time is not good enough when the downside is a regulatory letter. The agent had no fail-closed mechanism — if the model decided the instruction was advisory rather than binding, the post went out anyway.

The third was operational. The agent could not be paused, audited, or rolled back. A bad draft published in seconds was hard to retract; a string of bad drafts was a brand problem.

The lesson was direct: the LLM is part of the system, not the system. The system needs deterministic layers that the LLM cannot route around.

## V2 — Agent plus guardrail library

The second version added a thin Python library between the LLM and the publishing APIs. The library held the rules: language check, price hedging check, denylist regex, channel-specific voice constraints. The LLM produced a payload; the library inspected it; the publishing call only fired if the payload passed.

This was a substantial improvement. The compliance class of bugs collapsed. The library was the source of truth for what could be published, and the LLM could not argue with it. The library also became the place where new rules landed — a new forbidden phrase became a one-line regex addition, deployed in seconds.

Two problems remained.

First, the system still had no scheduling. Posts went out when the operator was at the keyboard, which meant nothing went out on weekends or in the evenings — exactly when social engagement peaks for this audience. Adding cron jobs to the operator's local machine worked until the machine slept, and then it didn't.

Second, multi-channel publishing meant maintaining four separate API integrations (YouTube, Instagram, X, LinkedIn), each with its own auth flow, rate limits, and edge cases. The publishing surface area was disproportionate to what it was buying.

## V3 — Current state

The current architecture, laid out as a text-style topology diagram in the [Topology section](../README.md#topology) of the repo README, addresses both gaps.

Scheduling is split between two agents. The interactive agent — Anthropic Claude running in Cowork on the operator's desktop — handles in-the-moment work, the editorial decisions, the unusual posts. The scheduled agent — OpenClaw, a Dockerized Anthropic-API-backed agent on a low-cost VPS — handles the weekly cadence. It drafts content on a 10:00–21:00 window and routes previews to a Telegram channel for the operator to approve. Approved drafts flow into the same guardrail library and the same publishing surface as desktop-initiated work, so there is one editorial pipeline regardless of where the draft originated.

Multi-channel publishing collapsed to a single REST API (Upload-Post) that fans out to YouTube, Instagram, X, and LinkedIn from one call. The free tier caps at ten uploads per month, which forced a discipline of rotation: not every post goes to every channel. A video goes to YouTube and Instagram; a photo goes to Instagram and X; a text post goes to X. The cap turned out to be a feature, not a constraint — it stopped the system from over-publishing.

The compliance library matured into something with a more defensible posture. The German-language check became permissive rather than strict, because the early strict version blocked legitimate German posts that happened to use cognates. The current heuristic blocks only when text shows multiple English function words AND zero German signals — and German signals include umlauts, common function words, and Skoda-specific terms. The price hedging check moved from a substring scan to a regex with positive context, after a bug let "ab Lager" ("from stock") pass as a price hedge because "ab" appeared somewhere near a "€". The denylist enforces word boundaries so compounds like "Beispielrechnung" pass while bare "Rechnung" still blocks.

## What the next version probably looks like

Two pressures point at V4.

One is observability. The current system logs to Firebase Functions logs and Telegram alerts, which is fine when the operator is paying attention and a problem when the operator is not. A real ops dashboard — even a simple Grid.js view backed by Firestore — would replace the "scroll Telegram for missed alerts" workflow.

The other is the LinkedIn Company Page reconnect. Upload-Post is currently linked to a personal LinkedIn profile, which is the wrong publishing surface for a referral business. Until the Company Page is reconnected in the Upload-Post dashboard, LinkedIn is gated in the publishing library and posts there opt-in only. This is the single biggest distribution gap and the most clearly fixable thing in the backlog.

The deeper question for V4 is whether to keep two agents or collapse to one. The split made sense when desktop and headless had different reliability profiles, but if the headless agent gets stable enough to do everything the desktop one does, the operator becomes a pure approver rather than an editor. That changes the design center, and probably the cost structure.

---

[← Business problem](business-problem.md) · [Up: README](../README.md) · [Next: Orchestration design →](orchestration-design.md)
