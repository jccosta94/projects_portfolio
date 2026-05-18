# Scheduling logic

> **Status:** outline draft.

## The cron window: 10:00–21:00 DE
- Why not 24/7: the operator's Mac is off overnight, and Cowork-initiated work needs the operator awake.
- Why not 09:00–22:00: avoid edge cases where a job fires right as the operator opens or closes the laptop.

## The five weekly content jobs
- Blog preview → Tuesday morning
- Newsletter preview → Wednesday morning
- LinkedIn preview → Thursday afternoon
- Instagram draft → Friday morning
- X draft → Saturday morning

## Each job's contract
- Produces a preview to Telegram, never publishes.
- Tagged with the cron job ID for traceability.
- Failure routes to the operator with a one-line post-mortem.

## What happens on a missed window
- Jobs are idempotent — re-running on the next window is safe.
- No backfill. If Tuesday's blog draft missed, Wednesday will not also do Tuesday's.
