# Velocity first, economics second
**A sequencing pattern for AI-augmented systems that have to validate at production cadence before they can be optimised.**

*~1,500 words · drawn from the [OpenClaw → Hermes migration](../openclaw-hermes-evolution/) and [Hugo v1 → v2](../hugo/)*

---

There's a temptation, when designing an AI-augmented system that has to ship at production cadence, to start by asking what it costs. This is the wrong first question. Asked too early, it locks in a model class, a topology, and a billing tier before you know whether any of the three actually works for the workload. The right first question is whether the system can produce the output you need, at the cadence you need it, *at any cost*. Once that's answered, you have a workload to optimise. Trying to optimise before that is a premature commitment.

This essay is about the sequence. It's a pattern that shows up in both case studies in this portfolio where I had to migrate a working system to a cheaper architecture — the OpenClaw → Hermes Agent build-pipeline migration that ships [Psinest](../psinest/), and the Hugo v1 → v2 collapse from a 7-agent team to a single-agent + MCP-tools topology. The two systems are unrelated; the sequence is the same.

## The trap is the cost question

The default move when designing an AI system in 2026 is to start with cost: which model, which tier, what's the per-call price, what's the monthly burn at projected throughput. This is what an engineering team sounds like when they don't yet know whether the system works. It's a substitute for the harder question, which is whether the system can produce reliable output at production cadence at all.

The trap is that the cost question has an obvious-sounding answer: pick the cheapest model that can plausibly do the work, and design the system around it. This is what I did in the first version of Hugo. The constraint was per-customer pricing — each tenant runs Hugo on their own ChatGPT Plus subscription, which gates inference to `gpt-5.4-mini`. I designed v1 around that constraint from day one: a 7-agent team, each specialist on `gpt-5.4-mini`, with a Director routing work between them. The math worked on paper. The system didn't.

It didn't work for a reason I couldn't have predicted from a cost spreadsheet: `gpt-5.4-mini` doesn't bind multi-step delegation contracts reliably. When the SKILL.md tells the model to "first call X, then synthesize Y from X's output", the model treats the procedural step as a suggestion. If there's any plausible alternative — ask the customer, fabricate from context, replay from session memory — the model takes the alternative. Three independent multi-agent delegation failures broke v1 before it shipped to customers. The cost design was correct; the topology was wrong; you can't tell the difference from a cost spreadsheet.

## What "velocity" actually means

Velocity, in this context, isn't speed. It's the ability to validate the design at production cadence. You want to know whether the multi-agent topology binds, whether the per-task model selection actually selects the right tasks, whether the human-in-the-loop gates are positioned where the human can actually approve work fast enough to keep up. You need real workload — real customers, real messages, real PRs, real publishing decisions — to see where the system breaks. Synthetic benchmarks miss the failure mode that broke Hugo v1 entirely, because synthetic benchmarks don't have soft preconditions for the model to fail on.

The cheapest way to get to "is this design real?" is to run the design on the metered tier of whichever vendor's API gives you the model class you need. For the OpenClaw v1 build pipeline, that was Anthropic API metered pricing — 5 worker agents on Opus 4.6, the CEO on Sonnet 4.6, a Haiku-based router. The bill was a material monthly line item. It bought a system that shipped Psinest at the cadence the pilot required. Hiring would have cost more and taken months; the metered Anthropic burn was sustainable for the validation window, even though it wasn't sustainable as the steady state.

The metered phase has a specific job: prove that the workflow shape works. Once that's proven, the economics question becomes answerable, because you have a real workload to compare against. Without the metered phase, the economics question is a guess about a system that doesn't exist yet.

## What "economics" actually means

Economics, in this context, is the migration from the metered shape to the subscription shape. The metered shape scales with throughput — every API call is a line item, the bill is unbounded, the cost-per-unit is premium. The subscription shape is flat per month — bounded regardless of throughput, capped per-unit, predictable. You don't get to start on the subscription shape because the subscription shape doesn't tell you whether your design works. You don't get to *stay* on the metered shape because the metered shape doesn't financially sustain a system that's working.

The migration has to preserve the workflow on top. This is the part that gets missed when teams try to optimise too early — they conflate "we need to spend less" with "we need to change what the system does". You don't change what it does. You change *how it pays* for what it does. The OpenClaw → Hermes Agent migration is the canonical example: same three human-in-the-loop gates, same kanban-board-as-source-of-truth, same agents-propose-humans-dispose discipline. What changed was the substrate — v1's 5 parallel Opus agents on the Anthropic API metered tier became v2's single autonomous dispatcher under flat Claude Max subscription, orchestrating Codex (gpt-5.4-mini, OpenAI subscription) calling Claude Code CLI. Order-of-magnitude cheaper. Same workflow on top.

The Hugo migration is the same shape on a different axis. v2's Pattern C collapse — single agent + 8 area-scoped MCP tool servers — wasn't motivated by cost in the same direct way. It was motivated by the capability constraint that broke v1. But the cost win came along for free: 5–7 inference calls per customer turn dropped to 1–2, with deterministic work moved into Python (where it costs nothing). Two different platform pivots, same architectural sequence: prove the pattern works first, then re-architect once the binding constraint becomes clear.

## The middle phase is real, and it's expensive

The hardest part of the sequence is the middle phase: you have a working system, you know it costs too much, and you don't yet have v2 running. The instinct is to shortcut — kill v1, start v2 immediately, stop the burn. The discipline is to *not* kill v1 until v2 is running at parity. The OpenClaw → Hermes migration kept v1 running on the same VPS, on the same kanban board, on the same `hermes-ready` issue label, until v2 demonstrated the same throughput at the new cost shape. The team-of-roles concurrency that v1 had was given up in v2 — that's a real trade-off — and the way to know you can afford to give it up is by measuring v1 against v2 on the same workload.

This middle phase is where impatient teams pay the highest costs. They either ship v2 too early and lose v1's working state without v2 being ready to replace it, or they delay v2 indefinitely because v1 still works and the migration feels optional. The discipline is to commit to the migration as a known piece of work, time-box it, and accept that the metered burn during the transition is the cost of the pattern. The OpenClaw v1 monthly Anthropic bill became its own problem before v2 shipped — and yes, that was painful — but the alternative (running on subscription tier from day one with an unvalidated topology) leaves more value on the table than the metered-phase burn costs.

## What this isn't

This isn't an argument for spending recklessly during the metered phase. The metered phase has a specific job — validate the workflow — and once that's done, the meter has to come down. It also isn't an argument for delaying cost optimisation indefinitely. The sequence is *velocity first, economics second*, not *velocity only*.

What it is, instead, is an argument for paying attention to the order. You can't optimise a system you haven't validated. You can't validate a system you've designed to cost as little as possible from day one, because cost-driven design forecloses on the topology iteration the validation phase needs. Build the system that works first. Then make it sustainable.

## The fork to look for

If you're designing an AI-augmented system right now and want to know whether you're in the trap, the fork is this: *am I confident enough about the workflow shape to make a long-term cost commitment?*

If yes — you've already validated, you have a working system, you're ready to migrate. The economics phase is in front of you.

If no — you don't yet know whether your design works. The metered phase is in front of you. Ship on the meter, validate, then migrate.

The two phases compose. They don't substitute. Trying to skip the first one is what kills early-stage AI systems; trying to skip the second one is what kills them later. The portfolio has two examples of the full sequence end-to-end; both are documented with the architectural detail and the cost numbers, in [openclaw-hermes-evolution](../openclaw-hermes-evolution/) and [hugo](../hugo/).

The pattern works.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
