# Workflow: Blog generation

> **Status:** outline draft.

## Trigger
- Weekly cron (Tue 09:00 DE) on OpenClaw, or manual from Cowork desktop.

## Pipeline
1. Topic selection from the Q-plan content calendar.
2. Outline draft from Anthropic Claude with the brand-voice skill loaded.
3. Body draft → guardrail library inspection.
4. Hero image request → Gemini Nano Banana with the project visual rules.
5. Composite preview → Telegram approval gate.
6. On approval → POST to `/api/publishBlog` with bearer auth → Firestore.

## Failure modes and how they're handled
- Guardrail rejection → returns reasons; operator decides whether to revise or skip.
- Image gen failure → re-roll with different seed; never block on this.
- `/api/publishBlog` 4xx → preserved draft in Firestore `drafts` collection, alert to Telegram.

## What's in the partial code
- See `partial-code-examples/publish_blog.py` for the redacted publishing call.
