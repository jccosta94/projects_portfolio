# Shared Architecture Patterns
**Cross-project patterns I keep coming back to — extracted from the five case studies in this portfolio.**

This is a working catalog of the architectural patterns that show up more than once across [Hugo](../hugo/), [Psinest](../psinest/), [OpenClaw-Hermes Evolution](../openclaw-hermes-evolution/), [Skoda AI Ops](../skoda-ai-ops/), and [Claude Cowork Vertical Workflows](../claude-cowork/). Each pattern below describes the problem it solves, the shape of the response, the places it's exercised in this portfolio, and the conditions under which I'd reach for it on the next project. The patterns aren't independent — several of them compose into the same system, and the connections matter as much as the patterns themselves.

The portfolio is small (five case studies, three production-live, two in pilot), so this catalog is deliberately concrete and example-led rather than abstract. Each pattern has at least one production-deployment reference; the dates and pilot footprints are visible in each case study's `## Status` section.

---

## Pattern 1 · Centralised access gate at the query layer

**The problem.** In a multi-tenant system with several role-distinct surfaces, *tenancy enforcement is the single largest source of regulatory-and-trust risk*. The naive model — every controller adds a `where tenantId = currentTenantId` filter — loses by construction, because every new endpoint is a new opportunity to forget the filter. The first endpoint that ships without it is the first patient-record-cross-tenant-leak.

**The pattern.** Centralise tenancy enforcement at the *query construction layer*, not the controller layer or the client filter layer. Every list query passes through one gate that derives scope from the authenticated identity (JWT, session token, signed claim — whatever the framework provides), resolves the operator's role, and applies the scope to the EF Core / ORM query construction before it leaves the API. No controller can opt out of the gate; no client filter can override it. The gate's invariant is *if the gate function ran, scope is enforced; if it didn't run, no data comes back*.

**Where it's exercised.**
- [Psinest's `RoleScope.cs`](../psinest/README.md#security-architecture--role-scoped-access-via-rolescopecs) — the canonical implementation. ClinicOwner blocked from documents entirely. Patient reads own only. Psy reads partnered clinics' patients only with valid partnership + consent rows. Every list endpoint passes through `RoleScope.Apply()` at query construction time.
- [OpenClaw-Hermes Evolution's dispatch authorization](../openclaw-hermes-evolution/README.md#routing-and-authorization) — the same pattern applied to agent-to-agent dispatch instead of data access. `subagents.allowAgents` is a centralised whitelist; no agent can dispatch outside its allowlist; the whitelist lives in one place and audits in one place.

**When to reach for it.** Any multi-tenant system, but the cost-benefit tilts hardest when (a) the data class is regulated (healthcare, legal, financial), (b) the audit requirement names *which-role-saw-what* not just *which-record-was-accessed*, or (c) the surface is growing (new endpoints land faster than security review can catch up). The discipline cost of centralising the gate is small; the cost of *not* doing it scales with every endpoint.

**Trade-off.** The gate function becomes a chokepoint. Every change to scope semantics flows through it, which is exactly what makes it auditable but also makes it a hot spot under heavy schema evolution. The Psinest convention — `RoleScope.cs` is the *only* place tenant resolution lives, and any new role-affecting feature has to extend it — keeps the chokepoint coherent over time.

---

## Pattern 2 · Parallel-tier capability gate (compliance-gated AI capability rollout)

**The problem.** Adding AI capabilities to a regulated platform introduces a *second* enforcement concern, distinct from data access. "Who can read this patient's document" is one question; "what AI can do with this document once it's read, by whose consent, with what audit trail, against what cost budget" is a different one. Bolting AI permissions onto the data-access gate conflates the two — the consent semantics, cost shape, and audit requirements are different enough that one gate doing both jobs ends up doing neither well.

**The pattern.** Give AI-capability access its *own* enforcement tier, parallel to the data-access tier, operating one layer higher in the request flow. Data-access gate (Pattern 1) decides what the API can return; AI-capability gate decides what the API can *do with that data once retrieved*, before it reaches an external service. The AI gate enforces per-capability consent (a patient might consent to mood-journal AI but not to session transcription), always-on PII redaction (middleware-level, not opt-in per call), per-capability cost budgets (one runaway capability shouldn't eat another's monthly budget), immutable audit log emission (every AI invocation logged with model + version + input hash + output + downstream action), and model-version pinning (so audited outputs are reproducible).

**Where it's exercised.**
- [Psinest's AI Policy Manager](../psinest/README.md#ai-augmentation-roadmap-future-state) — the canonical design. Operates one tier above `RoleScope`. Lands before any external-LLM call against patient data. Documented as the future-state centerpiece for 8 AI capability domains (transcription, clinical insights, treatment planning, etc.).
- [Claude Cowork's per-vertical HITL gate model](../claude-cowork/README.md) — same logic, different surface. Each vertical (Practice Command / Legal Efficiency Suite / Field Service Command) tunes the gate independently: owner-eyes-only on patient identifiers in healthcare, supervising-lawyer review on every drafted document in legal, dispatcher-confirms in home services. The data-access layer is the same Cowork; the AI-action gate is per-vertical.

**When to reach for it.** Whenever an AI capability touches data that has its own consent regime (regulated industries, B2C platforms with explicit data-use opt-ins, multi-tenant SaaS with customer-data-residency commitments). The fork to look for: *if a user can plausibly consent to seeing the data but not consent to AI processing the data*, you need two tiers.

**Trade-off.** Two gates = two places to check. The discipline of keeping them independent — RoleScope answers "data?", Policy Manager answers "action on data?" — is what keeps the audit story coherent. If they start overlapping, you've built one confused tier instead of two clear ones.

---

## Pattern 3 · Single agent + specialists-as-MCP-tools (the topology collapse)

**The problem.** Multi-agent designs feel natural for any product where work decomposes into specialties (a marketing team has 6 specialties; an engineering team has 7). The temptation is to map specialties to agents 1:1 and let an orchestrator handle dispatch. **For cheap LLMs — the tier you need to make per-customer subscription pricing viable — multi-step delegation contracts don't bind reliably.** The failure mode is silent: the model synthesizes plausible content from soft preconditions instead of executing the procedural step the SKILL.md prescribed.

**The pattern.** Collapse the procedural contract into a *single tool call per turn*. The agent's only job is to pick a tool, invoke it once, and summarize the customer-facing reply from the tool's return value. Specialist capabilities move from separate agents to area-scoped MCP tool servers. There are no soft preconditions for the model to fail on — the tool either returned a result or it didn't. The agent's behaviour reduces from "execute a procedure" to "summarize a tool return".

The canonical skill template for this pattern (the *Pattern C SKILL.md*) has six structural sections: signature triggers · numbered tiers · GOOD/BAD examples · counterfactual guardrail · forbidden-tokens list · state-readback rule (state queries re-invoke tools rather than replaying from session memory).

**Where it's exercised.**
- [Hugo v2](../hugo/README.md) — the canonical case. v1 was a 7-agent team that broke under three independent multi-agent delegation failures; v2 collapsed to a single agent + 8 area-scoped MCP namespaces (`marketing__*`, `creative__*`, `web__*`, `inbox__*`, `outbound__*`, `analytics__*`, `audit__*`, `content_plan__*`). Side effect: 5–7 inference calls per customer turn dropped to 1–2.
- [Hermes-Psinest (OpenClaw v2)](../openclaw-hermes-evolution/README.md) — same architectural shape applied to a build pipeline. A single Hermes dispatcher orchestrates Codex (gpt-5.4-mini brain) which shells out to Claude Code CLI (code executor) per turn. Same "one tool call per turn, summarize the return" discipline.

**When to reach for it.** When the model class you're locked into (because of per-customer pricing, on-prem constraint, latency budget, or any other reason that makes a model upgrade infeasible) can't bind multi-step delegation reliably. Don't wait for a better model — change the topology.

**Trade-off.** You give up concurrency. The multi-agent team did work in parallel; the single-agent topology serializes. For low-volume per-customer workloads this is invisible; for high-throughput pipelines (Psinest's build cadence on busy days) it's the trade you accept for predictable cost.

---

## Pattern 4 · Deterministic library between LLM output and public API

**The problem.** When LLM output is going to be published *publicly* — to a website, a regulated advertising channel, a customer's social feed, an API call that can't be retracted — a 95%-reliable system is a 5%-regulatory-exposure system. Prompt-level instructions ("don't use the word `Festpreis`", "always hedge prices with `ab`") are advisory at best. The system has to *refuse to publish* content that violates the rules, not be asked nicely to comply.

**The pattern.** Move the rules from the prompt to a deterministic library (Python regex with positive context, in Skoda's case ~300 LOC). The library runs *before* any publishing API call fires. Every payload is checked for the rules that constitute compliance for the domain (German-language signals, price hedging, regex denylist with word-boundary enforcement, channel-specific voice). **If any check fails, the publishing API call does not fire.** The LLM is allowed to be wrong; the library refuses to publish until it isn't. The library is *not smart; it is right* — that's the point.

The pattern decomposes into: (a) the rule library is the *only path* to the publishing surface (the LLM and the human operator both call through it), (b) defense-in-depth on approval — the library re-runs on the approved payload because the payload may have been re-encoded since approval, (c) channel-specific routing applied *after* the library passes (so the same checked payload fans out to multiple channels with channel-specific framing).

**Where it's exercised.**
- [Skoda AI Ops](../skoda-ai-ops/README.md) — the canonical case. The Python compliance library is the load-bearing piece; brand voice can live in a prompt, compliance cannot. Generalises to any commercial AI system in a regulated domain.
- The same pattern is the *bones* of Psinest's planned PII redaction proxy ([AI Policy Manager](../psinest/README.md#ai-augmentation-roadmap-future-state)) — redaction is always-on middleware before any external-LLM call, not opt-in per call.

**When to reach for it.** Any time LLM output crosses a regulated or trust-bearing boundary. The fork to look for: *would a single bad publish cost the business significantly more than the engineering effort to write the rule library?* For Skoda's German consumer-protection regime, yes. For Psinest's RGPD scope on patient data, yes. For an internal Slack draft assistant, no — the human reads it before sending.

**Trade-off.** The library is brittle in proportion to how brittle the rules are. Skoda's library is permissive (blocks only affirmative-English-with-zero-German, not strict German enforcement) because an over-strict version blocked legitimate German posts using cognates. The library has to evolve as the false-positive set is discovered — but that's a discipline cost, not an architectural fault.

---

## Pattern 5 · Three human-in-the-loop gates (priority · merge · deploy)

**The problem.** Autonomous-execution systems that "ship code" or "publish content" or "send messages on the operator's behalf" need a clear answer to *what does the agent decide vs. what does the human decide?* Two failure modes bracket the design space: too few gates = agents make decisions they shouldn't (which feature ships, which PR merges, when to deploy); too many gates = the human is the bottleneck the agent was supposed to remove.

**The pattern.** Three gates per cycle, no more, no fewer:
1. **Priority gate** — the agent proposes from a curated queue (kanban board, issue label, intake form); the human picks what to execute.
2. **Output gate** — the agent produces a draft (PR, document, post, dispatched job); the human reviews and approves the draft. No agent self-merges product code; no agent publishes public content; no agent sends an externally-visible message.
3. **Effect gate** — the human triggers the side-effect operation (deploy, publish, send). Trigger is manual (`workflow_dispatch`, "post now" tap, "send" click).

The agent's job is everything *between* the gates: scope the work, write the code or content, run tests, prepare the message, draft the proposal. The human stays in the loop on the high-blast-radius decisions; the agent absorbs the rest.

**Where it's exercised.**
- [OpenClaw v1 (Anthropic-API metered) + Hermes v2 (subscription flat)](../openclaw-hermes-evolution/README.md) — same three gates carry verbatim across both generations of the build pipeline. The agents got cheaper; the gates didn't move. Joao approves priorities, reviews every PR, triggers every deploy manually.
- [Skoda's editorial pipeline](../skoda-ai-ops/README.md) — Telegram preview approval (minutes, not hours) is the output gate; the operator's `ok` triggers the publishing call (effect gate); the priority gate is the editorial calendar.
- [Claude Cowork's per-vertical HITL model](../claude-cowork/README.md) — three gates per cycle, tuned per vertical. Healthcare = owner-eyes-only on patient identifiers; Legal = supervising-lawyer review on every drafted document; Field Services = dispatcher confirms every job assignment.

**When to reach for it.** Any system where the agent operates on the human's behalf with externally-visible consequences. The fork to look for: *can the action be reversed cheaply if the agent gets it wrong?* If no, gate it.

**Trade-off.** The human is in the loop for every cycle. For high-cadence pipelines (Psinest's daily PR throughput), this is real time cost. The OpenClaw/Hermes case study documents the answer: the gates aren't optional, but the *time-cost-per-gate* can be tuned (Telegram approval beats email approval; one-tap mobile approval beats desktop review). The architecture stays; the latency budget gets engineered down.

---

## Pattern 6 · Verticalisation as configuration, not code

**The problem.** A consulting practice that ships custom AI builds per customer has a cost-structure that can't survive past the third engagement — each build is a snowflake, the second customer's build can't reuse anything from the first, and the cost-of-delivery swallows the engagement fee. The inverse failure is also real: a single horizontal package sold to every vertical is too generic to move the operator's bottleneck in any given vertical.

**The pattern.** Fix the platform, the cockpit shape, and the skill-library *structure*. Vary the wiring on either side of the cockpit, per vertical. What's per-vertical: the MCP fleet (which PMS / DMS / dispatch tool), the skill-library *composition* (which core skills + which vertical-specific extensions), the memory seeds (what counts as a VIP, what's special-category data, what's privileged), and the HITL gate model. *None of this is code; all of it is configuration.* Adding a new vertical = adding ~5-7 skills, ~2-3 MCP-server bindings, and a vertical-tuned HITL gate model. The platform doesn't change; the wiring does.

**Where it's exercised.**
- [Claude Cowork Vertical Workflows](../claude-cowork/README.md#the-architectural-insight) — the canonical case. Same Cowork install, three operator cockpits (Practice Command for healthcare clinics, Legal Efficiency Suite for law firms, Field Service Command for home services), varying only the wiring on either side.
- The same logic applies to Hugo's per-tenant Docker pods — each tenant runs the same Hugo agent + same Pattern C skill library + same MCP server fleet; what varies is the per-tenant brand context (BCP.md), customer goals, channel allowlist. Per-tenant *configuration*, not per-tenant *code*.

**When to reach for it.** Any productized service or platform-templated consulting practice where the same underlying capability has to be re-shaped per customer-segment. The fork to look for: *would you rather sell five "bespoke" engagements or five "configured" engagements?* The configured engagements compound; the bespoke ones don't.

**Trade-off.** The discipline cost: you have to refuse to add custom code for the customer in front of you, even when it would be a one-day win, because the next customer in the same vertical won't have the same custom code. The pattern that ships is the pattern you can reproduce on the second sale without breaking the cost-structure of the first.

---

## Pattern 7 · Configure a reliable platform, don't build a custom agent from scratch

**The problem.** The default move in early-2026 AI consulting is to position every engagement as a "bespoke AI build" — promise a custom agent per customer, scope it from scratch, debug connectors in production, hand-author memory files, ship one snowflake at a time. The cost-of-delivery swallows the engagement fee; the second customer can't reuse the first customer's build.

**The pattern.** Treat the platform vendor's investment in the foundation as the *substrate*, not the competition. When a vendor has built a reliable cockpit, integration plane, runtime, and security model, the highest-leverage consulting work is *configuring that platform for the customer's vertical* — not building bespoke alternatives. Salesforce consultants don't write their own CRM; they configure Salesforce. The same logic applies to AI cockpit delivery: Claude Cowork is the platform; the consulting is the vertical configuration on top.

The pattern composes with Pattern 6 (verticalisation as configuration) — the platform fixes the cockpit shape, the consultancy ships per-vertical configurations on top.

**Where it's exercised.**
- [Claude Cowork Vertical Workflows](../claude-cowork/README.md#the-architectural-insight) — the canonical articulation. Same operating model that worked for years of Salesforce CRM delivery, applied to AI cockpit delivery.
- [Hugo](../hugo/README.md) and [Skoda AI Ops](../skoda-ai-ops/README.md) sit *on top of* Hermes Agent and Claude Cowork respectively. Neither rebuilds the underlying agent runtime or the desktop cockpit. The work that's mine in each case is the architecture *on top of* the adopted platform — Pattern C SKILL template (Hugo), the Python compliance library (Skoda), the vertical-configured skill libraries (Cowork).
- [OpenClaw-Hermes Evolution](../openclaw-hermes-evolution/README.md) — explicitly structured around this pattern. v1 adopted OpenClaw; v2 adopted Hermes Agent; what's *mine* is the hierarchical agent role definitions, the dispatch-allowlist authorization, the three human-in-the-loop gates, the cron-based watchdog auto-recovery. I configure; the platforms provide.

**When to reach for it.** Whenever a credible platform vendor exists in the space. The fork to look for: *would the same customer hire a SaaS consultant for this problem if the platform were Salesforce or Microsoft Dynamics?* If yes, the AI version of the same engagement should follow the same operating model — configure the platform, don't write a competing one.

**Trade-off.** You're betting on the platform vendor. Cowork's roadmap, MCP-spec stability, and runtime reliability are the substrate; if the platform goes sideways, the consulting practice has to follow. The hedge is in Pattern 6 — the per-vertical configuration is portable in shape (skill structure, memory seeds, HITL gate model) even if the substrate has to migrate.

---

## Pattern 8 · Skill-naming as API design

**The problem.** When the agent decides which skill to invoke based on a `description` field (which is how Claude Code, Claude Cowork, and most modern skill-loading runtimes work), the description's wording *is* the routing logic. If two skills have descriptions that overlap in plausible use-cases, the agent picks one of them — wrong about half the time — and the operator's first impression is that the system is flaky. The cost of a confusing description is paid every time the operator asks a question that could plausibly invoke two skills.

**The pattern.** Treat skill names and descriptions like public-API endpoint names: stable, mutually exclusive, descriptive enough that the operator can predict what they'll get without reading the body. Adding a new skill is a deliberate naming decision, not a routine one — the cost is the *whole library's* routing accuracy, not just the new skill's. The convention from Cowork: the line between `admin-inbox-triage` and `admin-meeting-prep` is the line between "what gets pulled up at 09:00" vs. "what gets pulled up before a 14:00 with a specific person on the calendar" — and an "inbox prep for meetings" skill would compete with both, so it's *not* a skill.

**Where it's exercised.**
- [Claude Cowork Vertical Workflows](../claude-cowork/README.md#the-architectural-insight) — the canonical articulation. The 11-skill core library + per-vertical extensions are deliberately named to be mutually exclusive in their routing surface.
- [Hugo's Pattern C skill template](../hugo/README.md) — every skill has a `signature triggers` section at the top that enumerates the canonical phrasings the operator uses; this is what makes the skill mutually exclusive against neighbours in the same area-scope.
- [OpenClaw v1's 23-skill library](../openclaw-hermes-evolution/README.md) — same discipline, applied to a development team's skill catalog (planning · quality · domain · debugging · frontend · workflow · testing categories, with mutually-exclusive naming inside each category).

**When to reach for it.** Whenever skill selection is description-based (which is the norm). The fork to look for: *can I predict, from the name alone, which skill the agent will pick for a given operator phrasing?* If not, the names need to change before the next skill ships.

**Trade-off.** Renaming a shipped skill is a breaking change for any operator who learned its old name. The remedy is to over-invest in naming *at first ship*, not to keep iterating after the fact. Naming a skill costs more than writing it; that's the discipline.

---

## Pattern 9 · Velocity first, economics second (the migration sequence)

**The problem.** AI-augmented systems that ship at production cadence have two cost shapes: the metered shape (per-token or per-API-call billing, scales with throughput, premium per unit) and the subscription shape (flat monthly, bounded regardless of throughput, capped per unit). The metered shape is what makes early validation feasible — you don't know which model class, which agent topology, or which workflow shape works until you ship the workload at real cadence. The subscription shape is what makes the same workload financially defensible once you do.

**The pattern.** Build v1 on the metered shape, validate the workflow at production cadence, then re-architect to the subscription shape once the binding constraint becomes clear. Don't try to optimize cost from day one — that's a premature commitment that constrains the prompt-engineering and topology iteration v1 needs. Don't *ignore* cost once velocity is proven — that's a budget death-spiral. The migration is sequential: prove the pattern works first, then migrate to the architecture that sustains it.

**Where it's exercised.**
- [OpenClaw-Hermes Evolution](../openclaw-hermes-evolution/README.md) — the canonical case. v1 OpenClaw on Anthropic API metered pricing solved velocity (5 worker agents in parallel, no rate-limit ceiling); v2 Hermes Agent on flat subscription solved economics (order-of-magnitude cheaper, same workflow on top). Same three human-in-the-loop gates carry over verbatim.
- [Hugo v1 → v2](../hugo/README.md) — the same architectural sequence on a different constraint axis. v1's 7-agent topology was the velocity bet; v2's Pattern C collapse was the capability + cost-efficiency response. Two different platform pivots, same architectural sequence.

**When to reach for it.** Any AI-augmented system that has to validate at production cadence before optimising cost. The fork to look for: *am I confident enough about the workflow shape to make a long-term cost commitment?* If no, ship on metered first.

**Trade-off.** The metered phase is *meaningfully* more expensive while it lasts. The OpenClaw case study has the numbers; the v1 monthly Anthropic bill became its own problem before v2 shipped. The trade-off is real and visible — but the alternative (running on subscription tier from day one with an unvalidated topology) leaves more value on the table than the metered-phase burn costs.

---

## How these patterns relate

| Pattern | Composes with | Why |
|---|---|---|
| 1. Centralised access gate | 2. Parallel-tier capability gate | Pattern 2 is *one tier above* Pattern 1. Both centralise their enforcement; together they give a two-tier compliance story (data access · capability invocation) that's auditable independently. |
| 3. Single agent + MCP tools | 5. Three HITL gates · 8. Skill-naming | Pattern 3 changes the agent's behaviour from "execute a procedure" to "summarize a tool return"; Pattern 5 wraps the gates *around* the agent regardless of which shape it has; Pattern 8 makes the tool-selection step actually work. |
| 4. Deterministic library | 2. Parallel-tier capability gate · 5. Three HITL gates | Pattern 4 is the *implementation pattern* for one specific gate in Pattern 5 (the output gate). When the gate is "compliance check on public-facing content", Pattern 4 is how you build it. |
| 6. Verticalisation as configuration | 7. Configure a reliable platform · 8. Skill-naming | Pattern 6 says *the platform stays fixed, the wiring varies per vertical*; Pattern 7 says *the platform is the vendor's product, not mine*; Pattern 8 is the discipline that makes the per-vertical wiring actually route correctly. |
| 9. Velocity first, economics second | All of the above | The migration sequence is platform-agnostic. v1's "metered phase" validates which of patterns 1-8 you actually need; v2's "subscription phase" carries them forward at a cost shape that scales. |

---

## What's not here yet

A few patterns are recurring across the portfolio but haven't been written up here yet:

- **Architecture-pause for DB migrations** — the discipline of stopping before generating an EF migration to review the entity diff with a human, then again after generating to review the migration file. Exercised in Psinest's build pipeline.
- **Pre-commit branch guard** — every commit verifies `git branch --show-current` matches the expected feature branch before committing. Prevents cross-agent worktree collisions in the OpenClaw/Hermes multi-agent setup.
- **Layered testing for user-facing changes** — API regression specs aren't enough; pair with full-pipeline e2e specs for anything where the SPA branches on response shape. Exercised across Psinest's build pipeline.

These will land here as the next layer of patterns gets distilled.

---

## Reading order

If you're new to the portfolio and want to walk these patterns through real systems:

1. Start with [Psinest](../psinest/README.md) — Pattern 1 (`RoleScope.cs`) and Pattern 2 (AI Policy Manager) live there in their canonical form.
2. Then [Hugo](../hugo/README.md) — Pattern 3 (the topology collapse) is the architectural through-line.
3. Then [Skoda AI Ops](../skoda-ai-ops/README.md) — Pattern 4 (deterministic library) is the load-bearing piece.
4. Then [OpenClaw-Hermes Evolution](../openclaw-hermes-evolution/README.md) — Patterns 5 (three gates) and 9 (velocity first, economics second) are the migration spine.
5. End with [Claude Cowork Vertical Workflows](../claude-cowork/README.md) — Patterns 6 (verticalisation as configuration), 7 (configure-don't-build), and 8 (skill-naming as API design) are the consulting-practice through-lines.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
