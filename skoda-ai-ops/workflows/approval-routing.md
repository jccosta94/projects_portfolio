# Workflow: Approval routing

> **Status:** outline draft.

## The Telegram gate
- One channel per workflow class: `blog-approval`, `social-approval`, `newsletter-approval`.
- Preview format: image + caption + metadata footer (cron job ID, draft timestamp, guardrail check results).

## What gets approved automatically
- Nothing.

## What gets rejected
- Operator replies `reject` or `skip`. Draft moved to Firestore `rejected` collection with reason.
- After three rejects in a 24h window, the orchestrator pauses the originating cron job and alerts.

## What approvals look like from the operator side
- Reply `ok` or 👍 in the Telegram channel.
- The publishing layer picks up the reply via webhook and fires the actual API call.

## Audit trail
- Every approval + rejection is logged with timestamp, operator user ID, and the diff that was approved.
