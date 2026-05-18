# Architecture Evolution

How Hugo went from a seven-agent team to a single agent plus specialists-as-MCP-tools, across four weeks and seventeen PRD amendments.

## Where we started

The original Hugo design (April 2026) was a seven-agent team running on the Hermes Agent framework:

- **Director** — customer-facing, handled Telegram conversation, delegated to specialists
- **Performance Marketer** — Google + Meta Ads
- **Web Developer** — landing pages, Google Business Profile
- **Brand Designer** — image / video generation
- **Community Manager** — Instagram / Facebook inbound
- **Outbound Specialist** — cold email
- **Analytics** — GA4 / Meta Ads / Stripe pulls

Each agent had its own `SOUL.md`, `AGENTS.md`, `USER.md`, `MEMORY.md` workspace. Director would translate a customer voice note into a structured brief, ping the specialists, gather their outputs, synthesise a response. Multi-agent orchestration was the load-bearing architectural premise.

This made sense for several reasons:

- It mapped 1:1 to how marketing agencies are structured (specialists with their own expertise)
- Each agent could have its own learning loop and memory
- Parallel execution across specialists could speed up complex campaigns
- I'd shipped this exact pattern before in Psinest (CEO + PM + 2 devs + 2 QAs as separate agents)

## Hosting and brain — the unstable layer

The first three weeks were dominated by trade-offs in two areas neither I nor anyone else had a clean answer to: where to host the agents, and what model to run them on.

### Hosting: Mac mini → VPS

I started with a dedicated Mac mini M4 at home running one Docker container per tenant (Amendment #2). The rationale was clean Anthropic Max ToS — running an LLM subscription on hardware I owned felt defensible.

A week later I moved to a cloud VPS (Amendment #7). The Mac mini setup couldn't scale past my home internet, didn't survive a power cut, and required physical access for hardware issues. The Anthropic Max ToS argument turned out to be moot because I'd also moved off Anthropic by then. VPS with per-tenant Docker isolation gave the same per-customer separation without the home-internet dependency.

### Brain: Anthropic → split with Gemini → all-Codex on ChatGPT Plus

The brain story is more interesting. I started with **Anthropic Max per customer** (Amendment #3) — each tenant signs up for their own Anthropic Max subscription, Hugo shells out to Claude Code CLI inside that authorised session for code-gen tasks.

Then Amendment #8: I dropped Anthropic entirely for **per-customer ChatGPT Plus + Codex CLI**. Same ToS grey area, an order of magnitude cheaper at the per-customer floor, sufficient quality for Hugo's actual code-gen workload (HTML landing pages, Python analytics scripts).

For non-code agent reasoning I kept Gemini Flash 2.5 — large context window, multimodal, fast token gen.

Then Amendment #12: **all agents onto Codex CLI**. A Gemini API failure during phone-smoke testing surfaced how much operational overhead per-customer Gemini key management added (separate billing, key rotations, no Keychain integration). Everything onto one model, one auth path. The trade-off was accepting CLI shell-out latency (~1–3s per turn vs direct HTTP API) and shared rate limits across all agents on one ChatGPT subscription.

### Framework: Hermes → OpenClaw → Hermes

Three frameworks in seven days. The driver: I'd built Psinest on OpenClaw with native multi-agent orchestration, and the seven-agent Hugo design was a natural fit for that pattern. So Amendment #10 swapped Hermes for OpenClaw.

Then Amendment #17 swapped back to Hermes. By then the architecture had collapsed to single-agent (next section), and OpenClaw's multi-agent value prop didn't apply. Hermes's personal-assistant-first design fit the new shape better.

## The collapse — seven agents to one

Amendment #16 was the big one.

After getting OpenClaw running, I started shipping the canonical V1 features: presence audit, content plan execution, ad-hoc social posting. All required Director to delegate to a specialist, get a result back, synthesise the customer response.

Three independent multi-agent delegation failures, all under `gpt-5.4-mini` (OpenAI's Codex-CLI default):

**Issue #83 — audit delegation.** Director was supposed to invoke Performance Marketer to run an account audit. Instead, Director made up audit findings. The model decided that synthesising plausible audit content was an easier path than executing the delegation contract.

**Issue #81 — Brand Designer topic confab.** Director was supposed to convene a multi-step Brand Designer skill to draft a topic. Instead, it asked the customer for input and then fabricated the topic from that. The model took the "ask the customer" escape valve instead of running the prescribed flow.

**Issue #86 — Director→BD post drafting.** Same pattern: a delegation contract with soft preconditions, the model resolved them by improvising.

Three failures, one root cause: **`gpt-5.4-mini` doesn't bind soft procedural framing.** When you tell it "first call X, then synthesise Y from X's output, do not skip step 1," and step 1 has a plausible alternative (ask the user, fabricate from context, look at session memory), the model takes the alternative. The contract is treated as a suggestion.

Two options:

- **Fix the model.** Wait for a model capable enough to bind multi-agent delegation reliably. The next tier (Codex with `gpt-5-pro`) might do it, but at higher latency and cost, and the wait was indefinite.
- **Fix the topology.** Collapse the multi-step delegation into a single tool call, and let the agent's only job be to synthesise the customer summary from the tool's return value.

I chose the second. Three failures with the same shape made it clear the multi-agent pattern wasn't a tooling problem I could iterate around — it was a model-capability assumption that the cheap model couldn't satisfy, and the expensive model wasn't a clean substitute either.

## What collapsed and what stayed

**Collapsed:**

- Seven OpenClaw agents → one Hermes agent per tenant
- Specialist `SOUL.md` / `AGENTS.md` workspaces → archived as reference material
- `agent_send` inter-agent routing → irrelevant
- `tenants/<id>/openclaw/agents/<role>/workspace/` directories → one Hugo workspace per tenant
- `install-tenant.sh` registering 8 agents → registers 1 (plus the Hermes built-in `main`)
- Specialist learning loops as per-agent `MEMORY.md` files → domain expertise becomes tool-internal state instead

**Stayed:**

- Customer-facing model (customer talks to Hugo — specialists were always internal)
- Autonomy tiers and HITL gates (still code-enforced, still block public actions)
- Plans-as-source-of-truth
- Telegram as default channel
- Codex CLI inference
- Upload-Post for social posting
- Instagram-only V1 inbound scope
- VPS hosting + per-tenant Docker container isolation

The customer-facing thing didn't change at all. The internal architecture changed completely.

## What Pattern C is

The new canonical skill template, validated five times across the shipped V1 features:

For any specialist capability, all deterministic mechanical work goes in a single MCP tool. Hugo's role is to invoke the tool once and synthesise the customer-facing summary from the tool's return value. Hugo does not perform multi-step procedural work inline.

Why it works: Mini's *"synthesise plausible content from soft preconditions"* failure mode requires soft preconditions to fail on. Pattern C removes them. The tool either returned a result or it didn't. Hugo synthesises **from** the result, not from inference, not from BCP contents, not from prior session memory.

The full Pattern C writeup is in [`../orchestration/mcp-integration.md`](../orchestration/mcp-integration.md).

## Trade-offs accepted

- **Tool-surface concentration.** One agent now navigates a wider tool menu (10+ MCP tools across 8 namespaces). Risk: wrong-tool-selection at a different scale than the soft-procedural-framing problem. Mitigation: tight tool descriptions + per-tenant tool allowlist.
- **No specialist learning loops as separate agent memories.** Domain expertise becomes tool-internal state, not per-agent `MEMORY.md`. Loss: harder to develop "expert" personalities for each specialist. Acceptable: no customer needed that.
- **No cross-agent parallelism.** Hugo serialises. Long-running tool work blocks customer messages unless an async pattern is built. Deferred to post-V1.
- **Reversibility is moderate.** If multi-agent delegation becomes reliable on a future model class, MCP tools can be refactored back into agent skills. Cost: 1–2 weeks of focused work.

## What I learned

**The "fix the model" instinct is usually wrong.** When a model-capability constraint shows up, the temptation is to wait for the next tier. But model release cadence is unpredictable and per-token cost is unpredictable. Topology changes you control on your own timeline. Pick those when the option exists.

**Multi-agent patterns are an LLM-capability bet.** If the model can bind soft procedural framing reliably, multi-agent is straightforward. If not, every cross-agent contract becomes a probabilistic gamble. There's a long window where the cheaper models can't do it and the expensive ones cost too much per call to be viable in production. Single-agent + tools sidesteps the whole question.

**Frameworks are less load-bearing than they look.** Three framework pivots in seven days, none of which changed what the customer experiences. Frameworks shape how the team is organised and how skills are authored — they don't shape what gets built. Pick the one that fits the topology and stop iterating.

**Writing the PRD amendments as I went paid for itself.** Seventeen amendments in four weeks, each with reasoning logged at the time. When I needed to reverse a decision (Hermes → OpenClaw → Hermes), I had the original reasoning to reverse against. Without that, it would have felt like flip-flopping. With it, it was a transparent sequence of "here's what I knew, here's what changed."
