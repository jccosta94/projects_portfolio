# Skoda AI Ops
**A content-operations architecture for an automotive referral brand. Managed entirely from [Claude Cowork](https://claude.com/cowork) — the operator cockpit, the scheduled drafter (via Cowork Automations), and the approval surface, all in one app — with a deterministic publishing library between every LLM draft and every public API call.**

This is a case study in **AI-orchestrated content operations** — specifically: how to architect a production content engine for a small commercial brand when the marketing-team budget is zero, the compliance bar is real, the publishing surface spans six channels, and the operator runs the whole thing from a desktop AI cockpit rather than a custom dashboard.

The system runs the content pipeline for [Skoda Clever Kaufen](https://skoda-clever-kaufen.de), a referral business in the German automotive market. Production traffic to a real audience. The constraint that triggered the architecture work: a single operator can't produce the cadence the category expects (13 blog posts a quarter · weekly newsletter · weekly short-form video · daily social presence) and hiring a content team wasn't a viable option. My job was to design a system that closed the gap — without becoming a vehicle for compliance accidents the moment an LLM hallucinated the wrong word, and without burying the operator in a bespoke admin UI to manage it.

### What I architected

**Claude Cowork as the operator cockpit.** Everything starts there. Cowork is the desktop AI app — Anthropic Claude with MCP-tool access — and it's where I (the operator) sit all day to manage the business. Through one chat surface I author and review content, drive the publishing library, talk to Firebase, Gmail, Airtable, Gemini, Veo, and Excalidraw via their MCP servers, edit landing pages, generate Preisvorschlag PDFs, look up lead statuses, and trigger every gated publishing action. There is no separate admin dashboard. There is no scheduled-task UI. **Cowork is the dashboard, the editor, the orchestrator, and the keyboard for the whole content operation.** The architecture choices below all flow from that — including the choice to have *only one* dashboard surface so the operator never has to context-switch between tools.

**A three-generation evolution, with consolidation as the through-line.** v1 was a generic chat tab — no specialised cockpit, no library, no fail-closed guards — and broke within a week. v2 added a deterministic Python library that runs before every publishing API call, solving the compliance class of bugs, but the operator was still juggling separate tools for blog publishing, social scheduling, email, CRM, analytics, and landing-page deploys. v3 collapsed the entire operator surface into Cowork: one chat thread, one MCP fleet, one approval queue, one Automation scheduler. Along the way, multi-channel publishing collapsed behind a single fan-out API (Upload-Post), and Google Ads joined as a precision acquisition channel with dual-tracked attribution.

**The publishing library is the load-bearing piece.** Roughly 300 lines of Python plus tests. It checks every payload for German-language signals (permissive heuristic, only blocks affirmative-English-with-zero-German), price hedging (every `€` value must be preceded by `ab` / `bis zu` / `ca.` / `etwa`, or the whole text framed as `Beispielrechnung`), a regex denylist with word-boundary enforcement (`Festpreis`, `garantierter Preis`, bare `Rechnung` — but compounds like `Beispielrechnung` pass), and channel-specific voice rules (LinkedIn requires `wir`, blocks `ich`). **If any check fails the publishing API call does not fire.** The LLM is allowed to be wrong; the library refuses to publish until it isn't. Every Cowork session — interactive or scheduled — calls the library before any publish; the library is the only path to the publishing APIs.

**Cowork is the cockpit *and* the scheduler.** Interactive work happens in the Cowork chat — novel posts, editorial judgement calls, in-the-moment edits, anything that needs an operator at the keyboard. Scheduled work happens via [Cowork Automations](https://claude.com/cowork) — native recurring tasks the operator configures once and forgets about. Same Claude. Same skills library. Same MCP-tool fleet. Same publishing library on the way out. The weekly cadence (Tue blog · Wed newsletter · Thu/Fri/Sat social previews) is a set of Cowork Automation tasks that wake the agent on schedule, draft the content, run it through the library, and post a preview to Telegram. The operator opens Telegram, replies `ok` or `reject`, and a webhook trips the publishing call. **There is no separate scheduling runtime. There is no second agent. There is one Cowork, talking to everything.**

**Three human-in-the-loop gates per content cycle.** (1) Preview approval — every outbound post is reviewed in Telegram before the publishing API is called. (2) Channel-rotation discipline — Upload-Post's free tier caps at 10 uploads/month, which forced a routing policy (video → YouTube + Instagram; photos → Instagram + X; text → X; LinkedIn opt-in only). (3) CRM hand-off — a quote request enters Airtable and the dealer makes the call. The system stops automating at the dealer conversation, deliberately.

### The architectural insight

The through-line is **everything runs through one cockpit, and that cockpit is Claude Cowork.** The operator never leaves it. Through one chat thread, Cowork's MCP fleet reaches Firebase (for Hosting deploys, Firestore CRUD, Cloud Function logs), Gmail (for newsletter sends and dealer hand-off), Airtable (for the CRM), Gemini Nano Banana (for image generation), Veo 3.1 (for video), Excalidraw (for diagrams), the operator's own desktop (for one-off workflows), Cowork's native Automations scheduler (for the weekly cadence), and a custom MCP wrapper around the publishing library. The architectural bet was simple: **a single AI cockpit with the right MCP fleet replaces eight separate admin tools.** No bespoke CMS, no scheduler dashboard, no CRM frontend, no analytics dashboard, no landing-page editor. The conversational interface inside Cowork is the dashboard for everything. Without that consolidation the solo operator drowns in context-switching; with it, every action against the business happens in one place and the operator's leverage compounds.

The second through-line is **putting determinism where the LLM is unreliable.** Brand voice can live in a prompt; compliance cannot. The publishing library — ~300 lines of Python with regex and positive-context validation — sits between every draft Cowork produces and every outbound API call. Fail-closed by default. The library is not smart; it is right. Inside the cockpit, the library is the deterministic guard that lets the operator delegate to Claude without supervising every word, and lets compliance-sensitive content ship safely under regulated-industry rules.

The third through-line: **organic and paid are routed separately, but the system reads them as one funnel.** Organic content is the volume play, compounding over months at near-zero marginal cost. Google Ads is the precision channel for in-market searches the organic library hasn't yet earned. GA4 (Consent Mode v2, default-denied) feeds conversions back to Google Ads for bid optimisation; a Firestore `quote_events` collection captures the same events as source-of-truth attribution for commission accounting. The two will disagree. That is by design — feed each system the version of the data its optimiser was built for, keep your own canonical copy for everything else.

This repo documents the architecture of each generation, the topology I designed on top of off-the-shelf platforms (Firebase + Cowork + Upload-Post), per-job model routing decisions, the v1 → v2 → v3 evolution, and the alternative deployment options I considered along the way (including a headless agent on a VPS — see [Alternative deployment](#alternative-deployment--headless-agent-on-a-vps) below).

---

## TL;DR

| | **v1 — Prompt-gated** | **v2 — Library-gated** | **v3 — All ops in Cowork (current)** |
|---|---|---|---|
| **Solves which problem** | Nothing in production. Failed in <1 week. | **Compliance** — fail-closed on forbidden phrases and unhedged prices | **Operator consolidation** — one AI cockpit replaces 8 admin tools, plus cadence + distribution + acquisition layered on top |
| **Operator cockpit** | Generic Claude chat tab — no specialised surface | Generic Claude chat tab — library guards added, but the operator still juggles separate tools for blog, social, email, CRM, analytics | **[Claude Cowork](https://claude.com/cowork)** desktop — the entire business runs from one chat thread. Cowork is the dashboard, the editor, the scheduler, the CRM viewer, the analytics review, the PDF generator, the approval surface |
| **Scheduled work** | None | None | **Cowork Automations** — native recurring-task feature inside Cowork. Same Claude, same skills, same MCP fleet, same publishing library. No separate runtime. |
| **MCP fleet inside Cowork** | None | None | 8 servers — firebase · gmail · airtable · gemini-image · veo · excalidraw · computer-use · scheduled-tasks · custom skoda-mcp (publishing-library wrapper) |
| **Editorial control** | LLM system prompt (advisory at best) | Python library — regex + heuristics inspecting every payload pre-publish | Same library — called from Cowork on every publish, no agent bypasses it |
| **Approval gate** | None — agent publishes directly | Email approval (latency: hours) | **Telegram approval** (latency: minutes) — operator reads in Cowork, replies in Telegram |
| **Publishing surface** | Per-channel APIs (YT, IG, X, LinkedIn) wired individually | Same | **Single fan-out** via [Upload-Post API](https://upload-post.com) — 4 channels, 1 call |
| **Free-tier ceilings** | Direct API rate limits per channel | Same | Upload-Post: 10 uploads/month — forces rotation policy |
| **Acquisition** | Organic only | Organic only | **Organic + Google Ads** with dual-tracked attribution (GA4 + Firestore `quote_events`) |
| **Models — text** | Anthropic Claude | Same | Anthropic Claude primary · Ollama on VPS as resilience fallback only |
| **Models — image** | Mid-tier diffusion | Same | Google Gemini (Nano Banana) — order-of-magnitude cheaper at this volume |
| **Models — video** | None (manual) | None (manual) | Google Veo 3.1 — native synchronized audio · 9:16 + 16:9 |
| **Compliance posture** | 95% reliable on prompt rules — 5% regulatory exposure | 100% on library-enforced rules — soft errors in style only | Same fail-closed library, plus operator review inside Cowork for judgement calls |
| **What broke it / what survived** | Inconsistent output, compliance miss within a week — *killed* | Compliance solved, but the operator was still tool-juggling — *evolved* | Current — running |

The headline insight: **one AI cockpit, deterministic guards around it.** Every generation pulled more of the operator's surface into a single AI cockpit — landing on Claude Cowork in v3 — and pushed more of the load-bearing logic into deterministic layers around the LLM (the publishing library, the approval gate, the rotation policy, the dual-tracked attribution). The LLM is part of the system; the cockpit is the system.

---

## Why this matters

Most small commercial AI systems in 2026 hit four walls: **cockpit fragmentation first, then compliance, distribution, and attribution.**

**Wall 1 — Cockpit fragmentation.** A solo operator can't run a content business across eight separate admin tools: a CMS for the blog, a scheduler for social, a CRM for leads, an image tool for hero shots, a video editor for shorts, an email platform for newsletters, an analytics dashboard for GA4, and a landing-page builder. Each tool has its own login, its own mental model, its own dashboard, and most of them have nothing to say to each other. The context-switching cost alone defeats the small operation. **v3 solved this by collapsing the entire operator surface into Claude Cowork.** One AI cockpit, one chat thread, one MCP fleet reaching every external system. No bespoke admin UI — the conversational interface is the dashboard for everything. This is the foundational bet the rest of the architecture is built on; without it, none of the other walls could be solved at one-operator scale.

**Wall 2 — Compliance.** A regulated domain (German consumer protection, GDPR, automotive price advertising rules) is not forgiving of a 95%-reliable system. The wrong word in a published price claim isn't a writing error, it's an exposure. Prompt-based instructions to "avoid Festpreis" are advisory at best. The system has to refuse to publish content that violates the rules — not be asked nicely to comply. **v2 solved this by moving the rules into a library that runs before the publishing API call; v3 kept it as the deterministic guard layer inside the Cowork cockpit.**

**Wall 3 — Distribution.** Publishing the same piece of content to YouTube, Instagram, X, and LinkedIn from a one-person operation means maintaining four direct API integrations — four auth flows, four rate-limit profiles, four sets of edge cases. Engineering work that produces no business value. **v3 solved this by consolidating behind Upload-Post — single REST call, fan-out to all four.** The free tier's 10-uploads/month cap turned out to be a feature: it forced a rotation policy that prevents over-publishing.

**Wall 4 — Attribution.** The moment paid acquisition enters the picture (Google Ads), conversion data lives in two places — GA4 (which the Ads bidder consumes) and your own database. They will disagree. Treating either as the single source of truth eventually breaks the books. **v3 solved this with dual-tracked attribution**: GA4 → Google Ads for bid optimisation, Firestore `quote_events` for source-of-truth commission accounting.

The lesson is **the sequence.** Solve cockpit fragmentation first because everything else compounds on it — a fragmented operator surface makes every other problem harder. Solve compliance next, because shipping non-compliant content even once is a brand and legal event. Solve distribution next, because the per-channel integration work has no upside until you have content to distribute. Solve attribution last, because it only matters when you're spending money on acquisition.

The lesson is also **what stayed constant.** All three generations route through Firebase (Hosting + Functions + Firestore). All three use Airtable as the CRM. All three hand off to the dealer for the actual sale call. The internal architecture changed; the customer-facing surface and the lead-handling boundary didn't.

---

## v3 — the current architecture

Three flow charts cover the system end to end: the **orchestrator** (how content moves from idea to live post), the **lead funnel** (how a customer becomes a tracked lead in the CRM), and the **attribution split** (why GA4 and Firestore deliberately disagree). All diagrams in this repo are text-style ASCII inside fenced code blocks — readable on GitHub, version-controlled, diff-able, no rendering tooling required.

### The orchestrator

How a draft becomes a published post. Cowork sits at the top as the cockpit; both lanes (interactive + Automations) produce the same payload shape and feed into the same publishing library. Nothing fans out to the publishing surfaces unless the library passes it and the operator approves.

```text
                       Cowork (cockpit · operator's Mac)
                                  │
                                  ▼
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
       ┌─────────────┐                   ┌──────────────────────┐
       │ Interactive │                   │ Automations 📅       │   ← Cowork-native
       │    Lane     │                   │    Lane              │     scheduler
       │ (operator   │                   │ (5 weekly tasks:     │
       │  at the     │                   │  Tue blog · Wed news │
       │  keyboard)  │                   │  · Thu LI · Fri IG · │
       │             │                   │  Sat X)              │
       └──────┬──────┘                   └──────────┬───────────┘
              │                                     │
              └──────────────────┬──────────────────┘
                                 │ same payload shape
                                 ▼
                      ┌──────────────────────┐
                      │ Publishing Library 🔒│   ← ~300 LOC Python
                      │  German-lang check   │     fail-closed
                      │  Price-hedge regex   │     regex + heuristics
                      │  Denylist regex      │
                      │  Channel voice rules │
                      └──────────┬───────────┘
                                 │ pass (else: returns reason · retry loop)
                                 ▼
                      ┌──────────────────────┐
                      │ Telegram Preview ⏸  │   ← HUMAN GATE 1
                      │ (operator approves   │     latency: minutes
                      │  in Telegram)        │
                      └──────────┬───────────┘
                                 │ approved · library re-runs (defense in depth)
                                 ▼
                      ┌──────────────────────┐
                      │ Channel routing      │   ← HUMAN GATE 2
                      │ (10/mo Upload-Post   │     rotation policy
                      │  cap forces choice)  │
                      └──────────┬───────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌────────────┐    ┌──────────────┐    ┌──────────────┐
       │ Upload-Post│    │ /api/publish │    │ Gmail SMTP   │   ← three fan-out
       │ → YT · IG  │    │   Blog →     │    │ → newsletter │     endpoints,
       │   X · LI   │    │   Firestore  │    │   recipients │     one payload
       │   (gated)  │    │ → blog live  │    │              │
       └────────────┘    └──────────────┘    └──────────────┘
       terminal: live in audience feeds · audit in Firebase + Telegram
```

### The lead funnel

How a customer becomes a tracked lead and arrives in front of the dealer. Organic and Google Ads traffic hit the same landing page and share the same downstream pipeline; the only difference is the `lead source` field on the Airtable record. The system deliberately stops automating at the dealer call — the conversation that closes the sale is a human relationship, not a script.

```text
                       Customer (Germany · in-market)
                                  │
                                  │ organic OR google-ads
                                  ▼
                       ┌────────────────────────┐
                       │ /rabatt/<model>        │   ← Firebase Hosting
                       │  landing page          │     GA4 page_view fires
                       └────────────┬───────────┘
                                    │ submits quote form
                                    ▼
                       ┌────────────────────────┐
                       │ /api/quote             │   ← Cloud Function
                       │ (validate · write)     │     24h response SLA
                       └────────────┬───────────┘
                                    │
              ┌─────────────────────┴─────────────────────┐
              ▼                                           ▼
       ┌──────────────────┐                    ┌──────────────────────┐
       │ Firestore        │                    │ Auto-responder       │   ← Cloud Function
       │   quotes +       │                    │   + Preisvorschlag   │     · Gmail SMTP
       │   quote_events   │                    │     PDF              │
       │   (truth)        │                    │                      │
       └────────┬─────────┘                    └──────────┬───────────┘
                │ webhook                                 │
                ▼                                         ▼
       ┌──────────────────┐                    ┌──────────────────────┐
       │ Airtable CRM     │                    │ Customer inbox       │   ← within minutes
       │   Leads · Quotes │                    │   PDF in hand        │     first impression
       │   Commissions    │                    │                      │
       └────────┬─────────┘                    └──────────────────────┘
                │ lead source: organic | google-ads
                ▼
       ┌──────────────────┐
       │ Dealer 📞        │   ← HUMAN GATE 3
       │ Salesperson      │     automation ends here
       │ pulls + calls    │     dealer is seller of record
       └──────────────────┘
       terminal: sale conversation is out of scope
```

### The attribution split

GA4 and Firestore both capture every quote form submission. Each one is the right answer to a different question — GA4 is what Google Ads needs to optimise bids, Firestore is what the operator and the dealer need to settle commissions. Treating either as the single source of truth eventually breaks the books.

```text
                    Quote form fires (every submission)
                                  │
                                  ▼
              ┌───────────────────┴────────────────────┐
              ▼                                        ▼
       ┌──────────────────────┐               ┌──────────────────────┐
       │ GA4 event            │               │ Firestore            │
       │ (Consent Mode v2,    │               │   quote_events       │
       │  default-denied)     │               │   collection         │
       └──────────┬───────────┘               └──────────┬───────────┘
                  │                                       │
                  ▼                                       ▼
       ┌──────────────────────┐               ┌──────────────────────┐
       │ Google Ads           │               │ Commission accounting│
       │  conversion column   │               │   + board reporting  │
       │  (deduplicated)      │               │   (raw, untransformed)│
       └──────────────────────┘               └──────────────────────┘
       ← drives bid                          ← source of truth
         optimisation                          for the business

       The two will not agree. By design.
```

### Three human gates per content cycle

Three gates total, each catching a class of error the layer above can't:

| Gate | Where it sits | Catches |
|---|---|---|
| **Gate 1 — Preview approval (Telegram)** | In the orchestrator, between library and channel routing | Factually wrong claims about a model variant. Clumsy phrasing that's technically legal but reads badly. Redundancy with last week's post. |
| **Gate 2 — Rotation discipline** | In the orchestrator, between approval and fan-out | Over-publishing the same piece across every channel. Cap-aware discipline keeps reach quality high without burning the 10/mo Upload-Post cap. |
| **Gate 3 — Dealer hand-off** | In the lead funnel, between Airtable and the sale conversation | Anything that requires a relational sales conversation. Pricing nuance the dealer can discuss in real time. Trust the dealer relationship is building with the customer. |

**The agents propose; the operator disposes.** The library catches compliance violations deterministically. The operator catches judgement calls the library can't reason about. The dealer catches the actual sale because no system should try to automate a relational sales conversation in this category.

---

## v1 → v2 → v3 — what failed, what survived

### v1 — Generic chat tab, no cockpit, no library (week 1)

The obvious first version. One LLM in a generic chat tab with a long system prompt covering tone of voice, language rules, compliance phrases, channel-specific formatting, and schedule. Operator gives direction, agent does the rest. No specialised cockpit, no MCP fleet, no library between the agent and the publishing APIs.

**What failed:**
- **Inconsistency.** Same instructions produced subtly different outputs on different days. Prices were sometimes hedged with `ab`, sometimes not. Brand voice drifted.
- **Compliance.** A prompt instruction not to use `Festpreis` is honoured *most* of the time. Most isn't enough when the downside is a regulatory letter.
- **No fail-closed.** If the model decided the instruction was advisory, the post went out anyway.
- **No operator surface.** Even when the agent produced good drafts, the operator still had to copy them into a separate CMS, switch to a different tool to schedule, switch again to log the lead, switch again to check analytics. Eight separate tools, no consolidation.

**What survived:** the Claude reasoning-model choice, the Firebase + Airtable substrate, the dealer hand-off boundary.

**The lesson:** the operator surface matters as much as the model. A generic chat tab without scaffolding can't run a business; the system needs a real cockpit AND deterministic layers around it.

### v2 — Library bolted on, operator surface still fragmented

Added a thin Python library between the LLM and the publishing API. Library owns the rules; LLM produces drafts; publishing call only fires if the library says yes. The compliance problem went away.

**What this solved:**
- **Compliance class of bugs collapsed.** Library is the source of truth for what publishes; agent can't argue with regex.
- **New rules are 1-line additions.** A new forbidden phrase = one regex line, deployed in seconds.
- **The bug surface stayed small.** ~300 LOC plus tests.

**What still didn't work — and why this version still couldn't scale to a real one-operator business:**
- **No unified cockpit.** Posts were drafted in a generic chat tab, then the operator switched to a separate CMS to publish the blog, then to Airtable to log the lead, then to GA4 to check traffic, then back to the chat tab for the next post. Eight tools, eight logins, eight mental models. The operator spent more time context-switching than producing.
- **No scheduling.** Posts went out when the operator was at the keyboard. Nothing went out on weekends or evenings — exactly when social engagement peaks for this audience.
- **Cron-on-laptop is brittle.** Mac sleeps overnight; jobs miss.
- **Multi-channel publishing meant four direct integrations.** Disproportionate surface area for what it bought.

**The lesson:** solving compliance is necessary but nowhere near sufficient. Without consolidating the operator surface, the one-person economics still don't work.

### v3 — All ops in Cowork (current)

Four changes from v2:

**Adopted Claude Cowork as the operator cockpit.** Cowork is the desktop AI app — Anthropic Claude with MCP tools — and v3 leans into it as the only management surface for the whole business. Every content draft, every publishing trigger, every CRM edit, every PDF generation, every landing-page change happens inside Cowork through MCP tool calls. There is no bespoke admin UI, no separate scheduler dashboard, no custom CMS. The architectural bet: the operator's leverage compounds when every action against the system happens through one AI cockpit, not a swarm of half-built admin tools. **The operator stays in Cowork; Cowork talks to everything else.**

**Used Cowork Automations as the scheduler.** Cowork has a native scheduled-tasks feature — configure a recurring task once, Cowork wakes Claude on schedule, runs the task to completion, posts the output wherever you tell it to. The five weekly content jobs (blog Tuesday, newsletter Wednesday, social previews Thursday/Friday/Saturday) are Cowork Automation tasks. Each one drafts the content in the same Cowork context the operator uses interactively — same skills, same MCP tools, same publishing library on the way out — and posts a preview to the relevant Telegram approval channel. The operator opens Telegram, approves or rejects, and a webhook trips the publishing call. **No separate scheduling runtime exists.** The job that would have lived on a cron in a VPS instead lives as a recurring task inside the cockpit. One platform. One Claude. One context.

**Collapsed multi-channel publishing behind Upload-Post.** Single REST API fans out to YouTube + Instagram + X + LinkedIn. 10-uploads/month free-tier cap forced a rotation policy. The integration gotcha: `/api/upload_text` uses `title` as the post-body field, not `text` — silently returns 200 with empty post if you call it the obvious way. One-line dispatch fix, but the kind of integration bug that exists only because the contract differs from the obvious mental model.

**Bolted on Google Ads + dual attribution.** Google Ads catches in-market intent searches the organic library hasn't yet earned. GA4 (Consent Mode v2, default-denied) feeds conversions back to Google Ads for bid optimisation. A Firestore `quote_events` collection captures the same events as source-of-truth attribution. The two disagree by design — feed each system the version of the data its optimiser was built for.

**The Automation schedule lives inside a 10:00–21:00 DE window.** Not 24/7. Cowork runs on the operator's Mac, the Mac is off overnight, and missed-window jobs would fire on the next available slot. Scheduling within the operator's daily uptime guarantees the schedule actually runs. (If you wanted 24/7 — and were willing to take on a second runtime — see the [Alternative deployment](#alternative-deployment--headless-agent-on-a-vps) section below.)

---

## Alternative deployment — headless agent on a VPS

The current architecture runs the scheduled drafting inside Cowork via its native Automations feature. The same job could be done by a headless agent on a VPS — the architectural choice is whether you want one platform (Cowork) doing both interactive and scheduled work, or a split where a separate agent runs the cron-driven side independent of the operator's laptop uptime.

If you wanted the headless option, the obvious shape is [OpenClaw](https://openclaw.ai/) by [@steipete](https://x.com/steipete) deployed as a single-persona configuration in Docker on a small Hostinger VPS. Same platform I run as a 7-agent team for [the Psinest build pipeline](../openclaw-hermes-evolution/); here it would be one editorial drafter. The setup would look like:

- A Docker container running OpenClaw as a Node.js process under systemd, exposing its single persona via Telegram or webhook.
- The same publishing library bundled into the container so the agent calls the same compliance check before any publish.
- The same MCP server fleet (firebase, gmail, airtable, gemini-image, veo) re-pointed at the container — either by running stdio servers inside the container or by consuming the SSE versions over the network.
- Five cron jobs on the VPS (Tue blog · Wed newsletter · Thu/Fri/Sat social) firing on whatever schedule you want, including overnight.
- Every draft routed into the same Telegram approval channels the operator reads from inside Cowork. The operator's experience doesn't change — same approval queue, same `ok` / `reject` reply pattern.

**Trade-offs of the headless option:**

| | Cowork Automations (current) | Headless OpenClaw (alternative) |
|---|---|---|
| **Schedule window** | Bound by Mac uptime (~10:00–21:00 DE) | True 24/7 |
| **Runtimes to maintain** | One (Cowork desktop) | Two (Cowork desktop + VPS container) |
| **Hosting cost** | €0 — already running for interactive use | €5–€15/month VPS |
| **Operator surface** | One app (Cowork) | One app (Cowork) — same Telegram approval channels |
| **MCP-tool fleet** | Already configured in Cowork | Has to be re-configured / mirrored in container |
| **Failure modes** | Mac asleep = missed schedule | Container died = missed schedule, plus container needs systemd hygiene |
| **Code surface to audit** | Cowork's Automation tasks + skills | All of the above + Dockerfile + systemd unit + cron + OpenClaw config JSON |

The current bet is that single-platform simplicity wins at this scale. The operator's daily routine sits comfortably inside the 10:00–21:00 window; the schedule fits the routine. The day that stops being true — for example, if a high-engagement audience window opens at 04:00 DE that the current schedule can't reach — the headless alternative is one VPS provision and a config-mirror away.

This is the kind of "designed two options, picked the simpler one, kept the other warm in the diagram" pattern that small commercial systems benefit from. Documenting the alternative is part of the architecture work, not separate from it.

---

## Architecture diagrams

All diagrams in this repo are inline text-style ASCII inside fenced code blocks — readable on GitHub, version-controlled, diff-able, no rendering tooling required.

- **Orchestrator flow chart** — how a draft becomes a published post, both Cowork lanes feeding the same library + approval queue + fan-out. See [The orchestrator](#the-orchestrator) above.
- **Lead funnel flow chart** — how a public visitor becomes a tracked CRM lead, organic and paid both routed through the same landing → quote → Airtable → dealer pipeline. See [The lead funnel](#the-lead-funnel) above.
- **Attribution split flow chart** — why GA4 (for Google Ads bid optimisation) and Firestore (for commission accounting) deliberately disagree, and which optimiser consumes which copy. See [The attribution split](#the-attribution-split) above.
- **Three-human-gates table** — preview approval · channel-rotation discipline · CRM hand-off boundary. See [Three human gates per content cycle](#three-human-gates-per-content-cycle) above.

Not yet rendered: a media-generation flow (Gemini Nano Banana + Veo 3.1 pipeline) — will land as inline ASCII when the matching section of the case study is written.

---

## What I designed vs what the platforms provide

| Concern | Platform provides | My architectural contribution |
|---|---|---|
| **Operator cockpit** | [Claude Cowork](https://claude.com/cowork) — desktop AI app, Anthropic Claude with MCP-tool support, file-system access, computer-use | Configured Cowork as the *only* management surface for the business — wired in the MCP servers (firebase, gmail, airtable, gemini-image, veo, excalidraw, computer-use, custom skoda-mcp), authored the skill library (16 marketing skills) the operator invokes from inside Cowork, made the architectural bet to not build a separate admin UI |
| **Reasoning model** | [Anthropic Claude](https://www.anthropic.com/claude) (the model that powers Cowork and OpenClaw) | Per-job model routing — Claude for editorial, Gemini for image, Veo for video, Ollama as resilience fallback only |
| **Web hosting + serverless** | [Firebase](https://firebase.google.com) (Hosting, Cloud Functions, Firestore, Auth) | Trust-boundary scoping (single project), bearer-auth `/api/publishBlog`, Firestore `quote_events` as source-of-truth attribution collection |
| **Multi-channel publishing** | [Upload-Post](https://upload-post.com) — REST API fan-out to YouTube, Instagram, X, LinkedIn | Rotation policy (video → YT+IG; photo → IG+X; text → X), LinkedIn-gated config until Company Page reconnect, integration gotcha (`title` vs `text` field) documented in dispatch |
| **Image generation** | [Google Gemini](https://ai.google.dev) (Nano Banana model) via MCP | Selection (Nano Banana over premium tier — cost discipline), prompt patterns for editorial-grade automotive imagery |
| **Video generation** | [Google Veo 3.1](https://deepmind.google/models/veo/) via MCP | 9:16 vs 16:9 routing by destination, bed-reuse pattern when Veo errors (403/404), [`veo_interpolate_video`](https://deepmind.google/models/veo/) for before/after transitions |
| **Scheduled drafting** | [Cowork Automations](https://claude.com/cowork) — native recurring-task scheduler inside Cowork, same Claude / skills / MCP fleet as interactive sessions | Configured the five weekly content Automation tasks (Tue blog · Wed newsletter · Thu/Fri/Sat social previews), tuned the schedule to the 10:00–21:00 DE window, wired every Automation output through the publishing library and into the existing Telegram approval channels |
| **Compliance + voice rules** | None — Anthropic Claude allows system-prompt rules but doesn't enforce them | **The publishing library** — ~300 LOC Python · German-lang heuristic · price-hedge regex with positive context · denylist regex with word boundaries · channel-voice rules · fail-closed |
| **Approval gate** | Neither platform ships this | Telegram channel pattern (per-workflow), preview format with metadata footer, audit log of every approve/reject |
| **Attribution** | [GA4 Consent Mode v2](https://developers.google.com/tag-platform/security/concepts/consent-mode) — modeled conversions to Google Ads | Dual-track pattern — Firestore `quote_events` as source-of-truth, GA4 as bid-optimisation feed, deliberate divergence documented |
| **CRM** | [Airtable](https://airtable.com) — base + view + automations | Schema (Leads · Quotes · Commissions tables), lead-source field, dealer-hand-off view |
| **Dealer hand-off boundary** | N/A — this is a business-process decision | Made it the explicit stopping point of automation — system delivers a clean lead, dealer makes the call, no SMS chatter, no automated nurture |

The platforms do the heavy lifting; the editorial pipeline (library + approval gate + rotation policy + attribution model) and the operational architecture (two-agent split + 10:00–21:00 window + dealer-hand-off boundary) are mine.

---

## Stack

| Layer | Component |
|---|---|
| **Operator cockpit** | **[Claude Cowork](https://claude.com/cowork)** desktop app — Anthropic Claude (Sonnet/Opus) · MCP server fleet (Firebase, Gmail, Airtable, Gemini Image, Veo, Excalidraw, computer-use, scheduled-tasks, custom skoda-mcp) · skills library (16 marketing skills) · the *only* surface the operator uses to manage the business |
| **Scheduled drafting** | **Cowork Automations** — native recurring-task feature inside Cowork · 5 weekly content tasks (blog · newsletter · LinkedIn · Instagram · X) · 10:00–21:00 DE schedule window · every Automation output passes through the same publishing library and Telegram approval channels as interactive drafts |
| **Customer-facing web** | Firebase Hosting · Cloud Functions (`/api/quote`, `/api/publishBlog`, auto-responder) · Cloud Firestore |
| **Analytics + paid acquisition** | GA4 with Consent Mode v2 (default-denied) · Google Ads (search + Performance Max) · Firestore `quote_events` source-of-truth attribution |
| **Content production** | [Google Gemini Nano Banana](https://ai.google.dev) (images, via MCP) · [Google Veo 3.1](https://deepmind.google/models/veo/) (video, 9:16 + 16:9 native audio) |
| **Publishing surface** | [Upload-Post](https://upload-post.com) REST API → YouTube · Instagram · X (`@SkodaCleverKauf`) · LinkedIn (gated) · Gmail SMTP (newsletter) · Cloud Function `/api/publishBlog` (blog) |
| **Editorial control** | Python publishing library (~300 LOC) — language check · price-hedge regex · denylist · channel voice rules · fail-closed · runs at draft-time AND at final-payload-time |
| **Approval gate** | Telegram channels (`blog-approval` · `social-approval` · `newsletter-approval`) with webhook back into publishing layer |
| **CRM** | [Airtable](https://airtable.com) workspace `Skoda Referrals` — Leads · Quotes · Commissions · dealer-hand-off view |
| **Observability** | Firebase Functions logs · Telegram error alerts (per-cron-job ID) · Upload-Post usage tracker (warns at 8/10 monthly cap) |
| **Kill switch** | `.env.upload-post` → `.env.upload-post.disabled` — halts all publishing instantly; script refuses to run without the key |

---

## Deeper reading

### Case study (the v1 → v2 → v3 story)
- [`case-study/business-problem.md`](./case-study/business-problem.md) — the one-operator constraint, why a traditional content stack doesn't solve it, why paid + organic both belong in the mix
- [`case-study/architecture-evolution.md`](./case-study/architecture-evolution.md) — v1 → v2 → v3 narrative; each version's failure mode explains the next version's choices
- [`case-study/orchestration-design.md`](./case-study/orchestration-design.md) — the technical centerpiece: two-agent split, library as hard layer, Telegram gate, single-fan-out trade-off, hand-off boundary
- [`case-study/ai-stack.md`](./case-study/ai-stack.md) — model routing by job, why MCP is the integration plane, what's not in the stack and why
- [`case-study/cost-routing.md`](./case-study/cost-routing.md) — production vs acquisition cost buckets, organic-as-volume vs Google-Ads-as-precision, what gets cut first
- [`case-study/lessons-learned.md`](./case-study/lessons-learned.md) — prompt-gated guardrails don't survive contact with regulation · "ab Lager" is not a price hedge · attribution as directional vs gospel · the kill switch matters

### Workflows (how a piece of content moves through the pipeline)
- [`workflows/blog-generation.md`](./workflows/blog-generation.md) — cron-triggered weekly draft → image gen → library → Telegram preview → `/api/publishBlog` → Firestore
- [`workflows/social-post-pipeline.md`](./workflows/social-post-pipeline.md) — caption + media → library → preview → Upload-Post fan-out
- [`workflows/veo-video-workflow.md`](./workflows/veo-video-workflow.md) — when to generate fresh, when to reuse beds, cost discipline
- [`workflows/approval-routing.md`](./workflows/approval-routing.md) — Telegram channels per workflow class, what gets rejected, audit trail
- [`workflows/publishing-orchestration.md`](./workflows/publishing-orchestration.md) — approved-draft → final-payload → library-rerun → Upload-Post call

### Orchestration deep-dives
- [`orchestration/mcp-integration.md`](./orchestration/mcp-integration.md) — the MCP servers in the system, stdio vs SSE transports, the launchd-supervised pattern
- [`orchestration/agent-routing.md`](./orchestration/agent-routing.md) — Cowork's interactive lane vs Automation lane, what they share, why a single cockpit beat the two-runtime alternative
- [`orchestration/scheduling-logic.md`](./orchestration/scheduling-logic.md) — Cowork Automations configuration, the 10:00–21:00 schedule window, the 5 weekly content jobs, missed-window handling
- [`orchestration/model-selection.md`](./orchestration/model-selection.md) — routing by job, why local models are fallback-only, what would change the stack

---

## Status

- ✅ **v3 — current production architecture.** Cowork cockpit operational. Cowork Automations configured for the five weekly content tasks. Publishing library deployed. Telegram approval gate active. Upload-Post integration live (first post 2026-05-05). Google Ads campaigns running. Dual attribution feeding both bidders and books.
- ✅ **Architecture diagrams.** Orchestrator flow chart · lead funnel · attribution split · three-human-gates table · deployment view — all shipped as inline text-style ASCII in fenced code blocks. Media-generation flow (Gemini + Veo pipeline) pending.
- ✅ **Case studies (4/6).** Business problem · architecture evolution · orchestration design · AI stack all in full prose. Cost routing + lessons learned in structured-outline form.
- 🚧 **Workflows + orchestration deep-dives (9 files).** Outlined; full prose pending.
- 🚧 **Partial code examples.** Publishing library snippets + Cloud Function excerpts pending sanitisation for public release.
- 🚧 **LinkedIn re-enablement.** Currently gated in publishing library — Upload-Post is connected to a personal LinkedIn profile, not the Company Page. Reconnect required in Upload-Post dashboard before enabling.

---

## Acknowledgements

The platforms this case study is built on:
- **[Claude Cowork](https://claude.com/cowork)** by [Anthropic](https://www.anthropic.com) — the desktop AI app that serves as the operator cockpit *and* the scheduled drafter (via Cowork Automations). Cowork is where the entire content business is managed from: drafting, reviewing, publishing, CRM editing, PDF generation, landing-page deploys, analytics review, approval-gate handling, and the recurring weekly content cadence. The architectural bet that "the operator stays in Cowork, Cowork talks to everything else, Cowork's own scheduler runs the cadence" is what made the whole single-operator model viable on a single platform.
- **[Anthropic Claude](https://www.anthropic.com/claude)** (Sonnet 4.6 / Opus 4.6) — the reasoning model powering every Cowork session, interactive and scheduled, and every editorial draft
- **[Upload-Post](https://upload-post.com)** — single-fan-out multi-channel publishing API
- **[Firebase](https://firebase.google.com)** — Hosting, Cloud Functions, Firestore, Auth
- **[Google Gemini](https://ai.google.dev) (Nano Banana)** + **[Veo 3.1](https://deepmind.google/models/veo/)** — image and video generation via MCP
- **[Airtable](https://airtable.com)** — CRM substrate

The alternative deployment ([Alternative deployment — headless agent on a VPS](#alternative-deployment--headless-agent-on-a-vps)) sketches how the same scheduled work could run on **[OpenClaw](https://openclaw.ai/)** by [@steipete](https://x.com/steipete) — the agent gateway I run as a 7-agent team for [the Psinest build pipeline](../openclaw-hermes-evolution/), here in single-persona configuration. The headless option is documented but not adopted; the current single-platform bet wins on simplicity at this scale.

The editorial-pipeline pattern — library-as-hard-layer for compliance, deterministic guards around a probabilistic model, human-in-the-loop only where judgement is required — is borrowed from how regulated industries already manage editorial workflows. **AI systems work best when they mirror the practices that already work in the regulated domains they enter.**

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com) · 🌐 [skoda-clever-kaufen.de](https://skoda-clever-kaufen.de) · 🐙 [github.com/jccosta94](https://github.com/jccosta94)
