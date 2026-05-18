# Hugo — AI Marketing Employee for SMBs

**Live commercial product.** A single-agent AI marketing employee for small businesses. Talks to owners via Telegram / Slack / WhatsApp / email and runs their ads, website, creative, Google Business profile, reviews, outreach, and weekly reports — with the owner approving important work before it goes live.

🌐 [hugo-website (marketing site)](https://jccosta94.github.io/hugo-website/) · 🤖 [`@steipete`'s OpenClaw](https://openclaw.ai/) under the hood · 💬 multi-channel I/O

---

## What Hugo is *for*

Small businesses don't usually need another marketing tool. They need **someone to do the work**. The owner runs a service business; marketing is the thing that always slips. Hugo is the in-house marketer they couldn't justify hiring.

**Replaces, depending on alternative:**

| Alternative | Typical monthly cost | Problem with it |
|---|---|---|
| In-house marketer | €2,500–4,500 | One person, can't cover every channel well |
| Fractional CMO | €3,000–8,000 | Good strategy, not day-to-day execution |
| Digital agency | €2,000–10,000 | Slow, project-based, layers of account managers |
| **Hugo** | Lean monthly + setup + your ad budget on your accounts | One proactive AI, approval-gated, multi-channel |

The interesting architectural decision is **scope**: Hugo isn't a marketing assistant ("here's a draft post for you to edit") and isn't a marketing platform ("here's a dashboard, you configure campaigns"). It's a **marketing employee** — Hugo finds the next useful thing to do, prepares it, and asks the owner to approve. Same shape as a junior marketer reporting to the owner.

---

## Architecture

### Single-agent persona, multi-channel I/O

```
                    SMB OWNER (the human)
                          │
                          │ talks naturally
                          ▼
        ┌──────────────────────────────────────┐
        │  Hugo                                 │
        │  ─────                                │
        │  Single agent persona                 │
        │  Persistent business context per      │
        │   client (industry, services, ads,    │
        │   site, budget limits, brand voice)   │
        └────────────┬─────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
    Telegram      Slack         Email             ◄── client chooses any channel
    WhatsApp                                          (Hugo replies on the same one)
```

**One agent, not a team** — different from OpenClaw's hierarchical 7-agent setup ([see the OpenClaw-Hermes Evolution case study](../openclaw-hermes-evolution/)). For a marketing context, a single agent with strong persistent context and clear approval gates outperforms a multi-agent team. The owner shouldn't have to think about "who handled my ad campaign last week?" — it's always Hugo.

### Approval-gated execution

The defining architectural pattern. Every "important" action — new campaign, public post, page change, Google Business update — is **prepared by Hugo and approved by the owner** before it goes live.

```
1. Hugo finds work to do
   (reviews website, ads, profile, reviews, calendar)
              │
              ▼
2. Hugo prepares the work
   (writes ad copy, drafts page, generates creative)
              │
              ▼
3. Hugo presents to owner via chat:
   "Here's the next step. Cost: €X/day. Risk: Low.
    Reply APPROVE to launch."
              │
              ▼
   ┌──────────┴──────────┐
   │                     │
   APPROVE              REDIRECT
   │                     │ ("not yet" / "do X instead")
   ▼                     ▼
4a. Hugo launches      4b. Hugo backs off
   (Google Ads, Meta,    or re-scopes
    site update, etc.)
              │
              ▼
5. Hugo monitors + reports back at next checkpoint
```

This pattern is why Hugo can hold real account credentials (Google Ads, Meta Business Manager, Google Business Profile, WordPress / website CMS) without the owner losing control: **Hugo operates on the accounts, but doesn't decide what to spend on or what to publish without approval**.

### Weekly cadence orchestration

Hugo's clock isn't event-driven — it's a weekly cadence baked into the orchestration:

| Day | Hugo's action | Owner's role |
|---|---|---|
| **Monday** | "Here's what we should focus on this week." Surfaces 1–3 actions from the audit + plan | Approves or redirects |
| **Midweek** | "Here's what I noticed. Nothing urgent" — or — "I need approval on X." | Approves blocking items only |
| **Friday** | "Here are the results, spend, leads, next actions." | Reads (no action required) |
| **Monthly** | Wider review: bigger-picture trends + plan update | Strategy conversation |

That cadence is what turns "we should do more marketing" into "we did X this week, here's what shipped."

---

## Capabilities

What Hugo actually does for a client, by channel:

| Area | Hugo does | Hugo doesn't do |
|---|---|---|
| **Ads** (Google + Meta) | Campaign setup, ad text, creative previews, budget checks, spend monitoring | Run campaigns at scale without approval; touch the credit card |
| **Website** | Offer pages, service pages, CTAs, campaign landing pages | Major site redesigns without sign-off |
| **Creative** | Adapt brand assets for ads/posts, generate variants | Source new photography |
| **Google Business Profile** | Profile fixes, review drafts, photo updates, local trust improvements | Respond to reviews without approval |
| **Reviews + outreach** | Drafts replies, drafts outreach to past customers | Send without owner OK on the message |
| **Reports** | Weekly numbers — spend, leads, what changed, what to do next | Pretend numbers are better than they are |

---

## Stack

| Layer | Technology |
|---|---|
| **Underlying agent platform** | [OpenClaw](https://openclaw.ai/) by [@steipete](https://x.com/steipete) — same chat-based agent gateway as the dev pipeline in [openclaw-hermes-evolution](../openclaw-hermes-evolution/), differently configured (single persona vs 7-agent team) |
| **LLM brain** | Anthropic API · per-task model routing (heavier model for strategy + ad copy, lighter for routine status replies) |
| **Messaging I/O** | Telegram Bot API · Slack API · WhatsApp Business API · Email (IMAP/SMTP) |
| **Ad platforms** | Google Ads API · Meta Marketing API |
| **Local presence** | Google Business Profile API |
| **Image generation** | Anthropic / external image models (creative variants) |
| **Persistent context** | Per-client memory store (business profile, brand voice, budget limits, approval history) |
| **Approval log** | Audit trail of every "what was approved when by whom" — important for accountability when an owner asks "why did we launch this?" 3 months later |
| **Hosting** | Runs alongside the same agent infrastructure as Psinest's build pipeline (cost-shared infra) |

---

## Architecture decisions worth surfacing

These are the design choices that distinguish Hugo from a generic "AI marketing tool":

### 1. Client ad spend stays on client's own Google / Meta accounts
Hugo gets account-write access via OAuth invite. **Hugo never holds the client's ad money.** This is a deliberate trust + financial-separation pattern:
- Client's card pays Google / Meta directly
- Hugo operates the accounts on the client's behalf
- If the engagement ends, the client revokes access — no migration of historical campaign data

### 2. Budget-limit enforcement gates before any spending action
Every campaign or ad-spend change passes through a hard budget cap configured at onboarding. **Hugo cannot exceed the cap autonomously**, even with owner approval-fatigue. If a campaign would exceed the cap, Hugo explicitly asks for a cap raise as a separate decision from "launch this campaign."

### 3. Per-task model selection
Different content types call for different models. Strategic plan + ad copy + brand voice → premium model. Quick "here's the update" message in chat → fast cheap model. The persistent context (client business profile, brand voice file, prior approvals) is the same; what changes is which model generates the output for each task.

### 4. Multi-channel I/O without channel coupling
The client picks the channel — Telegram, Slack, WhatsApp, or email. Hugo replies on the same channel. **The agent itself doesn't care about the channel**; the gateway routes inbound messages to Hugo and routes Hugo's outbound replies back to the same channel. Switching channels (e.g. client travels and starts using WhatsApp instead of Slack) is seamless.

### 5. Single-agent over multi-agent (for this domain)
The OpenClaw build pipeline uses 7 agents in a hierarchical org chart because **producing code** has clear sub-roles (PM scopes, Dev writes, QA tests). Marketing for a small business is different — the owner has one relationship, not seven. A single Hugo with rich persistent context and clear approval gates is a better fit than a team. The trade-off: less parallel work per client, but higher coherence and lower owner cognitive load.

---

## Live site

[**jccosta94.github.io/hugo-website**](https://jccosta94.github.io/hugo-website/) — the public marketing site for Hugo as a productized service. Shows the value prop, the alternatives comparison, and the "talk to a person on your team" UX framing.

---

## What's not in this repo

The Hugo *application code* — credentials, prompts, client onboarding data, approval logs — lives in a private repo (`jccosta94/hugo-app`). This case study documents the **architecture and operational pattern**, not the implementation source. That separation is deliberate: the pattern is reusable; the specific client configurations and credentials aren't.

---

## Related case studies in this portfolio

- [openclaw-hermes-evolution](../openclaw-hermes-evolution/) — same OpenClaw platform underneath, but configured as a 7-agent dev team instead of a single marketing agent. Useful read for understanding *why* the single-agent shape works better for Hugo's domain.
- [psinest](../psinest/) — the healthcare CRM that Hugo's underlying OpenClaw build pipeline produced. Different domain, different agent topology, same platform.
- [shared-architecture-patterns](../shared-architecture-patterns/) — approval-routing, multi-channel I/O, single-agent vs multi-agent decision frameworks documented as reusable patterns.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com) · 🌐 [hugo-website](https://jccosta94.github.io/hugo-website/)
