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
**Live commercial product.** Architected a single-agent AI marketing employee that talks to small-business owners via Telegram / Slack / WhatsApp / email and runs their ads, website, creative, Google Business profile, reviews, outreach, and weekly reports. **Approval-gated workflow:** Hugo prepares work, owner approves via chat reply, Hugo launches.
**Architecture decisions:** single-agent persona with persistent business context · approval-routing across messaging channels · per-task model selection (ads copy vs creative gen vs report writing) · budget-limit enforcement gates before any action · weekly cadence orchestration (Monday plan / midweek check / Friday report / monthly review) · client ad spend stays on client's own Google/Meta accounts (Hugo operates them, doesn't hold money) · built on the same [OpenClaw](https://openclaw.ai/) gateway used for [the Psinest build pipeline](./openclaw-hermes-evolution/) — configured as a single persona instead of a 7-agent team.
**Case study:** [./hugo/](./hugo/) — architecture · capabilities · stack · design decisions · comparison vs alternatives
**Live site:** [jccosta94.github.io/hugo-website](https://jccosta94.github.io/hugo-website/)

#### 🎬 [Skoda AI Ops](./skoda-ai-ops/) — content automation for an automotive brand
Architected the content automation pipeline for a referral automotive brand — social media production, blog generation, Veo video workflows, publishing automation across channels. Production traffic to a real audience.
**Architecture decisions:** multi-channel content fan-out from single brief · Veo video generation orchestration · brand-voice consistency across formats · time-zone-aware scheduled publishing · approval routing before public release.

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
