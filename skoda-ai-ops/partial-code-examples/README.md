# Partial code examples

These are **redacted, illustrative** excerpts. They are not runnable. API keys, project identifiers, channel IDs, and secret tokens are replaced with placeholders like `<REDACTED>` or `${ENV_VAR}`. The intent is to show shape, structure, and design decisions — not to ship code.

## What's in this folder

- `publish.py` — the multi-channel publishing CLI (excerpt). Shows the dispatch logic and the integration with the guardrail library.
- `guardrails.py` — the publish-time content rules library (excerpt). Shows the German-language check, price hedging regex, and denylist enforcement.
- `publishBlog.ts` — the Cloud Function for blog publishing (excerpt). Shows the bearer auth check and the Firestore write.

## What's deliberately missing

- Real API keys (Anthropic, Gemini, Veo, Upload-Post, Airtable, Gmail SMTP).
- Real Firebase project IDs and GA4 measurement IDs.
- Real Telegram chat IDs and bot tokens.
- Customer data, dealer relationships, commission rates.
- Anything in `.env.*`, ever.

## Why redact rather than full source

The repository is a case study, not a starter kit. The value for a technical reader is in the patterns — fail-closed guardrails, single fan-out publishing, approval gates as a hard layer — not in the specific bindings. Anyone wanting to build a similar system will write their own version against their own service boundaries.
