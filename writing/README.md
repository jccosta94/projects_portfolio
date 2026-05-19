# Writing — Essays on Agentic Systems & AI Architecture
**Short essays extracted from cross-portfolio lessons. Where the project repos document *what* I built, this is where I document *what I learned* — and how the lessons generalise.**

The five case studies in this portfolio each have a *headline lesson* — the one architectural insight the case study exists to articulate. The essays below pull those lessons out of their case-study context and write them up as standalone pieces, so the patterns can be referenced and reused without needing the full case study as background.

Each essay is 1,400–1,800 words. Each leads with the lesson, walks through the case study that produced it, and ends with the fork-to-look-for that tells you when the lesson applies on the next project.

---

## Essays

### 📄 [Velocity first, economics second](./velocity-first-economics-second.md)
**A sequencing pattern for AI-augmented systems that have to validate at production cadence before they can be optimised.** Drawn from the OpenClaw → Hermes migration that ships [Psinest](../psinest/) and the Hugo v1 → v2 collapse. Why you can't optimise a system you haven't validated, why the metered phase is real and expensive, and why the migration has to preserve the workflow on top.
*~1,500 words.*

### 📄 [Configure a reliable platform, don't build a custom agent from scratch](./configure-dont-build.md)
**The consulting operating model that's worked for Salesforce delivery for two decades applies, almost unchanged, to AI cockpit delivery in 2026.** Drawn from the [Claude Cowork Vertical Workflows](../claude-cowork/) consulting practice. Why the bespoke-build model collapses past the third customer, why the horizontal package is too thin in every vertical, and why the third version of the practice — configure-don't-build, with per-vertical configuration packs on top — is the one that ships.
*~1,800 words.*

### 📄 [Put determinism where the LLM is unreliable](./put-determinism-where-the-llm-is-unreliable.md)
**Brand voice can live in a prompt. Compliance cannot. The library is not smart — it is right.** Drawn from [Skoda AI Ops](../skoda-ai-ops/)'s Python compliance library and [Psinest's](../psinest/) planned PII-redaction proxy. Why prompt-level compliance rules are a 5%-regulatory-exposure system, why a deterministic library is the only path to the publishing surface, and why a second LLM as compliance reviewer makes the failure mode worse, not better.
*~1,400 words.*

---

## Reading order

The three essays compose. If you read them in this order, the patterns build:

1. **Velocity first, economics second** sets up the sequencing discipline — when to build, when to optimise, what the migration looks like.
2. **Configure a reliable platform, don't build a custom agent from scratch** applies the same discipline to a consulting practice rather than a single product — the build-vs-buy decision at the operating-model level.
3. **Put determinism where the LLM is unreliable** is the implementation pattern that makes the systems built under the first two essays survive contact with regulated environments.

Each essay can be read standalone. The full set takes about 30 minutes.

---

## Coming next

Essays that are in the queue but not yet drafted:

- **Three human-in-the-loop gates** — the canonical pattern for AI-augmented systems that ship with externally-visible consequences. Why three, what each gate does, what changes when you go from a 2-agent system to a 7-agent team.
- **Centralised access gates and parallel-tier capability gates** — the two-tier compliance pattern for regulated platforms with AI on top. Why `RoleScope.cs` and the AI Policy Manager are *different concerns* rather than one bigger gate.
- **Skill-naming as API design** — why the description field is the routing logic and what naming discipline looks like for libraries that load skills by description match.
- **The topology collapse** — how Pattern C (single agent + specialists-as-MCP-tools) replaces a multi-agent team when the model class can't bind multi-step delegation. The architectural through-line of Hugo's v1 → v2 migration.

These will land here as they're written.

---

## See also

- **[Shared Architecture Patterns](../shared-architecture-patterns/README.md)** — the catalog of patterns the essays here articulate, with cross-links to the case studies that exercise each pattern.
- **[The case studies themselves](../README.md)** — five live or in-pilot systems that produced the lessons in the essays.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
