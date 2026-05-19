# João Costa
## AI Workflow Automation & Applied AI Systems

Functional Salesforce consultant and business analyst transitioning into AI systems and workflow automation.

Over the past months, I have been building and experimenting with AI-powered workflow systems, multi-agent orchestration setups, and operational automations across healthcare, CRM workflows, marketing operations, and internal business processes.

My work focuses on combining LLMs, automation tooling, orchestration workflows, and business operations into practical AI-enabled systems using Claude API, Claude Code, Gemini, MCP tooling, and workflow automation platforms.

Projects and areas explored include:

- AI-native CRM workflows
- Multi-agent orchestration systems
- AI-assisted operational workflows
- Workflow automation & copilots
- AI marketing operations
- Retrieval and orchestration concepts
- Multi-model routing strategies
- Local/private AI experimentation using Ollama

Alongside my AI projects, I bring several years of experience working in stakeholder-heavy enterprise environments across Salesforce consulting, CRM delivery, requirements engineering, UAT coordination, release execution, and cross-functional collaboration.

Current focus areas:

- AI workflow automation
- Agentic systems & orchestration
- Operational AI workflows
- AI deployment concepts
- Retrieval workflows
- Human-in-the-loop systems
- AI tooling ecosystems

This GitHub serves as a portfolio of AI systems, workflow experiments, operational automations, and applied AI projects exploring practical AI-enabled business operations.

---

## What's here

### Productized AI services

#### 🤖 [Hugo](./hugo/) — AI marketing employee for SMBs
**Live commercial product · first customer onboarding.** Architected an AI marketing employee that talks to small-business owners via Telegram / Slack / WhatsApp / email and runs their ads, website, creative, Google Business profile, reviews, outreach, and weekly reports. **Approval-gated workflow:** Hugo prepares work, owner approves via chat reply, Hugo launches.
**Two architectural generations — same lesson family as [Psinest's build pipeline](./openclaw-hermes-evolution/), different constraint axis:**
- **v1 — 7-agent multi-agent team** (Director + Performance Marketer + Web Developer + Brand Designer + Community Manager + Outbound Specialist + Analytics). Same hierarchical pattern that worked for Psinest's build system, applied to the marketing surface. **Broke under cost-tier model capability** — `gpt-5.4-mini` (cheap enough to make per-customer ChatGPT Plus subscriptions viable) doesn't bind soft procedural framing. Three independent multi-agent delegation failures traced to the same root cause: when the model has an escape valve (ask the customer, fabricate plausibly, replay from session memory), it takes the escape rather than executing the prescribed multi-step delegation.
- **v2 — single Hugo agent + specialists-as-MCP-tools** on [Hermes Agent](https://hermes-agent.nousresearch.com/) (open source, by [Nous Research](https://nousresearch.com)). Specialist capabilities collapsed from separate agents to area-scoped MCP tool servers (`marketing__*`, `creative__*`, `web__*`, `inbox__*`, `outbound__*`, `analytics__*`). **Pattern C**, the canonical Hugo skill template: each tool wraps deterministic mechanical work; the agent's only job is to synthesize the customer-facing summary from the tool's return value. Multi-step procedural contracts are gone; the failure mode that broke v1 has no surface to attach to. **As a side effect, v2 is materially cheaper to operate**: 5–7 inference calls per customer turn collapsed to 1–2, with deterministic work moved into Python (where it costs nothing).

**The headline lesson — model-capability is a topology constraint.** When the cheap model can't bind multi-step delegation, change the topology to remove the multi-step contract — don't wait for a better model. **Psinest's pipeline migration solved cost while preserving the team-of-roles topology. Hugo's migration solved capability by collapsing the topology entirely** — and the cost win came along for free. Two different platform pivots, same architectural sequence: prove the pattern works first, then re-architect once the binding constraint becomes clear.
**Stack — adopted platforms:** Hermes Agent (Nous Research, MIT) · Codex CLI (gpt-5.4-mini, per-customer ChatGPT Plus subscription) · MCP servers (Python, PEP 723 inline deps + `uv run --script`) · Upload-Post.com (managed-OAuth posting, 10 platforms) · Google AI Pro (Veo 3 + Imagen + Nano Banana, per customer) · Telegram Bot API · Docker · Cloudflare Tunnel + Tailscale · Cloudflare Pages
**Stack — my architecture on top:** single-agent topology with specialists-as-MCP-tools · **Pattern C** canonical skill template (6-section SKILL.md with counterfactual guardrail, forbidden-tokens list, state-readback via `mode: "readback"`) · per-tenant Docker isolation · approval-gated HITL with code-enforced spend caps · weekly cadence orchestration (Monday plan / midweek check / Friday report / monthly review) · per-task model selection · client ad spend stays on client's own Google/Meta accounts (manager OAuth, no float, no AR risk)
**Case study:** [./hugo/](./hugo/) — architecture evolution · Pattern C deep dive · capabilities · stack · design decisions · comparison vs alternatives
**Live site:** [jccosta94.github.io/hugo-website](https://jccosta94.github.io/hugo-website/)

#### 🎬 [Skoda AI Ops](./skoda-ai-ops/) — content operations for [Skoda Clever Kaufen](https://skoda-clever-kaufen.de)
**Live brand traffic.** A content-operations architecture for a German automotive referral business, managed entirely from [Claude Cowork](https://claude.com/cowork) — the operator cockpit, the scheduled drafter (via Cowork Automations), and the approval surface, all in one app — with a deterministic Python publishing library between every LLM draft and every public API call. **Three-generation evolution of a single-operator content engine:**
- **v1 (prompt-gated) — broke in under a week.** Relied on system-prompt instructions for German regulatory compliance ("don't use `Festpreis`", "always hedge prices with `ab`"). ~95% reliable, which is exactly the 5% of regulatory exposure a small business cannot absorb.
- **v2 (library-gated) — compliance fixed.** Collapsed the compliance surface into ~300 lines of Python that runs *before* any publishing API call fires. Checks German-language signals, price hedging (every `€` preceded by `ab`/`bis zu`/`ca.`/`etwa`), regex denylist with word-boundary enforcement (`Festpreis` blocked; `Beispielrechnung` compounds pass), and channel-specific voice rules. **If any check fails the publishing API call does not fire.**
- **v3 (Cowork-cockpit + Automations + dual attribution, current).** Cowork is the dashboard, editor, and orchestrator — every MCP server (Firebase, Gmail, Airtable, Gemini, Veo, Excalidraw, Upload-Post) lives in one app. Cowork Automations handles the weekly cadence (Tue blog · Wed newsletter · Thu/Fri/Sat social previews) — same Claude, same skills, same MCP fleet, same publishing library. **No separate scheduling runtime.** Telegram preview-approval gate (minutes, not hours). Single fan-out via [Upload-Post](https://upload-post.com) (1 call → 4 channels). Dual-tracked attribution: GA4 → Google Ads for bid optimisation, Firestore `quote_events` as source-of-truth for commission accounting.

**The architectural insight:** **putting determinism where the LLM is unreliable.** Brand voice can live in a prompt. Compliance cannot. The library is not smart; it is right. That swap — soft prompt rules to hard library checks — is the single most important decision in the whole system, and it generalises to any commercial AI system operating in a regulated domain.
**Stack:** Claude Cowork (operator cockpit + Automations) · Python compliance library · Firebase/Firestore · Upload-Post API · Telegram approval webhook · Anthropic Claude (text) · Gemini Nano Banana (image) · Veo 3.1 (video) · Google Ads + GA4 Consent Mode v2
**Case study:** [./skoda-ai-ops/](./skoda-ai-ops/) — v1→v2→v3 evolution · the Python compliance library · Cowork-as-cockpit architecture · alternative deployment (headless agent on VPS)

### Founder product + methodology case study

#### 🧠 [Psinest](./psinest/) — healthcare CRM with AI augmentation roadmap
Live at [psinest.duckdns.org](https://psinest.duckdns.org). Portuguese-language platform for psychologists, clinics, and patients. **Architected** the full system: role-scoped access via `RoleScope.cs` (ClinicOwner / Psychologist / Patient), document storage with consent-aware permissions, multi-tenant data isolation, observability + backup topology. Current architecture work: **AI-augmentation roadmap** mapping 8 capability domains (transcription, clinical insights, treatment planning, patient companion, smart booking, document AI, ops intelligence, knowledge search) onto a compliance-aware deployment plan (RGPD / ERS gates, PII redaction proxy, AI Policy Manager centerpiece).
**Stack:** React 19 · TypeScript · Vite 8 · Tailwind 4 · ASP.NET Core 8 · EF Core 8 · PostgreSQL 16 · Firebase Auth · Backblaze B2 · Sentry · Hostinger VPS · pgvector (future)
**Case study:** business problem · architecture decisions · AI stack roadmap · lessons learned

#### ⚙️ [OpenClaw-Hermes Evolution](./openclaw-hermes-evolution/) — multi-agent build system
**Built to solve a velocity problem. Evolved to solve the economics problem velocity created.** A 4-person team couldn't ship Psinest at the cadence the pilot required, and hiring wasn't viable — so I architected an AI dev team on top of two off-the-shelf agentic platforms, in sequence:
- **v1 (OpenClaw) — solved velocity.** Configured [OpenClaw](https://openclaw.ai/) (by [@steipete](https://x.com/steipete)) as a **hierarchical 7-agent team** on a VPS: Telegram → CEO → PM → Dev + Bugfixing Dev → QA + QA2. All on **Anthropic API metered pricing** — **5 worker agents on Opus 4.6**, CEO on Sonnet 4.6, Haiku-based router. **Three human-in-the-loop gates** every cycle: CEO proposes priorities from the kanban board (I approve), I review/merge every PR (no agent self-merge for product code), I trigger every deploy via manual `workflow_dispatch`. The monthly Anthropic bill then became its own problem.
- **v2 (Hermes) — solved economics.** Migrated to [Hermes Agent](https://hermes-agent.nousresearch.com/) (open source, by [Nous Research](https://nousresearch.com)). **Single autonomous dispatcher** in Docker, orchestrated by Codex (gpt-5.4-mini, OpenAI Plus/Pro subscription) calling Claude Code CLI under Claude Max subscription. **The same three human gates carry over from v1** — "autonomous dispatcher" describes how Hermes handles execution *between gates*, not what it decides to ship or deploy. Flat monthly cost regardless of throughput, **order-of-magnitude cheaper** than v1. Lost the team-of-roles concurrency in exchange for a predictable bill.

The headline lesson: **velocity first, economics second.** Prove the pattern works before optimizing what it costs. Documented: the hierarchical team design, per-agent model routing, the three human-in-the-loop gates, throughput vs cost analysis, the v1→v2 migration playbook, when to run both in parallel.
**Stack — adopted platforms:** OpenClaw (v1) · Hermes Agent (v2) · Claude Code CLI · Anthropic API (v1 metered) · OpenAI + Claude Max subscriptions (v2 flat) · Docker · Telegram Bot API · GitHub Actions · systemd
**Stack — my architecture on top:** hierarchical agent role definitions (CEO/PM/Dev/Bugfixing Dev/QA/QA2) · dispatch-allowlist authorization · **three human-in-the-loop gates** (priority approval, merge review, deploy trigger) · cron-based watchdog auto-recovery · GitHub-issue-driven workflow · kanban-board-as-source-of-truth · dev-on-same-VPS-as-prod cost optimization

### AI workflow tools & vertical workflows

#### 💼 [Salesforce Delivery Copilot](./salesforce-delivery-copilot/)
Architected an AI copilot for Salesforce delivery teams. Takes meeting transcripts → produces structured notes → outputs process flows and user stories → keeps the project plan, phases, deliverables, and context up-to-date as the project evolves. **Single input becomes a coherent multi-artifact output**: notes + process diagrams + user stories + a continuously-updated project plan, cross-referenced. Production-deployed for Salesforce delivery work.

#### 🤝 [Claude Cowork Vertical Workflows](./claude-cowork/)
Architected Cowork-mode use cases tailored to specific verticals — healthcare clinics, law firms, home services businesses. MCP workflows for vertical-specific tools, customer-journey designs, deployment models per industry. Vertical-templated Cowork setups for non-developer professionals.

---

## Patterns and writing

#### 📐 [Shared Architecture Patterns](./shared-architecture-patterns/)
Cross-project patterns I keep coming back to: AI routing patterns, multi-agent design, orchestration philosophy, cost-vs-quality tradeoffs, enterprise AI governance, evaluation and observability.

#### 🗺️ [Diagrams Library](./diagrams/)
Reusable architecture diagrams: enterprise RAG patterns, agentic systems, deployment topologies, orchestration patterns. All inline text-style ASCII inside fenced code blocks — readable on GitHub, version-controlled, diff-able.

#### ✍️ [Writing](./writing/)
Essays on agentic systems and the economics of AI-augmented delivery: *Lessons from building agent companies*, *AI workflow architecture*, *Model routing economics*, *Enterprise AI deployment lessons*, *Orchestration bottlenecks*.

---

## Quick facts

- 🇨🇭 Based in **Basel, Switzerland** · open to remote · open to relocate within Switzerland or anywhere in Europe
- 🎓 PSPO-certified Product Owner · transitioned from Salesforce / enterprise delivery consulting into AI-native systems architecture in September 2025
- 🤖 Production AI systems operating today: **Hugo** (productized AI marketing employee, live), **Psinest + Hermes Agent build pipeline** (healthcare CRM), **Skoda content-ops pipeline** (live brand traffic), plus internal copilots (**Salesforce delivery**, **Cowork verticals**)
- 🛠️ Architecting with: Claude Code · Anthropic API · OpenAI API · Codex · Gemini · Ollama · OpenClaw · Hermes Agent · Veo · MCP servers · GitHub Actions · pgvector · Docker · Azure AI Foundry
- 💬 Languages: Portuguese (native pt-PT), English (fluent)

---

## Contact

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
🌐 [psinest.duckdns.org](https://psinest.duckdns.org) — live product
🌐 [jccosta94.github.io/hugo-website](https://jccosta94.github.io/hugo-website) — Hugo, the AI marketing employee
🐙 [github.com/jccosta94](https://github.com/jccosta94) — this profile

---

*Every project linked above includes architecture diagrams, a written case study, and documented lessons. The repos are living artifacts — they get updated as the systems evolve. If you're a recruiter or hiring manager wondering whether the documentation matches the working systems, start with [Psinest](./psinest/) and click through to [psinest.duckdns.org](https://psinest.duckdns.org) — the live product is the proof.*
