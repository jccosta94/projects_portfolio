# Orchestration design

The orchestration layer is where a useful generative system becomes a publishable one. The design points are not original — most of them are visible in any production agent system — but the specific choices made here were forced by the constraint of a one-person operator running a regulated commercial business with no on-call rotation.

## Two agents, one editorial pipeline

The system runs two agents: an interactive one and a scheduled one. The interactive agent is Anthropic Claude running in Cowork, the desktop application, on the operator's Mac. It handles the in-the-moment work — editorial decisions, novel posts, anything where the operator wants to think out loud and shape the output. The scheduled agent is OpenClaw, a Dockerized container running on a low-cost VPS, configured against the Anthropic API directly and supervised by cron. It handles the weekly cadence: blog drafts on Tuesday, newsletter drafts on Wednesday, social previews on Thursday through Saturday.

The reason for two agents rather than one is reliability, not capability. The desktop agent is the right place for interactive work because the operator is there to react to it. It is the wrong place for scheduled work because the laptop closes at the end of the day. The scheduled agent is the right place for cadenced content because the VPS is awake at all hours. It is the wrong place for novel work because there is no operator next to it to make a judgment call when the model gets something subtly off.

The seam between the two is the editorial pipeline. Both agents produce the same shape of payload, both run it through the same publishing library, both route it through the same Telegram approval channel. From the operator's perspective there is one approval queue, regardless of which agent drafted what. From the system's perspective there is one set of rules. The agents are interchangeable on the output side, which is what makes the split sustainable — the operator does not have to remember which agent made which thing.

## The publishing library as a hard layer

A draft that an agent produces is a string of text plus a media file. A post that goes out the door is a payload that has been inspected, hedged, language-checked, denylist-screened, and channel-formatted. The transformation between the two happens in a small Python library — perhaps three hundred lines, less than a thousand including tests — that the agent has to call to publish anything.

The library does four things. It checks that the text reads as German, using a permissive heuristic. It checks that every `€` price value is hedged with `ab`, `bis zu`, `ca.`, or `etwa`, or that the whole text is framed as a `Beispielrechnung`. It runs a regex denylist that blocks compliance-sensitive terms like `Festpreis`, `garantierter Preis`, and the bare word `Rechnung` (without blocking the compound `Beispielrechnung`, which the word-boundary regex handles correctly). It enforces channel-specific voice — LinkedIn requires the first-person plural `wir` and blocks `ich`, because LinkedIn posts publish from a company page rather than a personal one.

If any check fails, the library raises and the publishing call does not fire. The agent receives the failure reason and can revise; the operator receives a Telegram notification. There is no override.

The choice to put these rules in code rather than in the agent's system prompt is the single most important design decision in the whole system. The early version of this stack relied on prompt instructions — "do not use the word Festpreis", "always hedge prices with ab" — and it worked roughly 95% of the time. The remaining 5% is a regulatory exposure that a small business cannot afford. A regex with positive context is 100%. The library is not smart; it is right.

## The Telegram approval gate

Every outbound post passes through a Telegram channel before the publishing API is called. The orchestrator drafts the content, runs it through the library, generates a preview, and posts the preview to a per-workflow channel — `blog-approval`, `social-approval`, `newsletter-approval`. The operator replies with an approval (or a rejection) in the channel. The publishing layer picks up the reply via webhook and either fires the actual API call or shelves the draft.

The reason for Telegram and not email is latency. Email lives in a tab the operator might not have open; Telegram lives on the phone in the operator's pocket. Approval times dropped from hours to minutes when the gate moved from email to Telegram, which is the difference between a system that posts within the editorial window and one that does not.

The reason for an approval gate at all is that the cost of a wrong post is asymmetric. A missed post is a missed opportunity; a bad post is a brand problem, and in this domain it can be a compliance problem. The expected value of operator review at this volume — somewhere under a dozen approvals per week — is comfortably positive.

The gate is also where the operator catches things the library cannot. The library will not catch a factually wrong claim about a Škoda model variant; it will not catch a clumsy phrase that is technically allowed but reads badly; it will not catch a piece of content that is fine in isolation but redundant with last week's post. Those are judgment calls, and the system is designed to route them to the operator rather than try to automate them.

## Single fan-out via Upload-Post

Multi-channel social publishing is a wiring problem. YouTube, Instagram, X, and LinkedIn each have their own API, their own auth flow, their own rate limits, their own quirks around video encoding and image aspect ratios. Maintaining four direct integrations for a one-person business is the kind of work that produces no business value and eats every weekend.

The decision was to consolidate behind a single REST API — Upload-Post — that fans out to all four. The publisher makes one call with the media file and the caption, specifies which channels to target, and gets back the post URLs. The free tier caps at ten uploads per month, which is a real constraint, but it forced a discipline of rotation: not every post goes to every channel. Video goes to YouTube and Instagram. Photos go to Instagram and X. Text-only posts go to X. The cap is not a limit on what gets said; it is a limit on how broadly each thing gets repeated.

The trade-off is loss of fine control. If Instagram changes how it handles 9:16 aspect ratios next month and the fan-out service does not adapt in time, posts to Instagram are broken until the service catches up. The bet is that for a small business this is a better trade than maintaining four integrations. The bet has held so far.

The integration carries one production gotcha that is worth documenting because it cost an afternoon to discover. The Upload-Post `/api/upload_text` endpoint uses `title` as the post-body field, not `text` — the parameter naming is ambiguous in the docs and the wrong field silently returns a 200 with an empty post. The fix is one line in the dispatch code, but it is the kind of integration bug that exists only because the contract differs from the obvious mental model.

## CRM hand-off boundary

The system stops automating at the dealer call. A quote request lands in Firestore via a Cloud Function, triggers an auto-responder with a personalized Preisvorschlag PDF, and is written to the Airtable CRM with all the metadata — model configuration, customer contact details, lead source (organic vs Google Ads), timestamp, and a Firestore record ID for cross-referencing. The dealer pulls the lead from Airtable and calls the customer.

The reason this is the boundary is that the value created in the dealer conversation is not a function of automation. A good dealer relationship — the discount available, the trust the customer feels, the willingness to follow up if the customer wants to think about it for a few days — is a human asset, and trying to wrap it in scripts or automated SMS would degrade it. The system's job ends when the lead is delivered cleanly and quickly; the system's job does not extend to talking to the customer on the dealer's behalf.

This boundary has the secondary benefit of keeping the system simple to audit. Anyone reviewing the architecture can see exactly where the AI work stops, which is a useful property when the question "is this compliant" gets asked.

## What this design is not

It is not autonomous. There is a human approval gate in the middle of every outbound flow, and that is intentional. A system that publishes without approval has to be right every time; this system has to be right most of the time, with a reviewer catching the rest.

It is not stateless. The library carries denylist regexes, channel-voice rules, and rotation logic. The agents carry skills. The publishing layer carries usage counters. All of this state is small enough to fit in a single repository and a single Firestore project, which keeps the audit surface tractable.

It is not optimized for scale. If the business grew to ten operators, the Telegram channel approach would not work and the publishing library would need a proper admin UI. The system is built for the operating model it has, not the one it might have in two years. When that bridge needs crossing, it can be crossed; until then, the simplest thing that works is the right thing.

---

[← Architecture evolution](architecture-evolution.md) · [Up: README](../README.md) · [Next: AI stack →](ai-stack.md)
