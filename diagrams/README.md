# Architecture Diagrams Library
**Catalog index of every inline ASCII architecture diagram across this portfolio.**

Every diagram in this portfolio is **inline text-style ASCII inside a fenced code block** — readable on GitHub, version-controlled, diff-able, no rendering tooling required, no external image dependencies. This page is the index: it groups every diagram in the portfolio by what it shows, with anchor links to where each one lives in its case study. If you're looking for a specific kind of architecture diagram — a multi-agent topology, a deployment view, a security flow, a permission model, an evolution sequence — start here.

---

## Multi-agent topology diagrams

How dispatch authority flows between agents in a hierarchical team, and what each role does.

- **[OpenClaw v1 — the 7-agent team](../openclaw-hermes-evolution/README.md#the-team)** — Telegram → main (Haiku) → CEO (Sonnet 4.6) → PM (Opus 4.6) → Dev + Bugfixing Dev → QA + QA2. All five workers on Opus 4.6. Dispatch-authorization whitelist labeled on each arrow.
- **[Hugo v1 — the 7-agent marketing team](../hugo/README.md#the-team)** — Director → Performance Marketer / Web Developer / Brand Designer / Inbox Manager / Outbound Specialist / Analytics. Six specialists, all on `gpt-5.4-mini`, terminal nodes (no further dispatch).

---

## Single-agent topology diagrams

The shape that replaces a multi-agent team when the model class can't bind multi-step delegation — one agent, multiple area-scoped MCP tool servers.

- **[Hugo v2 — single agent + 8 MCP namespaces](../hugo/README.md#architecture)** — Customer → Hugo (gpt-5.4-mini) → 8 area-scoped MCP servers (`marketing__*` · `creative__*` · `web__*` · `inbox__*` · `outbound__*` · `analytics__*` · `audit__*` · `content_plan__*`). One tool call per turn.
- **[OpenClaw v2 / Hermes-Psinest — single dispatcher + tool chain](../openclaw-hermes-evolution/README.md#architecture)** — Hermes Agent (autonomous dispatcher) → Codex CLI (orchestrator brain, gpt-5.4-mini) → Claude Code CLI (code executor, Claude Max subscription) → workspace. Pattern C applied to a build pipeline.

---

## Enterprise per-pod / per-vertical detail topologies

Layered top-to-bottom enterprise architecture views — every tier, every integration, every security boundary. These are the "everything that's running where" diagrams.

- **[Hugo's per-pod detail](../hugo/README.md#enterprise-topology--per-pod-detail)** — customer-facing layer → network edge → Hugo tenant pod (process supervision · channel adapter · agent runtime · brain/inference · skills library · MCP servers · HITL gate · state+audit · secrets) → external integration layer → admin / data spine → security boundaries. Per-tenant Docker isolation.
- **[Claude Cowork per-vertical detail](../claude-cowork/README.md#enterprise-topology--per-vertical-detail)** — operator's device → Cowork cockpit (skill library · MCP fleet · memory · scheduled tasks) → external integrations per vertical → HITL gate model per vertical → audit + security. Same shape, different per-vertical wiring.

---

## Host topology (system overview) diagrams

The "what runs on which box" overview — clean tree-style boxes with labelled arrows, sized for the landing read.

- **[OpenClaw v1 host topology](../openclaw-hermes-evolution/README.md#the-topology)** — Telegram → Hostinger VPS → systemd / openclaw-gateway → 7-agent runtime → live repos / cron / production stack → external APIs. The whole v1 system in one diagram, with the per-tier detail in a table beneath.
- **[Psinest current-state architecture](../psinest/README.md#current-state-architecture)** — Public users → Hostinger VPS → nginx → React SPA + Kestrel API → PostgreSQL → external services (Firebase Auth · Backblaze B2 · Sentry · DuckDNS · GitHub Actions). Pilot footprint on one VPS.

---

## Security architecture (access gate) diagrams

How tenancy and role-based access get enforced at the request layer.

- **[Psinest's `RoleScope.cs` flow](../psinest/README.md#security-architecture--role-scoped-access-via-rolescopecs)** — Authenticated request (JWT) → ASP.NET middleware → `RoleScope.cs` (reads role, resolves tenant, resolves URL claims, prevents privilege climb) → role-routes (ClinicOwner · Psychologist · Patient) → controller-level guard → EF Core query with scope applied at query construction time. The canonical centralised-access-gate pattern, drawn out end-to-end.

---

## Permission model diagrams

Document-level / per-resource permissions with consent semantics.

- **[Psinest's document permission model](../psinest/README.md#document-permission-model)** — document access request → permission resolver → upload-path vs read/delete-path split → per-role `canDelete` policy (psy-in-patient-space deletes anything, patient deletes own only, ClinicOwner blocked entirely) + read-scope check (patient: own only · psy: in partner clinic AND assigned patient · ClinicOwner: BLOCKED). Healthcare-grade consent-aware permissions in one tree.

---

## Editorial / publishing flow diagrams

Content goes from a draft to a published artefact through a series of approval and compliance gates.

- **[Skoda's editorial pipeline](../skoda-ai-ops/README.md#topology)** — draft (text + media) → publishing library (German-lang check · price-hedge regex · denylist regex · voice rules) → Telegram preview → operator approve/reject → library re-runs (defense in depth) → channel routing (video → YT+IG · photo → IG+X · text → X · LinkedIn opt-in) → Upload-Post API + Cloud Function + Gmail. The Pattern 4 "deterministic library between LLM and public API" pattern, drawn end-to-end.
- **[Skoda's orchestrator flow chart](../skoda-ai-ops/README.md#the-orchestrator)** — how an idea moves from interactive-Cowork or scheduled-Automation to a published post, both lanes feeding into the same publishing library.
- **[Skoda's lead funnel](../skoda-ai-ops/README.md#lead-funnel)** — how a public visitor becomes a tracked CRM lead. GA4 + Firestore `quote_events` + Airtable + dealer hand-off.
- **[Skoda's attribution split](../skoda-ai-ops/README.md#attribution-split)** — why GA4 (for Google Ads bid optimisation) and Firestore (for commission accounting) deliberately disagree, and which optimiser consumes which copy.

---

## Issue lifecycle / human-in-the-loop diagrams

How a piece of work moves through the system, with the three human-in-the-loop gates explicitly marked.

- **[OpenClaw issue lifecycle — kanban → PR → deploy (three human gates)](../openclaw-hermes-evolution/README.md#issue-lifecycle--kanban--pr--deploy-three-human-gates)** — kanban board → CEO scan → priority proposal → **Joao approves (gate 1)** → PM scopes → Dev/Bugfix worker writes → QA runs e2e → PR opens → **Joao reviews + merges (gate 2)** → **Joao triggers workflow_dispatch (gate 3)** → deploy. The canonical three-gates pattern, drawn end-to-end.

---

## Evolution / migration diagrams

Side-by-side or before/after views showing the architectural pivot from one generation to the next.

- **[Psinest current-state → AI-augmented future state](../psinest/README.md#ai-augmentation-roadmap-future-state)** — current data plane (application platform + RoleScope + PostgreSQL) → future state (same + 8 AI capability domains + AI Policy Manager + pgvector + job queue + redaction proxy + audit ledger → external AI services). The Pattern 2 "parallel-tier capability gate" pattern, drawn as a current-vs-future split.

---

## Deployment view diagrams

C4-container-style "where the boxes physically run" views.

- **[Skoda's deployment view](../skoda-ai-ops/README.md#deployment-view)** — operator's Mac (Cowork + MCP servers + publishing library) → Firebase (Firestore + Hosting + Cloud Functions) + Upload-Post.com + Telegram Bot API + optional headless VPS (the warm-but-not-running alternative). Single-operator content-ops architecture.

---

## Diagram conventions (the style guide for any new diagram in this portfolio)

These conventions are applied across every diagram above. Use them when adding a new one.

**Container syntax.**
- Unicode box-drawing characters preferred: `┌─┐ │ └─┘`
- Plain ASCII fallback when Unicode would be ambiguous: `+---+ | +---+`
- Wide nodes with breathing room around the label; don't cram details inside the box

**Arrows.**
- Primary flow: `│` and `▼` for vertical, `→` for horizontal
- Plain ASCII fallback: `--->`
- Connector labels sit *on* the arrow line, not floating in whitespace
- Right-side annotations for callouts: `← Sonnet 4.6` or `← see Pattern C above`

**Layout.**
- Top-to-bottom, persona → experience → application → integration → data — same layered convention used in the Enterprise per-pod / per-vertical diagrams
- One branching point per diagram maximum (keeps the eye-path simple)
- Whitespace generously — sparse beats dense for readability on GitHub

**Detail placement.**
- The diagram shows the *shape*; tables beneath it show the *detail*. Don't cram details inside boxes.
- Per-tier tables for "what's running where"; per-row tables for "what each thing is and what it does".

**Fenced code blocks.**
- Every diagram lives inside a ` ```text ... ``` ` fence so GitHub renders it as monospace and indentation is preserved.
- Never use indent-based code blocks for diagrams — they break on GitHub when the indentation isn't consistent across the whole block.

**What we don't do.**
- No PNGs, SVGs, JPGs, or other raster/vector image files for architecture diagrams. (Product UI screenshots are fine — see [Psinest's screenshots](../psinest/README.md#screenshots) — but architecture diagrams stay text-style ASCII.)
- No Excalidraw / drawio / Mermaid source files. The portfolio used these in an earlier version; everything was migrated to inline ASCII to keep the diagrams diff-able, version-controlled, and dependency-free.
- No "open this URL to view the diagram" external links. If the diagram is part of the case study, it lives in the case study.

---

## Why text-style ASCII

The case for inline ASCII over rendered diagrams, in this specific context:

- **Diff-able.** Every change to a diagram shows up cleanly in `git diff`. Renamed boxes are visible in the diff. Re-flowed arrows are visible in the diff. Reviewers can read the diagram change without leaving the PR.
- **Version-controlled.** The diagram is the source of truth, not a generated artefact from a source elsewhere. There's no "this diagram is out of date because the source `.excalidraw` was edited but not re-exported" failure mode.
- **Rendering-free.** GitHub renders fenced code blocks as monospace. That's the entire toolchain. No build step, no plugin, no extension, no external service.
- **Portable.** The same diagram renders in GitHub, VS Code, JetBrains, vim, Cursor, Claude Cowork, Claude Code, terminal `cat`, `less`, `bat`, and any markdown viewer that handles fenced code blocks. Which is all of them.
- **Honest about the level of detail.** ASCII forces simplification — there's an upper bound on how much you can cram into a single diagram before it stops being legible. That upper bound is approximately the right amount of detail for a portfolio diagram. The "infinitely zoomable Excalidraw canvas" temptation is to keep adding detail until the diagram is unreadable; ASCII pushes back.

**The trade-off.** Pixel-perfect visual styling isn't possible. Coloured zones, gradient fills, icon badges, custom fonts — none of that is in scope. The portfolio explicitly trades visual flair for legibility, diff-ability, and rendering-independence. The result is more text-heavy than a typical architecture deck, and that's by design.

---

## See also

- **[Shared Architecture Patterns](../shared-architecture-patterns/README.md)** — the patterns that the diagrams above visualise. Each pattern references the diagrams that illustrate it.
- **[Each case study's `## Architecture diagrams` section](../psinest/README.md#architecture-diagrams)** — a per-case-study anchor index pointing to the inline diagrams within that case study.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
