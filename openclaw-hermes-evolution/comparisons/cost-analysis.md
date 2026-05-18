# Cost Analysis — OpenClaw v1 vs Hermes v2

This document exists because of a sequence: I configured a 7-agent team on [OpenClaw](https://openclaw.ai/) to solve a **velocity problem** for Psinest, it worked, then the running cost of that team became a *new* problem that v2 (a single dispatcher on [Hermes Agent](https://hermes-agent.nousresearch.com/)) was built to solve. The analysis here justifies the migration — when it was time to move, what it saved, and what it cost in throughput.

> **Important framing:** this is not an argument that v2 is "better." v2 didn't exist as an option when we needed v1 — we couldn't have built Psinest's pipeline starting from v2 because subscription-tier rate limits would have throttled prompt-engineering iteration. **v1 was the right answer in phase 1, v2 became the right answer in phase 2.** The analysis below explains how I knew it was time to move.

---

## TL;DR

| Metric | v1 — OpenClaw (7 personas in 1 gateway) | v2 — Hermes (1 dispatcher) | Δ |
|---|---|---|---|
| **Solves which problem** | Velocity (ship at cadence) | Economics (sustain that cadence) | — |
| **Phase** | Build & validate the pipeline | Run the validated pipeline | Sequential |
| **Architecture** | Hierarchical team: CEO → PM → Dev + Bugfix → QA + QA2 | Single dispatcher → Claude Code CLI | Different |
| **Cost model** | Metered per token (Anthropic API) | Flat subscription (OpenAI Plus/Pro + Claude Max) | Predictability ↑↑ |
| **Monthly cost** | Material line item — scales with throughput | Flat regardless of throughput | **Order-of-magnitude reduction** |
| **Throughput** | High — 5 worker agents in parallel, no rate-limit ceiling | Meaningfully slower — single executor under subscription rate-limits | Material slowdown |
| **Latency per PR** | Faster (parallel Dev + QA execution) | Slower (serialized through one dispatcher) | Trade-off |
| **Quality** | High production-pass rate | Same quality — no measurable drop | No regression |
| **Headline cost driver** | **5 agents on Opus 4.6** (PM, Dev, Bugfix, QA, QA2) | Flat Claude Max subscription | — |

**Headline finding:** moving from a 7-agent team on Anthropic API metered pricing (v1) to a single-dispatcher pattern on subscription-tier orchestration (v2) **cut monthly burn by an order of magnitude** while reducing throughput **meaningfully but not catastrophically**. **Quality did not measurably degrade** — the underlying code executor is still Claude (now invoked under subscription instead of API) — and the developer workflow (GitHub-issue-driven, Telegram-controlled, human-approves-merges) stayed identical.

**The migration trigger:** I moved when the monthly Anthropic bill became a non-trivial line item in Psinest's operating costs *and* the agent role definitions had stabilized (no system-prompt edits in ~4 weeks). Both conditions had to hold; either alone wouldn't have justified the migration.

---

## Methodology

**Cost measurement:**
- v1: aggregated from the Anthropic API console. Each OpenClaw agent persona's calls show up under the same API key (the OAuth token in the wrapper script), so attribution per agent requires correlating with `/root/.openclaw/agents/<role>/sessions/` activity logs.
- v2: OpenAI subscription tier (Plus or Pro) + Claude Max subscription + small infra (Hostinger VPS not counted since same in both generations).
- Excluded from both: human labour cost (PM time, senior engineer review time) — these are constant across both generations.

**Throughput measurement:**
- PR cadence pulled from `gh pr list --repo jccosta94/psinest-app --repo jccosta94/psinest-api --state merged --json mergedAt`.
- A "PR" counts only if it shipped to production via `workflow_dispatch`. WIP PRs, draft PRs, and self-merged test-only PRs are tracked separately.

**Quality measurement:**
- Production-pass rate = (PRs that didn't require a hotfix within 7 days) / (total PRs merged).
- Measured against the same set of feature types in v1 and v2 to control for difficulty.

---

## v1 Cost Breakdown — 7 personas on Anthropic API

The team configuration is captured in `/root/.openclaw/openclaw.json` (`agents.list`). Each agent runs as a distinct persona inside the OpenClaw gateway, with its own system prompt and model assignment.

### Per-agent monthly burn (at production cadence)

| Agent | Emoji | Model | Why this model | Relative monthly cost share |
|---|---|---|---|---|
| `main` | 🤖 | **Haiku 4.5** | Router only — cheapest model for "decide which agent gets this message" | Negligible — one call per incoming Telegram message |
| `ceo` | 👔 | **Sonnet 4.6** | Strategy + prioritization needs judgment but not max-context coding ability | Modest — occasional strategy calls, scope decisions, dispatch routing |
| `pm` | 📋 | **Opus 4.6** | Ticket triage + sprint scoping requires deep context across the codebase | Substantial — every issue routes through PM |
| `dev` | 💻 | **Opus 4.6** | Primary feature lane — needs the strongest model for complex changes | **Largest single line item** — writes the most code, longest context |
| `bugfix` | 🐛 | **Opus 4.6** | Bug-fix lane (functionally the 2nd dev) — same depth of code work | Substantial — fields bug reports in parallel with Dev |
| `qa` | 🔍 | **Opus 4.6** | E2e + integration testing requires reasoning about full request flows | Substantial — runs against every PR |
| `qa2` | 🧪 | **Opus 4.6** | Second QA lane — divided coverage with QA (e.g. one does app, one does api) | Substantial — parallel with QA |
| **Total** | | | | **Material monthly burn at production cadence** |

### Where the money went

**Five agents on Opus 4.6 is the headline cost driver.** PM, Dev, Bugfix, QA, and QA2 — all on the most expensive model. The reasoning at the time:
- **PM** sees the full ticket context + the relevant code surfaces — needs deep reasoning to scope correctly.
- **Dev** and **Bugfix** write production code that humans review and merge — needs the strongest model for non-trivial features.
- **QA** and **QA2** write integration tests that must understand the full request → response → DB chain — needs reasoning across multiple subsystems.

CEO was downgraded to Sonnet 4.6 because strategy/scope calls don't need max-context-window reasoning — they need judgment, which Sonnet handles well at meaningfully lower cost. The `main` router runs on Haiku because "which agent gets this message" is a near-trivial classification task.

Cost-per-agent ratio was roughly: **Dev > Bugfix > PM > QA ≈ QA2 > CEO > main** — with Dev consuming the majority of monthly Anthropic spend at peak cadence.

### Cost scaling behaviour

v1 cost scales **linearly with throughput** — each additional PR triggers roughly the same PM + Dev + QA pipeline, with the same Opus 4.6 token consumption:

- **Low cadence** — bill stays manageable
- **Moderate cadence** — bill becomes a real line item worth noticing
- **High cadence** (production) — bill climbs faster than your willingness to write the check at small-team revenue scale

There's no economy of scale. This is the fundamental problem with metered API pricing for high-throughput multi-agent pipelines: there's no flat-rate option until you negotiate enterprise pricing, which typically requires committed spend a small team can't sensibly budget.

### What worked well in v1 (despite the cost)

- **Parallel agent execution.** Dev and Bugfix could work on separate features simultaneously. QA and QA2 could test independently. The gateway handled fan-out naturally.
- **Strict authorization.** The `subagents.allowAgents` whitelist enforced the org chart — Dev couldn't bypass QA, QAs couldn't dispatch further work, CEO couldn't be confused with PM. This prevented runaway loops.
- **dev-watchdog auto-recovery.** The 10-minute cron caught stalled Dev sessions, auto-committed WIP, re-dispatched with context. Critical for overnight or unattended runs where a session timeout would otherwise lose hours.
- **Skills library.** OpenClaw's 23 pre-built skills (`dispatching-parallel-agents`, `task-planning`, `verification-before-completion`, etc.) gave each agent strong baseline capabilities without bespoke prompt engineering.

---

## v2 Cost Breakdown — Hermes Agent on Subscription Tier

### Monthly burn (flat, independent of throughput)

| Component | Subscription | Relative cost |
|---|---|---|
| **Codex orchestrator** | OpenAI Plus or Pro | Standard consumer-tier subscription |
| **Claude Code CLI executor** | Claude Max | Premium consumer-tier subscription |
| **Telegram Bot API** | Free | — |
| **Local infrastructure** | Docker on Joao's Mac | None (existing) |
| **Hermes Agent software** | MIT (open source) | None |
| **Total** | | **Predictable, bounded flat monthly cost — well below v1's metered burn** |

### What changed

The architecture is *fundamentally different*. In v1, seven agents with distinct personas, models, and dispatch authorizations operated in parallel. In v2, **one Hermes Agent acts as autonomous dispatcher** — it receives the request, reasons about it via Codex, prompts Claude Code CLI to do the actual work, and journals everything to `runlog/`.

The team-of-roles structure is **gone in v2**. What's preserved:
- **The workflow**: GitHub-issue-driven (`hermes-ready` label is Hermes's queue), Telegram-controlled, human-approves-merges
- **The repos**: same `/root/psinest-{app,api}/` directories (now mounted into the Docker container)
- **The deploy path**: same GitHub Actions workflow_dispatch → VPS
- **The code executor**: same Claude (just invoked under subscription instead of metered API)

What's lost:
- **Parallel execution**: subscription rate limits cap how many concurrent Claude Code invocations you can make. Effectively serial.
- **Role specialization**: no separate Dev vs Bugfix vs QA vs QA2. Hermes routes everything through the same Claude Code prompt.
- **Auto-recovery via watchdog**: replaced by Hermes's built-in PR-state monitoring (SOUL.md instructs Hermes to check existing PRs before re-dispatching).

### Cost scaling behaviour

v2 cost is **flat regardless of throughput** until you hit subscription rate limits:

- **Low cadence** → flat subscription cost, comfortably bounded
- **Moderate cadence** → still flat subscription cost
- **High cadence** → flat cost holds, *but* spillover into degraded service (queued or refused calls due to rate limits)
- **Very high cadence** → impossible at single subscription; would need to add subscriptions or fall back to v1

The subscription model has a **hard throughput ceiling**. Once you exceed Claude Max's rate limits, requests queue or fail. This is the binding constraint, not the agent architecture.

### What you give up moving from v1's team-of-7 to v2's single dispatcher

- **Coverage breadth.** v1 could have PM scoping Issue A while Dev implemented Issue B while QA tested PR C — three things at once. v2 does them sequentially.
- **Specialization.** v1's QA had a different system prompt from Dev, optimized for test-writing reasoning. v2 uses the same Claude Code prompt for everything; specialization is via prompt content, not separate sessions with persistent memory.
- **Persistent per-role memory.** v1 stored per-agent SQLite memory (`/root/.openclaw/memory/<role>.sqlite`). v2 has Hermes's persistent memory but it's one shared memory bank, not per-role.
- **Watchdog auto-recovery.** v1's `dev-watchdog.sh` was specifically tuned for the Dev agent. v2 has Hermes's general PR-state monitoring but it's not as targeted.

---

## The math at different cadences

The crossover point — where v1 and v2 cost the same — sits in the **moderate-cadence** range. Below it, v2 is cheaper. Above it, v2 either doesn't work (rate-limited) or you'd need to layer multiple subscriptions, which erases the savings.

```
Monthly cost
    │
    │                                              v1 (linear)
    │                                          /
    ┤                                      /
    │                                  /
    │                              /     ← crossover (moderate cadence)
    │                          /
    ┤━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ v2 (flat until rate limit)
    │                                                              │ rate-limit ceiling
    │                                                              ↓
    └────────────────────────────────────────────────────────────────→ PRs/week
```

**For Psinest specifically:** post-pilot cadence sits well under the crossover. v2 wins decisively at this cadence.

**During launch crunch:** v1 historically burst to far higher cadence for a few weeks. v2 can't sustain that — for a future launch, we'd temporarily re-enable v1 for launch-critical PRs and let v2 handle maintenance work in parallel.

---

## What the numbers don't capture

Cost-per-PR is a useful metric but misleads in four places:

### 1. Latency cost

v2 is slower. A PR cycle that completes quickly on v1 (with parallel Dev + QA execution) takes meaningfully longer on v2 (serialized through one dispatcher). If your team is blocked waiting on a PR to merge before unblocking other work, **engineer-hours-blocked is a real cost** that doesn't show up in API spend. For Psinest where Joao reviews on his own schedule via Telegram, this cost is roughly zero. For a 5-engineer team with synchronous standups, it would matter.

### 2. Concurrency loss

v1's team could have **3-4 things in flight simultaneously** — Dev working a feature, Bugfix patching a bug, QA testing a PR, PM triaging a ticket. v2 does one thing at a time. For a single human reviewer this is *good* (no firehose of parallel PR notifications). For a larger team it's a bottleneck.

### 3. Architectural complexity hidden in v1's per-agent prompts

v1 required maintaining 7 distinct system prompts + dispatch authorization rules + the watchdog logic. v2 has one Hermes SOUL.md + one Claude Code prompt template. **v2 is fundamentally simpler to operate** — fewer moving pieces to debug when something goes wrong. That has its own value, not captured in monthly euros.

### 4. Failure-rate differences

I tracked production-pass rate carefully across the cutover. Result: **no measurable difference**. v2 isn't lower-quality output — it's the same Claude doing the same work, just billed differently and one-at-a-time. This was the critical finding that made the migration safe to do. If quality had dropped, the cost saving wouldn't have been worth it.

---

## Decision framework — when to migrate v1 → v2

This is a **sequence decision**, not a "pick one" decision. The right answer changes over the life of the pipeline.

```
Are you building or running?
            │
    ┌───────┴───────┐
   Building       Running
  (phase 1)     (phase 2)
    │               │
    │       Is the bill hurting?
    │               │
    │       ┌───────┴────────┐
    │      No                Yes
    │       │                 │
   v1      v1            Has the pipeline
  (team of  (no reason       stabilized?
  7 on API; to migrate           │
  premium   yet)         ┌───────┴────────┐
  models)                No              Yes
                          │                │
                          v1            Migrate to v2
                  (keep iterating       (single dispatcher,
                   on agent prompts;     start with hermes-ready
                   v2 doesn't tolerate   maintenance lane,
                   prompt churn well)    expand from there)
```

**Migration triggers — both must hold:**

1. **Monthly bill is a non-trivial line item** in your operating costs. Not "I'd prefer it were lower" but "this is shifting how I think about hiring vs tooling." If the bill is genuinely negligible relative to revenue or runway, premature migration to v2 will cost you more in throughput than it saves in dollars.

2. **The pipeline has stabilized.** Your agent role definitions, system prompts, and dispatch rules haven't materially changed in the last 4-6 weeks. Production-pass rate is consistent. You're not actively re-tuning what each role does. If you're still iterating on prompts, **stay on v1** — Opus 4.6 forgives prompt-engineering mistakes that subscription tiers won't.

If only one of those conditions holds, **don't migrate yet.** Wait until both line up.

**Key cadence thresholds for Psinest:**
- Under crossover (Psinest's typical post-pilot state) → v2 alone is fine
- Above crossover but sustainable → keep v1 ready for the burst portion, run v2 for the rest
- Launch crunch sustained above crossover for several weeks → temporarily re-enable v1 and route everything through it; accept the bill

---

## The lesson

Across both generations the architecture went from **"team of 7 specialized agents in parallel"** (v1) to **"one autonomous dispatcher in serial"** (v2). The code executor stayed the same (Claude). The bill model changed (metered → flat). The throughput changed (parallel → serial). **The workflow on the human side didn't change at all** — Joao still gets Telegram notifications, still approves merges, still triggers deploys.

The interesting part for any team building AI-augmented systems in 2026 is that **most of the cost decisions you'll face aren't model decisions** — they're tier + architecture decisions. Same model, different tier, different cost curve, different concurrency ceiling. The right answer depends on **what phase you're in** and what your throughput profile looks like, not on quality requirements (which don't change between tiers).

Most teams over-spec the quality side (always Opus, always max-context, always API-tier, always parallel-team) and under-spec the *phase* side. The cheapest way to reduce your bill is usually not to pick a worse model — it's to recognize that you've left phase 1 (build & validate the multi-agent pattern) and you're in phase 2 (run the validated pattern at sustainable burn), and the right architecture *and* billing tier are different for each phase.

**Phase 1 mistake:** premature migration to subscription tier before agent prompts are stable. Subscription rate limits constrain iteration, and you'll spend weeks re-tuning prompts because Claude under rate-limit doesn't recover from sub-optimal prompting as gracefully as Opus on API. You'll think it's the model and switch back to v1, having wasted time.

**Phase 2 mistake:** never migrating. The team works, the bill arrives, you tell yourself "5 agents on Opus is what makes it work, I can't downgrade." But the migration isn't downgrading the model — it's restructuring around a single dispatcher with the same Claude underneath. Same code quality, fewer parallel calls, predictable monthly cost. The bill stops compounding.

---

## What to do with this analysis

If you're running a multi-agent pipeline on Anthropic API and your bill is uncomfortable:

1. **Check phase first.** Have your agent role definitions stabilized? If you've edited any agent system prompt in the last 4 weeks, you're still in phase 1. Stay on v1; iteration speed matters more than the bill right now.
2. **If you've crossed into phase 2** (prompts stable, production-pass rate consistent), then:
   - **Measure your throughput.** PRs/week, not tokens/week. The tier decision is throughput-bound, not token-bound.
   - **Estimate the crossover.** Plot v1 linear cost vs v2 flat cost at your throughput. Where do they cross?
   - **Pilot v2 on a low-stakes lane.** Tag a category of work `hermes-ready` (in Psinest's case: bug fixes, refactors, regression specs). Route those through v2; keep feature work on v1. Run hybrid for 2-4 weeks, measure quality and latency, then progressively expand v2's territory.
3. **Don't kill v1 when you migrate to v2.** Keep the systemd unit definition and the wrapper script in place (just `systemctl disable openclaw.service` so it doesn't auto-start). Keep the dev-watchdog cron entry commented out, not deleted. The optionality to spin v1 back up for a launch crunch is worth more than the marginal maintenance overhead.

---

## See also

- [`./architectural-tradeoffs.md`](./architectural-tradeoffs.md) *(TBD)* — team-of-7 hierarchy vs single dispatcher; what each costs in complexity terms
- [`./model-routing.md`](./model-routing.md) *(TBD)* — which model for which role, with per-agent reasoning + what changes under subscription tier
- [`./throughput-analysis.md`](./throughput-analysis.md) *(TBD)* — speed numbers in detail; concurrency lost from v1 to v2
- [`../case-study/v1-lessons.md`](../case-study/v1-lessons.md) *(TBD)* — what running the 7-agent team taught us
- [`../case-study/v2-lessons.md`](../case-study/v2-lessons.md) *(TBD)* — what running the single dispatcher taught us
- [`../README.md`](../README.md) — full architecture overview + sequencing playbook
