# João Costa

**Enterprise AI Systems & Workflow Architect**

Enterprise transformation consultant turned AI-native systems builder, specializing in multi-agent orchestration, operational AI workflows, autonomous business systems, and enterprise AI deployment architectures.

Since September, I rapidly transitioned from traditional Salesforce and enterprise delivery consulting into designing and operating AI-first systems across healthcare, CRM workflows, marketing operations, and autonomous development environments. My work focuses on bridging business operations with modern AI orchestration patterns — combining LLMs, agents, retrieval systems, tool-calling, MCP integrations, and workflow automation into production-oriented operational systems.

In a short timeframe, I have built and deployed:

- AI-native CRM systems for healthcare operations
- Multi-agent autonomous development environments
- Enterprise AI copilots for Salesforce delivery workflows
- Autonomous AI marketing operations systems
- AI automation infrastructures for SMBs across DACH and Iberian markets
- Local/private AI experimentation environments using Ollama and Apple Silicon
- Operational orchestration systems integrating Claude, Gemini, MCP tooling, and AI-generated media pipelines

### My architectural interests

- Agentic systems orchestration
- Enterprise AI deployment
- Operational AI workflows
- AI-native business systems
- Retrieval and knowledge architectures
- Human-in-the-loop automation
- Cost-aware model routing
- AI governance and operational reliability
- AI transformation strategy

### Current focus areas

- Azure AI Foundry & enterprise AI architecture
- Agentic enterprise workflows
- Retrieval-augmented operational systems
- AI observability & evaluation
- Multi-model orchestration strategies
- Enterprise AI transformation and deployment

This GitHub serves as a portfolio of AI systems, architecture experiments, operational workflows, and enterprise-oriented AI implementations exploring the intersection of autonomous systems, enterprise operations, and AI-native software architecture.

---

## What's here

### Productized AI services

#### 🤖 [Hugo](./hugo/) — AI marketing employee for SMBs
**Live commercial product. A case study in multi-agent system design under cost-tier model capability — two sequential architectures, same customer surface.**
- **v1 — hierarchical 7-agent team** (Director + Performance Marketer + Web Developer + Brand Designer + Community Manager + Outbound Specialist + Analytics) on [Hermes Agent](https://hermes-agent.nousresearch.com/) (briefly OpenClaw). Each tenant runs on the customer's own ChatGPT Plus subscription — gating Hugo to `gpt-5.4-mini`. **Three independent multi-agent delegation failures broke this design before V1 shipped.** Root cause: when SKILL.md tells `gpt-5.4-mini` to "first call X, then synthesize Y," the model treats the procedural contract as a suggestion and fabricates plausible content from soft preconditions.
- **v2 — single Hugo agent + specialists-as-MCP-tools** ("Pattern C"). Specialist capabilities collapsed from separate agents into area-scoped MCP tool servers (`marketing__*`, `creative__*`, `web__*`, `inbox__*`, `outbound__*`, `analytics__*`, `audit__*`, `content_plan__*`). Agent's only job: invoke one tool per turn, summarize the tool's return value. **Pattern C removes the failure surface entirely** — no soft preconditions for the model to fail on.

**The architectural insight:** model capability is a topology constraint. When the model is locked in by the per-customer pricing tier, you can't upgrade your way out of a bad topology — you have to change the topology. Pattern C is the response: collapse the procedural contract into a single tool call so there's nothing for the failure mode to attach to.
**Stack:** Hermes Agent · gpt-5.4-mini (per-tenant ChatGPT Plus) · Python MCP servers · Docker per-tenant isolation · Telegram (default) + Slack/WhatsApp/Signal/Email customer channels
**Case study:** [./hugo/](./hugo/) — v1/v2 architecture · Pattern C SKILL template · 6 marketing specialties mapped to 8 MCP namespaces · per-customer subscription economics
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
Reusable architecture diagrams: enterprise RAG patterns, agentic systems, deployment topologies, orchestration patterns. Excalidraw source + PNG/SVG exports.

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
