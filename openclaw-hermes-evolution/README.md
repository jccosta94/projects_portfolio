# OpenClaw-Hermes Evolution

**A multi-agent build-system architecture, in two generations. Designed for velocity, re-architected for economics.**

This is a case study in **multi-agent system design** — specifically: how to architect a production AI dev team for a small organization, when to re-architect it under cost pressure, and what survives the migration.

The system ships code to [Psinest](https://github.com/jccosta94/psinest), a production healthcare CRM. The constraint that triggered the architecture work: a 4-person team (1 senior FT engineer + 2 part-time engineers + me as PM) couldn't sustain production cadence by hand, and hiring wasn't a viable option. My job was to design an AI system that closed the gap — not to be that AI system myself.

### What I architected

**v1 — a hierarchical 7-agent team** on top of [OpenClaw](https://openclaw.ai/) (by [@steipete](https://x.com/steipete)): a Telegram-routed org chart with **CEO → PM → Dev + Bugfix → QA + QA2** (plus a Haiku-based router fielding incoming messages). I designed the dispatch-authorization graph (`subagents.allowAgents` whitelists), the per-agent model routing (Sonnet 4.6 for the CEO, Opus 4.6 for all five workers, Haiku 4.5 for routing), the system prompts for each role, and the `dev-watchdog.sh` auto-recovery cron that catches stalled Dev sessions and re-dispatches with context. Runs as a single systemd service on a Hostinger VPS that *also* hosts the production Psinest stack — a deliberate co-location for cost reasons. **This is the architecture that solved velocity.**

**v2 — a single autonomous dispatcher** on top of [Hermes Agent](https://hermes-agent.nousresearch.com/) (open source, by [Nous Research](https://nousresearch.com)): when 5 workers on Opus 4.6 at production cadence stopped being financially defensible, I re-architected the system around a Docker-isolated Hermes instance orchestrated by Codex (gpt-5.4-mini, OpenAI subscription) that shells out to Claude Code CLI (Claude Max subscription) as its code-writing tool. I wrote the SOUL.md identity prompt, the `runlog/` audit-trail structure, the `hermes-ready` issue-label routing protocol, and the Docker isolation pattern that keeps this instance independent of the native Hermes install. **This is the architecture that made velocity sustainable.**

### The architectural insight

The through-line is **the sequence**: prove the multi-agent pattern produces real velocity *first* (with whatever architecture and billing tier it takes), then re-architect the system once economics demand it. Trying to optimize cost from day one would have constrained the prompt-engineering iteration v1 needed; ignoring cost once velocity worked would have killed the budget. **Two designs, sequential. Same workflow on top. Same Claude doing the actual code-writing in both.**

This repo documents the architecture of each generation: the topology I designed, what each platform provides out of the box vs what I configured on top, the per-agent model routing decisions, the v1→v2 migration playbook, and the cost numbers that justified the re-architecture.

---

## TL;DR

| | **v1 — OpenClaw** | **v2 — Hermes** |
|---|---|---|
| **Solves which problem** | **Velocity** — ship at production cadence | **Economics** — make that cadence sustainable |
| **Platform** | [openclaw.ai](https://openclaw.ai/) by @steipete | [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/) (MIT, Nous Research) |
| **Architecture** | **Team of 7 personas in 1 gateway**: hierarchical org chart (CEO → PM → Dev + Bugfix → QA + QA2 + Haiku router) | **Single autonomous dispatcher** in Docker, calls Claude Code CLI as a tool |
| **Models** | Anthropic API: **5 agents on Opus 4.6**, CEO on Sonnet 4.6, router on Haiku 4.5 | Codex orchestrator (gpt-5.4-mini, OpenAI subscription) → Claude Code CLI (Claude Max subscription) |
| **Billing** | Anthropic API · metered per token | OpenAI Plus/Pro + Claude Max · flat monthly |
| **Cost** | Material monthly line item — metered, scales with cadence | Flat subscription — bounded regardless of throughput |
| **Throughput** | High — 5 worker agents in parallel, no rate-limit ceiling | Meaningfully slower — single executor under subscription rate-limits |
| **Quality** | High production-pass rate | Same quality — no measurable drop from v1 |
| **I/O surface** | Telegram (`@Openclaw_Psinest_bot`) → CEO → cascade | Telegram (`@Psinest-Hermes_bot`) → Hermes → Claude Code |
| **Audit trail** | Per-agent sessions in `/root/.openclaw/agents/<role>/sessions/` | `runlog/` directory: status.md, log.md, prompts/<issue>.md, claude-output/<issue>.txt |
| **Auto-recovery** | `dev-watchdog.sh` cron: every 10 min, detects stale Dev sessions, commits WIP, re-dispatches | Built-in: SOUL.md instructs Hermes to check PR state before re-dispatching |
| **Best phase** | Build & validate the pipeline, dial in the team roles | Run the validated pipeline at sustainable burn |

The headline insight: **same Claude models, two different billing tiers, two different architectural shapes.** v1 ran the full Anthropic API at premium rates with parallel agents. v2 routes the same Claude through Claude Max subscription and serializes work through a single dispatcher. The team-of-roles concurrency went away; the issue → PR → review → merge → deploy workflow stayed.

---

## Why this matters

Most teams shipping AI-augmented systems in 2026 hit two walls in sequence: **velocity first, economics second.**

**Wall 1 — Velocity.** The team is too small to ship at the cadence the product needs. Hiring is slow (months) and expensive (€50-100k+ all-in per FTE). AI-augmented workflows are the obvious answer, but a single Claude session pasted-and-edited doesn't scale either — what you need is a *team of specialized agents with proper authorization boundaries* (who can dispatch whom, who reviews what, who can merge). **This is what OpenClaw v1 solved, configured as a 7-persona team with strict dispatch rules.**

**Wall 2 — Economics.** Once the multi-agent team is producing real PRs at production cadence, the monthly Anthropic bill arrives. Five worker agents running on Opus 4.6 metered API at production throughput is real money. There's no flat-rate Anthropic tier until enterprise pricing, which a small team can't sensibly commit to. **This is what Hermes v2 solved, by re-platforming onto subscription-tier orchestration without changing the Psinest workflow itself.**

The lesson is **the sequence.** Prove velocity works with whatever it takes (premium models, paid-per-token, fine — get the team operational first). *Then* optimize economics once the pattern is stable. Trying to optimize cost from day one would have killed velocity (subscription rate limits constrain prompt-engineering iteration); ignoring cost once velocity worked would have killed the budget. Doing them in sequence preserved both.

The lesson is also **what stayed constant.** Both generations route through Telegram for I/O. Both work on the same Psinest repos (`/root/psinest-{app,api}/` on the VPS). Both push to GitHub. Both rely on Joao for merge approval and deploy triggers. The agent architecture changed; the developer workflow didn't.

---

## v1 — OpenClaw: the team-of-7

OpenClaw is an off-the-shelf "personal AI assistant" platform that supports configuring multiple agent personas inside a single gateway process. I configured it as a **hierarchical software team**, modelled after how a real engineering org operates.

### The team

```
                      Telegram (@Openclaw_Psinest_bot)
                              │
                              ▼
                       ┌──────────────┐
                       │ main (Haiku) │   ← default router
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │   CEO 👔      │   ← Sonnet 4.6
                       │  (strategy)  │
                       └──────┬───────┘
                              │ can dispatch to anyone
                              ▼
                       ┌──────────────┐
                       │   PM 📋      │   ← Opus 4.6
                       │ (planning)   │
                       └──────┬───────┘
                              │ can dispatch to workers
                              ▼
              ┌───────────────┴───────────────┐
              ▼                               ▼
       ┌────────────┐                 ┌────────────┐
       │  Dev 💻    │                 │ Bugfix 🐛  │   ← both Opus 4.6
       │ (features) │                 │ (bugs)     │
       └─────┬──────┘                 └─────┬──────┘
             │ can dispatch to QAs          │
             ▼                               ▼
       ┌────────────┐                 ┌────────────┐
       │   QA 🔍    │                 │  QA2 🧪    │   ← both Opus 4.6
       │ (testing)  │                 │ (testing)  │
       └────────────┘                 └────────────┘
       terminal: no subagent dispatch
```

### Agent roster (from `/root/.openclaw/openclaw.json`)

| Agent ID | Model | Role | Can dispatch to | Workspace |
|---|---|---|---|---|
| `main` | Haiku 4.5 | Default agent fielding incoming Telegram messages | (channel routing only) | shared default |
| `ceo` 👔 | **Sonnet 4.6** | Top-level strategy, priority calls, scope decisions | `pm, dev, bugfix, qa, qa2` | `/root/.openclaw/agents/ceo/` |
| `pm` 📋 | **Opus 4.6** | Project management, sprint scoping, ticket triage | `dev, bugfix, qa, qa2` | `/root/.openclaw/agents/pm/` |
| `dev` 💻 | **Opus 4.6** | Software engineer (primary feature lane) | `qa, qa2` | `/root/.openclaw/agents/dev/` |
| `bugfix` 🐛 | **Opus 4.6** | Software engineer (bug-fix lane — functionally my second dev) | `qa, qa2` | `/root/.openclaw/agents/bugfix/` |
| `qa` 🔍 | **Opus 4.6** | QA agent #1 | terminal | `/root/.openclaw/agents/qa/` |
| `qa2` 🧪 | **Opus 4.6** | QA agent #2 | terminal | `/root/.openclaw/agents/qa2/` |

Five of the seven agents run on **Opus 4.6** — that's the headline cost driver, documented in detail in [`comparisons/cost-analysis.md`](./comparisons/cost-analysis.md).

### Routing and authorization

Telegram is the only I/O channel. All incoming messages route to the CEO (configured in `bindings`):

```json
"bindings": [{"type": "route", "agentId": "ceo", "match": {"channel": "telegram"}}]
```

From there, dispatch flows down the hierarchy. The `allowAgents` whitelist on each agent enforces who can talk to whom. **QA agents are terminal nodes** — they can run tests and report findings, but they can't recursively dispatch further subagents. This prevents loops and keeps the org chart shallow.

### Deployment

OpenClaw runs as a **single systemd service** on a Hostinger VPS (Ubuntu 24.04, 1 vCPU / 3.8 GB):

```ini
[Unit]
Description=OpenClaw Gateway — Psinest AI Agent Team

[Service]
ExecStart=/usr/local/bin/openclaw-wrapper.sh
Restart=always
```

One Node.js process (~400 MB RAM) on port 18789, hosting all seven personas internally. The same VPS also runs the production Psinest stack (nginx, Kestrel API, PostgreSQL) and holds the live repos at `/root/psinest-app/` and `/root/psinest-api/`. **Dev and prod share infrastructure** — a deliberate cost optimization for a pre-revenue product.

### Auto-recovery: the dev-watchdog

A cron job (`*/10 * * * * /root/dev-watchdog.sh`) runs every 10 minutes to detect stalled Dev sessions:

1. Check for open GitHub issues labelled `agent:dev`
2. Find the latest Dev session file in `/root/.openclaw/agents/dev/sessions/*.jsonl`
3. If the file hasn't been touched in 10+ minutes, the Dev agent has stalled
4. Auto-commit any uncommitted WIP (`git commit -m "WIP: auto-commit from dev-watchdog"`), push to origin
5. Re-dispatch Dev with full context of remaining open issues
6. Ping Joao on Telegram with a recovery notice

This was essential because Opus 4.6 sessions occasionally time out mid-feature. Without the watchdog, an overnight session timeout would lose context until Joao manually noticed.

### Skills library

OpenClaw ships with — and lets you author — markdown-based **skills** that each agent can invoke. The Psinest install has 23 skills under `/root/.openclaw/workspace/skills/`, including:

- `dispatching-parallel-agents` — the parallel-fan-out playbook
- `task-planning` / `writing-plans` / `executing-plans` — multi-step work decomposition
- `verification-before-completion` — final-check protocol before claiming a task done
- `code-review-excellence` / `requesting-code-review` — QA flow
- `architecture-patterns` — system-design reasoning
- `security-best-practices` — generic security audit
- `e2e-testing-patterns` — Playwright test scaffolding
- `database-schema-designer` — schema reasoning for new tables
- `runesleo-systematic-debugging` — structured bug-investigation flow

These weren't mine — they ship with OpenClaw — but selecting which ones to load and how the team uses them is part of the architecture.

---

## v2 — Hermes Agent: the single autonomous dispatcher

[Hermes Agent](https://hermes-agent.nousresearch.com/) is an open-source autonomous agent framework from [Nous Research](https://nousresearch.com) (MIT-licensed). The Psinest setup runs it in a Docker container on my Mac, deliberately separated from the native Hermes Agent install elsewhere on the system.

### Architecture

```
        Telegram (@Psinest-Hermes_bot)
                  │
                  ▼
         ┌─────────────────┐
         │  Hermes Agent   │   ← Docker container, port 8643
         │  (autonomous    │
         │   dispatcher)   │
         └────────┬────────┘
                  │ orchestrator brain
                  ▼
         ┌─────────────────┐
         │     Codex       │   ← OpenAI Plus/Pro subscription
         │ (gpt-5.4-mini)  │     (reasoning, planning, conversation)
         └────────┬────────┘
                  │ shells out to
                  ▼
         ┌─────────────────┐
         │ Claude Code CLI │   ← Claude Max subscription
         │  (`claude -p`)  │     (code-writing executor)
         └─────────────────┘
                  │ writes to / reads from
                  ▼
         /workspace/psinest/   (mounted from host)
```

Per the SOUL.md system prompt that defines Hermes-Psinest's identity:

> "Your one job: pick up tasks from Joao's GitHub board, prompt Claude Code via the `/claude-code` skill to do them, and report results back. **You do not write code, edit files, or run git commands yourself. You are a dispatcher, not a worker.**"

### The audit trail

Every dispatch leaves a paper trail in `runlog/`:

| File | Purpose |
|---|---|
| `runlog/status.md` | One-block snapshot: `status:`, `issue:`, `started:`, `updated:`, `notes:` — overwritten on each state change |
| `runlog/log.md` | Append-only event log: `<ISO> | <event> | <description>` per meaningful event |
| `runlog/prompts/<issue>.md` | Verbatim Claude Code prompt saved BEFORE invocation. If re-dispatched: `-v2`, `-v3`, etc. |
| `runlog/claude-output/<issue>.txt` | Full `claude -p` stdout per dispatch |

This is the **operational journal** that lets Joao audit Hermes without reading internal session logs. Every prompt is reproducible. Every dispatch is traceable.

### What Hermes provides vs what I configured

**Hermes Agent provides** (per its docs):
- Autonomous agent loop with persistent memory
- Auto-generated skills that compound over time
- Multi-platform messaging (Telegram, Discord, Slack, WhatsApp, Signal, Email, CLI)
- Real sandboxing (5 backends: local, Docker, SSH, Singularity, Modal)
- Scheduled automations via natural-language cron
- Isolated subagents with their own conversations and Python RPC

**I configured / wrote**:
- The SOUL.md identity prompt (dispatcher-not-worker contract)
- The runlog/ audit-trail pattern (status / log / prompts / claude-output structure)
- The `hermes-ready` issue label as Hermes's queue
- The `/claude-code` skill integration for dispatching Claude Code as a tool
- The Docker isolation pattern (port 8643 + `~/.hermes-psinest/` data dir, separate from the native Hermes install on port 8642)
- The Telegram bot allowlist (only my Telegram user ID)

---

## Architecture diagrams

| File | What it shows |
|---|---|
| [`architecture/openclaw-v1.excalidraw`](./architecture/openclaw-v1.excalidraw) | v1 topology — 7-agent team in one OpenClaw gateway on the VPS |
| `architecture/hermes-agent-v2.excalidraw` *(TBD)* | v2 topology — Hermes dispatcher in Docker, Codex brain, Claude Code executor |
| `architecture/orchestration-comparison.excalidraw` *(TBD)* | Side-by-side v1 vs v2 with model + cost annotations |
| `architecture/deployment-flow.excalidraw` *(TBD)* | How both generations deploy code to the same Hostinger VPS |

PNG exports of each live alongside the source files. The diagrams open in [excalidraw.com](https://excalidraw.com) (drag-and-drop) or any compatible editor.

---

## Deeper reading

### Comparisons (the case for v2 over v1, with numbers)
- [`comparisons/cost-analysis.md`](./comparisons/cost-analysis.md) — **start here.** Per-agent monthly burn, v1 vs v2, with the math. 5 agents on Opus 4.6 is the cost driver.
- [`comparisons/throughput-analysis.md`](./comparisons/throughput-analysis.md) *(TBD)* — what we gave up moving to v2, measured in PRs/week and time-to-ship.
- [`comparisons/model-routing.md`](./comparisons/model-routing.md) *(TBD)* — which model for which agent role, and why the v2 single-executor pattern works.
- [`comparisons/architectural-tradeoffs.md`](./comparisons/architectural-tradeoffs.md) *(TBD)* — team-of-7 hierarchy vs single dispatcher; when each makes sense.

### Workflows (how an issue moves through the pipeline)
- [`workflows/agent-lifecycle.md`](./workflows/agent-lifecycle.md) *(TBD)* — v1: CEO → PM → Dev → QA hand-off. v2: Hermes → Claude Code dispatch.
- [`workflows/task-routing.md`](./workflows/task-routing.md) *(TBD)* — how the `agent:dev` / `hermes-ready` labels route issues.
- [`workflows/code-generation-flow.md`](./workflows/code-generation-flow.md) *(TBD)* — full issue → PR → merge → deploy chain.

### Case study (the v1 → v2 story)
- [`case-study/v1-lessons.md`](./case-study/v1-lessons.md) *(TBD)* — what we learned running the OpenClaw 7-agent team in production.
- [`case-study/v2-lessons.md`](./case-study/v2-lessons.md) *(TBD)* — what we learned after the cutover to Hermes.
- [`case-study/operational-insights.md`](./case-study/operational-insights.md) *(TBD)* — patterns that worked across both platforms.

---

## How to sequence the generations

This is not a "pick one" decision — it's a sequence. Learned by going through it.

### Phase 1: build v1 first, regardless of cost concerns

If you don't have a working multi-agent pipeline yet, **start with OpenClaw on Anthropic API.** Pay the metered price. Don't worry about cost — cost worry is premature. What you're solving for in phase 1 is *whether the multi-agent team produces production-grade output for your codebase*. That's not a cheap experiment; it requires the best models you can throw at it, with full parallelism, no rate-limit constraints, and the ability to iterate quickly on agent prompts.

Premium models also forgive prompt-engineering mistakes that subscription tiers won't. If your Dev agent prompt is 80% right, Opus 4.6 will probably ship a good PR anyway; the same Claude under a rate-limited subscription might struggle. **Get the prompts dialled in on v1 first.**

**You're done with phase 1 when:** the team is shipping real PRs at production cadence, production-pass rate is acceptable, and your agent role definitions have stabilized (you're not editing the CEO/PM/Dev system prompts every week).

### Phase 2: migrate to v2 once v1 is stable

If v1 is working but your monthly bill is uncomfortable, **migrate to Hermes Agent.** The team architecture *doesn't* transfer directly — Hermes is a single-dispatcher model, not a team-of-roles. What transfers is the *workflow*: GitHub-issue-driven, Telegram-controlled, audit-logged. The 7-agent hierarchy collapses into "Hermes decides what to dispatch, Claude Code executes."

Don't migrate everything at once. Start with a maintenance lane — issues labelled `hermes-ready` route to v2 while feature work stays on v1. Run that hybrid for 2-4 weeks, measure quality and latency, then progressively shift more work onto v2 as you trust it.

**You're done with phase 2 when:** the maintenance lane is reliable on v2, your monthly bill has dropped meaningfully, and you've codified which kinds of work belong in which lane.

### Phase 3: keep v1 ready, run v2 as default

For Psinest's current state: **v2 is the default; v1 is dormant but recoverable.** v2 handles all routine work. v1 stays in the systemd unit definition (currently disabled), ready to be re-enabled for launch crunches when you need the team-of-roles concurrency back. The crontab backup at `/root/crontab.backup-<timestamp>` includes the dev-watchdog reactivation line. Restoring v1 is a `systemctl enable && systemctl start && crontab <backup>` away.

**Don't kill v1 when you migrate to v2.** The optionality is worth more than the slight maintenance overhead.

---

## What I designed vs what the platforms provide

For clear credit attribution:

| Concern | Platform provides | My architectural contribution |
|---|---|---|
| **Chat orchestration** | OpenClaw: Telegram/Discord/WhatsApp channels, agent gateway, persona system; Hermes: autonomous loop, multi-platform messaging | Configured the channels (Telegram-only allowlist), wrote the bot allowlist policy |
| **Multi-agent topology** | OpenClaw: agent personas, `subagents.allowAgents` dispatch authorization, skills system; Hermes: subagent delegation | Designed the 7-agent **org-chart hierarchy** (CEO → PM → Dev+Bugfix → QA+QA2), wrote the dispatch allowlists, picked who's terminal vs who can recurse |
| **Per-agent system prompts** | OpenClaw / Hermes: load arbitrary markdown / SOUL.md as system prompt | Wrote each agent's identity, scope, branch-guard rules, contract-verification protocol |
| **Model routing** | OpenClaw / Hermes: per-agent `model` field in config | Decided which agent gets Opus 4.6 vs Sonnet 4.6 vs Haiku 4.5 — and *why* |
| **Audit trail** | Hermes: optional logging; OpenClaw: per-agent SQLite memory | Designed the `runlog/{status,log,prompts/,claude-output/}` structure; ensured every Claude prompt is reproducible |
| **Auto-recovery** | Neither platform ships this | Wrote `dev-watchdog.sh` — detects stale Dev sessions, auto-commits WIP, re-dispatches with context, pings Joao |
| **Deploy infrastructure** | Neither platform manages this | Co-located dev and prod on one VPS, configured GitHub Actions to ssh-deploy to the same box, kept agent work in repo dirs at `/root/psinest-{app,api}/` |
| **Migration playbook** | N/A — these are different products | Designed the `hermes-ready` label-based gradual rollout, the both-mode operational period, the v1-as-dormant-fallback pattern |

The platforms are the heavy lifting; the team design and operational architecture are mine.

---

## Stack

| Layer | v1 (OpenClaw) | v2 (Hermes) |
|---|---|---|
| Agent platform | [OpenClaw](https://openclaw.ai/) (Node.js, systemd) | [Hermes Agent](https://hermes-agent.nousresearch.com/) (Docker, MIT) |
| Orchestrator model | CEO on Sonnet 4.6 + Opus 4.6 workers (per agent) | Codex / gpt-5.4-mini (single orchestrator) |
| Code executor | Each agent is itself a Claude session | Claude Code CLI (`claude -p ...`) called as a tool |
| Model billing | Anthropic API · metered per token | OpenAI Plus/Pro + Claude Max · flat monthly |
| I/O surface | Telegram (`@Openclaw_Psinest_bot`) | Telegram (`@Psinest-Hermes_bot`) |
| Process isolation | None — single Node process, agents share gateway | Docker container with mounted `/workspace/psinest/` |
| Source of truth | GitHub (`jccosta94/psinest-app` + `jccosta94/psinest-api`) | Same |
| Deploy target | Hostinger VPS via GitHub Actions `workflow_dispatch` | Same — neither generation deploys; Joao still triggers manually |
| Auto-recovery | `/root/dev-watchdog.sh` via cron every 10 min | SOUL.md instructs PR-state check before re-dispatch |
| Audit trail | Per-agent SQLite memory + session JSONL files | `runlog/` directory with status, log, prompts, output |

---

## Status

- 🛑 **v1 (OpenClaw)** — currently **off**. Service `openclaw.service` stopped and disabled; `dev-watchdog.sh` cron disabled (line commented out in crontab, reversible). Crontab backup at `/root/crontab.backup-<timestamp>`. To revive: `systemctl enable openclaw.service && systemctl start openclaw.service`, then restore the crontab.
- ✅ **v2 (Hermes-Psinest)** — operational. Docker container running on Joao's Mac, handles all current Psinest maintenance via `hermes-ready` issue label.
- 🚧 **Diagrams** — v1 diagram complete; v2 + comparison + deployment diagrams TBD.
- 🚧 **Case studies** — outlined; full writeups TBD.
- 🚧 **Code examples** — agent config snippets, Codex orchestrator system prompt, Dockerfile, dev-watchdog script all pending sanitization for public release.
- 🚧 **Secrets rotation** — original v1 deployment hardcoded OAuth tokens in `/usr/local/bin/openclaw-wrapper.sh`. Migration to env-file pattern (`/etc/openclaw/env` with `chmod 600`) is recommended for any re-deployment.

---

## Acknowledgements

The two platforms this case study is built on:

- **[OpenClaw](https://openclaw.ai/)** by [@steipete](https://x.com/steipete) — the chat-based agent gateway that made the 7-agent team possible.
- **[Hermes Agent](https://hermes-agent.nousresearch.com/)** by [Nous Research](https://nousresearch.com) — the open-source autonomous agent framework (MIT) that v2 runs on.

Plus [Claude Code CLI](https://docs.claude.com/claude-code) (the code-writing executor in both generations) and [Codex](https://platform.openai.com/) (gpt-5.4-mini, the v2 orchestrator brain).

The team-design patterns — hierarchical role split, dispatch authorization, per-role system prompts, branch-guard discipline, contract verification, layered testing — are borrowed from how real human engineering teams work. **Multi-agent systems work best when they mirror the practices that already work for humans.**

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com) · 🌐 [psinest.duckdns.org](https://psinest.duckdns.org)
