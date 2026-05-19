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
**Live commercial product · first customer onboarding.** Productized AI marketing employee that talks to owners via Telegram / Slack / WhatsApp / email and runs their ads, website, creative, GBP, reviews, outreach, and weekly reports — approval-gated. Built on [Hermes Agent](https://hermes-agent.nousresearch.com/). v1 was a 7-agent team that broke under cost-tier model capability; v2 collapsed to a single agent + specialists-as-MCP-tools (**Pattern C**). **Headline lesson: model capability is a topology constraint** — when the cheap model can't bind multi-step delegation, change the topology.
📖 [Case study](./hugo/) · 🌐 [hugo-website](https://jccosta94.github.io/hugo-website/)

#### 🎬 [Skoda AI Ops](./skoda-ai-ops/) — content operations for [Skoda Clever Kaufen](https://skoda-clever-kaufen.de)
**Live brand traffic.** Content-operations architecture for a German automotive referral business, run entirely from [Claude Cowork](https://claude.com/cowork) with a deterministic ~300-line Python compliance library between every LLM draft and every publish call. Three generations: v1 prompt-gated (broke in <1 week), v2 library-gated (compliance fixed), v3 Cowork-cockpit + Automations + dual GA4 / Firestore attribution. **Headline lesson: put determinism where the LLM is unreliable** — brand voice can live in a prompt; compliance cannot.
📖 [Case study](./skoda-ai-ops/)

### Founder product + methodology case study

#### 🧠 [Psinest](./psinest/) — healthcare CRM with AI augmentation roadmap
**Live pilot** — 10 psychologists + 1 clinic on [psinest.duckdns.org](https://psinest.duckdns.org). Architected the full RGPD + ERS-aware platform: `RoleScope.cs` centralised JWT-derived tenancy at the EF Core query layer, consent-aware document permissions, multi-tenant isolation, observability + backup topology. The **AI augmentation roadmap** layers 8 capability domains (transcription, clinical insights, treatment planning, etc.) behind an **AI Policy Manager** that operates one tier higher than `RoleScope`. **Headline lesson: compliance is the load-bearing concern** — centralise the access gate, treat AI capability access as a parallel tier.
📖 [Case study](./psinest/) · 🌐 [psinest.duckdns.org](https://psinest.duckdns.org)

#### ⚙️ [OpenClaw-Hermes Evolution](./openclaw-hermes-evolution/) — multi-agent build system shipping Psinest
**Built for velocity, re-architected for economics.** A 4-person team couldn't ship Psinest at pilot cadence, so I built an AI dev team on top of two off-the-shelf platforms in sequence. v1 [OpenClaw](https://openclaw.ai/) — hierarchical 7-agent team on Anthropic API metered pricing, solved velocity. v2 [Hermes Agent](https://hermes-agent.nousresearch.com/) — single autonomous dispatcher under flat subscription, order-of-magnitude cheaper, solved economics. **Same three human-in-the-loop gates across both: priority approval, merge review, deploy trigger.** **Headline lesson: velocity first, economics second** — prove the pattern works before optimising what it costs.
📖 [Case study](./openclaw-hermes-evolution/)

### AI workflow tools & vertical workflows

#### 💼 [Salesforce Delivery Copilot](./salesforce-delivery-copilot/)
**Production-deployed.** AI copilot for Salesforce delivery teams. Takes meeting transcripts → produces structured notes → outputs process flows and user stories → keeps the project plan, phases, and deliverables continuously up to date. Single input becomes a coherent multi-artifact output.
📖 [Case study](./salesforce-delivery-copilot/) *(in progress)*

#### 🤝 [Claude Cowork Vertical Workflows](./claude-cowork/)
**Live consulting practice.** Three vertical templates on top of [Claude Cowork](https://claude.com/cowork) — same desktop install, same Claude, three different operator cockpits — varying only the MCP fleet, skill library, and HITL gate model. **Practice Command** (Healthcare, LIVE in PT) · **Legal Efficiency Suite** (Law Firms, LIVE × 2) · **Field Service Command** (Home Services, packaged) · **Command Center** (horizontal fallback). **Headline lesson: configure a reliable platform, don't build agents from scratch** — same operating model that worked for years of Salesforce CRM delivery, applied to AI cockpit delivery.
📖 [Case study](./claude-cowork/)

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
