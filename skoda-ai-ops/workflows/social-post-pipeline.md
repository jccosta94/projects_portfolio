# Workflow: Social post pipeline

> **Status:** outline draft.

## Trigger
- Weekly cron (one job per channel cadence) or manual from Cowork desktop.

## Pipeline
1. Topic + format chosen from rotation (video / photo / text).
2. Draft caption (German) + media (Veo or Gemini, depending on format).
3. Guardrail library check.
4. Preview → Telegram approval gate.
5. On approval → `scripts/upload_post/publish.py --media ... --caption ...` → Upload-Post API → fan-out.

## Channel rotation
- Video → YouTube + Instagram
- Photo → Instagram + X
- Text → X only
- LinkedIn → opt-in (gated until Company Page reconnected)

## Failure modes
- Caption fails German check or price hedge → operator decides.
- Upload-Post API 4xx → captured in response; usage count still consumed if partial success.
- Approaching 8/10 monthly cap → warn operator before queuing more.
