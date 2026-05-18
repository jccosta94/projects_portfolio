# Workflow: Publishing orchestration

> **Status:** outline draft.

## The flow from approved draft to live post
1. Approved payload pulled from Firestore `approved` collection.
2. Channel routing applied (video → YT+IG, photo → IG+X, text → X).
3. Guardrail library re-runs on the final payload (defense in depth — the approval might have been on a stale version).
4. Upload-Post POST fired.
5. Response captured: post URL(s), usage count, channel-specific IDs.
6. Result written back to Firestore `published` collection and Airtable for cross-referencing with leads.

## Why the guardrails run twice
- An operator might approve, then the orchestrator might assemble a slightly different final payload (re-encoded image, regenerated caption).
- The second check is cheap and catches drift.

## What this workflow does NOT do
- It does not retry on 4xx. If Upload-Post says no, the draft is preserved and the operator is alerted.
- It does not silently re-queue. Every retry is operator-initiated.
