# Agent routing

> **Status:** outline draft.

## Two agents, one editorial pipeline
- **Cowork (desktop, interactive):** human-in-the-loop with the operator. Drafts in real time, asks questions, handles novel posts.
- **OpenClaw (Docker, scheduled):** weekly cadence content, runs 10:00–21:00, posts previews to Telegram.

## How a request decides which agent
- Initiated from the operator's desktop → Cowork.
- Initiated by a cron job → OpenClaw.
- There is no router; the entry points are physically distinct.

## What both agents share
- The same skills library (16 marketing skills, loaded into both).
- The same publishing library (guardrails, channel routing).
- The same approval gate (Telegram, same channels).

## What only Cowork has
- Computer-use, file-system access, native MCPs for the operator's desktop apps.
- The ability to ask a question and wait for an answer.

## What only OpenClaw has
- Uptime. Cowork shuts down with the operator's laptop.
- Cooldown management between Anthropic-API calls (cron-aware throttling).
