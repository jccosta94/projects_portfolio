# Lessons learned

> **Status:** outline draft — not yet written in full prose.

## Prompt-based guardrails do not survive contact with a regulated domain
- The V1 failure mode
- Why a deterministic library layer became non-negotiable
- The few cases where prompt-level rules are still fine (and why)

## German-language heuristics need to be permissive, not strict
- The bug: strict check blocked legitimate German posts with cognates
- The fix: block only when text affirmatively looks English (multiple function words) AND has zero German signals
- The general principle: prefer false negatives over false positives in language detection for non-English content

## "ab Lager" is not a price hedge
- The bug: naive substring matching let "ab Lager" (from stock) pass as a price hedge
- The fix: regex with positive context — `ab` must immediately precede a `€` value
- The general principle: in regex-based compliance, write positive matches with explicit context, not negative matches with assumed proximity

## The first real publishing attempt fails for boring reasons
- The Upload-Post `/api/upload_text` endpoint uses `title` as the post-body field, not `text`
- The account-level `limit` returned by `GET /api/uploadposts/users` (2) is not the upload count cap (10)
- The general principle: integrate against the actual response shape, not the OpenAPI doc; track usage from real upload responses

## Local models are a fallback, not a primary
- qwen3.6:35b-a3b times out on multi-step skill workflows in practice
- A 30-second draft on a cloud model beats a 5-minute draft on a local one, every time
- The general principle: route by reliability first, cost second

## Treat ad-platform attribution as directional, not gospel
- Google Ads' conversion column and the Firestore `quote_events` collection will not agree.
- GA4 deduplicates and applies its own attribution model; Firestore captures raw event-level truth from the quote form.
- Resolution: feed Google Ads its own conversions via GA4 (it's the right loss function for the bid algo), but use the Firestore copy as the source of truth for board reporting and any commission accounting back to the dealer.
- The general principle: feed each system the version of the data its optimizer was built for; keep your own canonical copy for everything else.

## The kill switch matters
- Renaming `.env.upload-post` to `.disabled` halts all publishing instantly
- This was tested before the first real post; it should always be tested first
- The general principle: ship the kill switch before the feature

---

[← Cost routing](cost-routing.md) · [Up: README](../README.md)
