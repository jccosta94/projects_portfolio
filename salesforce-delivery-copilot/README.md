# Salesforce Delivery Copilot
**An AI copilot for Salesforce delivery work. Same configure-don't-build operating model the Salesforce ecosystem has lived by for two decades, now applied to the copilot that does the delivery.**

This is a case study in **bringing the Salesforce-consulting operating model across to AI consulting** — specifically: how I take a single Salesforce delivery conversation (a discovery call transcript, a build-review meeting, a steering-committee readout) and fan it out across the deliverable surface — process flows, draft user stories, project-plan updates, decision-log entries, action items, statement-of-work language — all cross-referenced through a single ID space so the artifacts stay queryable instead of dying in folders.

The copilot is live on a Salesforce delivery engagement at a DACH pharma-adjacent client (Service Cloud + Experience Cloud, ~9-month rollout, hybrid in-house + partner team). The constraint that triggered the architecture work: the consultant role on a Salesforce delivery doesn't bottleneck on writing artifacts — it bottlenecks on keeping the artifacts coherent with each other as the project evolves. Notes from Tuesday's call shouldn't reference user stories that Wednesday's plan rewrite already split. The first version I built — an n8n multi-agent system with a Telegram cockpit — produced individually-fine artifacts that drifted out of sync the moment the plan changed. The architecture had to be rebuilt around the cross-reference graph, not around the artifact-generation pipeline.

### What I architected

**v1 — an n8n multi-agent system with a Telegram cockpit.** The original design (April 2026) was a MainAgent orchestrator on n8n routing requests to three domain sub-agents — `ServiceDeliveryAgent`, `ProjectAgent`, `CommercialAgent` — each holding its own template library (~25 artifacts across the three) and running a five-step generation pipeline (fetch → template fill → critic pass → revise → return). Context grounded against Jira, Confluence, Google Drive, and Gmail through MCP tools. The consultant interacted via Telegram: type a request, MainAgent classified intent, the right sub-agent ran the pipeline, the draft came back for approve / edit / cancel. **It produced fine artifacts and a coherence catastrophe.** When the project plan got rewritten Tuesday and a user-story draft request landed Wednesday, the sub-agent generated against the *old* plan because nothing in the architecture made it look at the plan first. The fan-out was an afterthought; the cross-reference was a hope.

**v2 — a single Salesforce-delivery copilot configured on Claude Cowork.** The collapse: 3 agents + 25 templates + n8n routing → one Cowork install with a configured skill library, an MCP fleet wired to the project's real systems, and a cross-reference graph as the load-bearing data store. The consultant's cockpit is Cowork desktop — same app I use for the [Cowork vertical packages](../claude-cowork/). The skills are named for the fan-out steps (`transcript-to-notes`, `notes-to-user-stories`, `user-stories-to-process-flow`, `project-plan-update`, `decision-log-entry`, plus six more — see [Skill library](#skill-library) below). The MCP fleet reads from the transcript source (Granola), Jira, Confluence, Google Drive, and Salesforce metadata; it writes back to Jira, Confluence, and the project's Airtable cross-reference base. The HITL floor is universal — every artifact reviewed by the consultant before it lands in any system of record. **What changed in v2 is not the artifacts. What changed is that every artifact is born with an ID and a cross-reference set, and the skill library refuses to draft a new artifact without first reading the current state of the artifacts it would link to.**

### The architectural insight

**Configure the copilot the same way you configure Salesforce.** The Salesforce consulting operating model — twenty years of it — is "configure the platform, don't build from scratch." Salesforce consultants don't write their own CRM; they configure Salesforce. The corresponding move for AI delivery: don't write a bespoke Salesforce-aware agent per engagement. Configure [Claude Cowork](https://claude.com/cowork) — the cockpit Anthropic has invested in making reliable — with the right skill library, the right MCP fleet, and the right HITL gates. The [Cowork vertical-packages case study](../claude-cowork/) is where I first articulated this pattern; this case study is that argument applied to the platform I came from. The Salesforce Delivery Copilot is the bridge between my Salesforce-delivery past and my AI-consulting present, and the bridge holds because the operating model is the same on both sides.

**The fan-out is the architectural unit. The artifact is not.** v1's mistake was treating "produce a user story" as the unit of work. v2 treats "fan out a single delivery conversation across the deliverable surface" as the unit. One Granola transcript becomes a coherent set of five-or-six cross-referenced artifacts in a single pass: notes (NOTE-XXX) → process flow (PF-XXX) → draft user stories (US-XXX) → project-plan deltas (PP-XXX) → decision-log entries (DEC-XXX) → action items linked to all of the above. Each artifact is born holding pointers to its siblings and its parents in the cross-reference graph. The skill library is structured around the fan-out shape, not around the individual artifact types — `transcript-to-fanout` is the entry point, and the individual `*-to-*` skills are sub-stages that run in sequence and write back to the same cross-reference base atomically.

**The cross-reference graph is the load-bearing piece.** Notes are easy. User stories are easy. Process diagrams are easy. The thing that's hard — and the thing that's worth doing — is making sure that on Friday, when the steering committee asks "why did we split this user story?", the system can walk: action-item AI-104 → decision DEC-027 → notes NOTE-042 → transcript T-2026-05-11 → user-story split US-088 → US-088a + US-088b → project-plan revision PP-014. **Cross-referenced artifacts are queryable. Isolated artifacts are file-graveyard.** The architecture-defining choice in v2 is that no skill can write any artifact without first establishing its cross-reference set in the project's Airtable graph base. The graph is the spine; the artifacts hang off it.

**Three insights, layered. The configure-don't-build lineage is the inherited operating model; the fan-out is the work unit that makes the copilot useful on a delivery; the cross-reference graph is the data structure that keeps the fan-out coherent across weeks and rewrites.** Drop any one and the architecture stops working.

---

## TL;DR

|  | **v1 — n8n multi-agent + Telegram** | **v2 — Cowork + skill library + cross-reference graph** |
| --- | --- | --- |
| **Solves which problem** | **Coverage** — generate every artifact type a Salesforce delivery needs | **Coherence** — keep every artifact cross-referenced and current as the project evolves |
| **Topology** | MainAgent + 3 sub-agents + 25 templates routed via n8n | 1 Cowork install + 11 skills + cross-reference graph in Airtable |
| **Consultant cockpit** | Telegram bot | Claude Cowork desktop |
| **Input shape** | Free-text Telegram request, optionally pasted transcript | Granola meeting transcript (auto-ingested) + ad-hoc Cowork prompts |
| **Output shape** | One artifact at a time, drafted to a Drive folder | Coherent fan-out: notes + process flow + user stories + plan delta + decision log + action items, all cross-referenced |
| **Cross-reference model** | None enforced — links by human convention | Single ID space (NOTE-XXX, PF-XXX, US-XXX, PP-XXX, DEC-XXX, AI-XXX) in Airtable graph base; every skill reads + writes back atomically |
| **MCP fleet** | Jira · Confluence · Drive · Gmail (read-only, via custom n8n nodes) | Granola · Jira · Confluence · Google Drive · Salesforce metadata · Airtable graph base · Gmail draft |
| **Critic pass** | Inline, two-stage LLM pass per artifact | Per-skill `# Don'ts / escalation` block + cross-reference integrity check before any write |
| **HITL gate model** | Approve / edit / cancel per draft via Telegram | Drafts-only floor everywhere; consultant reviews every artifact in Cowork before write-back to Jira / Confluence / plan |
| **Inference model** | One call per agent stage (router + sub-agent + critic), per request | Sonnet 4.6 for routine skills; Opus 4.6 for the heavier reasoning skills (`transcript-to-fanout`, `user-stories-to-process-flow`) |
| **Operating state today** | Designed and prototyped; never deployed to a live engagement | **LIVE** — running on a Salesforce delivery engagement at a DACH pharma-adjacent client |
| **Reversibility** | N/A — abandoned before deployment | The n8n multi-agent design is archived as a reference for engagements where Cowork isn't the right cockpit (e.g. fully air-gapped clients) |

**Headline insight: the copilot doesn't compete with the Salesforce consultant. It removes the cross-artifact coherence work the consultant was doing in their head and made into a queryable graph.**

---

## Why this matters

Most consultancies trying to bring AI to Salesforce delivery in 2026 hit two walls in sequence: **first the artifact wall, then the coherence wall.**

**Wall 1 — Artifact generation.** A Salesforce delivery produces a lot of documents: discovery notes, process flows, user stories, project plans, statement-of-work language, change requests, status reports, steering decks, UAT scripts, training material. The temptation, when you have an LLM on the team, is to point it at each artifact type and accept whatever comes out. **This is what v1 did.** It works for individual documents and breaks for project-shaped work. The failure mode is silent: every artifact is fine in isolation, the consultant signs off on each draft, and a month later the project plan references user stories that don't exist anymore because Tuesday's plan rewrite split them and nobody back-propagated the change to last week's process flow.

**Wall 2 — Coherence.** Once you accept that individual-artifact generation isn't the unit of work, the obvious move is to make the artifacts link to each other. **But "link" is the wrong primitive.** Links are author-side conventions; nothing stops a future draft from rendering a link that points to a now-stale artifact. The coherence the project actually needs is structural: every artifact has an ID, every artifact is born holding pointers to the artifacts it depends on, every revision invalidates the right downstream artifacts, and the consultant can walk the graph forward (what did this decision produce?) or backward (where did this user-story split come from?) without re-reading three months of Slack. **This is where the cross-reference graph comes in:** the graph is the system of record for *how the artifacts relate to each other*; the artifacts themselves can live wherever the project tools want them (Confluence, Jira, Drive, Salesforce).

The lesson is **the sequence.** Ship v1 first to prove that artifact generation alone isn't the problem — the consultant doesn't lack capacity to draft, they lack capacity to keep drafts coherent. Then design v2 around the coherence machinery, not around adding more artifact types. **Doing them in the wrong order — building the cross-reference graph first, without v1 evidence that artifact generation is the easy part — leads to over-engineering. Sticking with v1 once the coherence failures pile up leads to a project where the AI's outputs are mistrusted on principle, because the consultant has been burned by stale links three times.**

The lesson is also **what stays constant.** Across both v1 and v2, the consultant is still the human who reviews every artifact before it lands in any system of record. The model writes; the consultant disposes. **That floor doesn't move regardless of how good the model gets**, because on a Salesforce delivery the consultant is the one accountable to the client — and accountability without authorship is a tenuous position. The architecture exists to give the consultant leverage, not to remove them from the loop.

---

## v1 — n8n multi-agent system + Telegram cockpit

The original design (April 2026) was an n8n-orchestrated multi-agent system modelled after a delivery team's role distribution. MainAgent acts as router; three domain sub-agents own their template libraries; the consultant interacts via Telegram and approves drafts one at a time.

### The team

```text
                Consultant · Telegram (per-engagement bot)
                              │
                              ▼
                ┌───────────────────────────┐
                │  MainAgent (n8n)          │   ← intent classifier
                │  Claude Sonnet 4.6        │     routing + gates
                └─────────────┬─────────────┘
                              │ dispatches to sub-agent
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ Service        │  │ Project        │  │ Commercial     │
│  Delivery      │  │  Agent         │  │  Agent         │
│  Agent         │  │                │  │                │
│ Sonnet 4.6     │  │ Sonnet 4.6     │  │ Opus 4.6       │
│ 9 templates    │  │ 8 templates    │  │ 8 templates    │
└────────┬───────┘  └────────┬───────┘  └────────┬───────┘
         │                   │                   │
         └───────────────────┴───────────────────┘
                             │ five-step pipeline per artifact
                             ▼
                ┌───────────────────────────┐
                │  fetch → fill → critic    │
                │  → revise → return        │
                │  context: Jira ·          │
                │  Confluence · Drive ·     │
                │  Gmail (read-only)        │
                └───────────────────────────┘
```

### Sub-agent roster

| Sub-agent | Domain | Template count (v1) | Returns |
| --- | --- | --- | --- |
| **ServiceDeliveryAgent** | Operational artifacts — service reports, RAID logs, status digests, change requests, incident summaries | 9 | Single artifact to a designated Drive folder |
| **ProjectAgent** | Delivery-lifecycle artifacts — project plans, user stories, test scaffolds, release notes, sprint summaries, retros | 8 | Single artifact to a designated Drive folder |
| **CommercialAgent** | Contract / commercial artifacts — statements of work, steering decks, proposals, T&M reports, scope-change memos | 8 | Single artifact to a designated Drive folder |

All sub-agents are terminal — they don't dispatch further. Two levels deep (MainAgent → sub-agent), one artifact per request, every request explicitly approved by the consultant before write-back.

### The consultant flow (designed)

A typical artifact workflow under v1:

```text
1. Consultant on Telegram:  "draft a user story for the lead-routing rule we agreed in today's call"
2. MainAgent classifies: intent = "user story draft" → ProjectAgent
3. ProjectAgent runs the five-step pipeline:
   ├─ fetch        → Jira epic + last week's Confluence call notes + relevant Drive prior stories
   ├─ template fill → Claude drafts the user story to the agent's user-story template
   ├─ critic pass  → second Claude call reviews the draft against the rubric
   ├─ revise       → applies critic edits, produces v2
   └─ return       → posts the draft back to Telegram with a summary
4. Consultant: approve / edit / cancel
5. On approve → ProjectAgent writes the artifact to the designated Drive folder
```

The design assumed that consultant-driven approval per artifact was sufficient for project coherence. It wasn't.

### How v1 broke

Three failure modes, all consistent, all the same shape — none of them were artifact-quality issues:

**Coherence drift across rewrites.** The consultant rewrote the project plan on a Tuesday based on a steering decision. Wednesday, they asked v1 to draft a status report. The status report referenced the previous plan's milestone names because nothing in v1's architecture made it re-read the plan before drafting a report that summarized progress against it. The consultant caught it on review, but only because they had personally rewritten the plan the day before. **For a delivery with a team of four consultants, each rewriting different parts of the plan asynchronously, this failure mode would not have been catchable by review.**

**No back-propagation on splits.** A user story (US-088 in the consultant's manual naming convention) got split into US-088a and US-088b after a refinement session. The previous week's process flow diagram still rendered the path through US-088. v1 had no concept that the process flow depended on the user story; the link existed only in the consultant's head.

**Statement-of-work language drifting from active scope.** The CommercialAgent drafted a scope-change memo against the project's original SOW. The original SOW had already been amended twice by previous scope-change memos. The new memo's language conflicted with the prior amendments. **Each individual artifact was internally coherent. The artifact set as a whole was not.**

Two responses:

- **Fix the artifact-generation pipeline.** Add cross-reference fetches to each sub-agent's pipeline, hope the consultant remembers to provide the right context. Brittle and depends on the consultant doing the coherence work the architecture was supposed to help with.
- **Fix the topology.** Make the cross-reference graph a first-class data structure, configure a single cockpit that reads and writes it atomically, and re-author the skill library around the fan-out shape (one transcript → coherent set of artifacts) rather than around individual artifact types.

Fixing the topology was the longer path on paper but the shorter path in practice — because it eliminated the failure surface instead of accreting around it. **That became v2.**

---

## v2 — Cowork + skill library + cross-reference graph

The collapse: 3 sub-agents + 25 templates + n8n routing → 1 Cowork install + 11 skills + a single cross-reference graph in Airtable. The consultant's cockpit is Claude Cowork desktop (same app I use for the [Cowork vertical packages](../claude-cowork/)). The artifacts didn't change; the architecture around them did.

### Architecture

```text
                       Consultant · Cowork desktop
                       (single window, single context)
                                  │
                                  │ chat-driven interactive
                                  │ + Granola auto-ingest
                                  ▼
                       +-----------------------------------+
                       | CLAUDE COWORK                     |
                       |   Anthropic Claude                |   <-- consultant-facing
                       |   Sonnet 4.6 routine              |       single cockpit
                       |   Opus 4.6 heavy reasoning        |       computer-use enabled
                       |   skill library auto-load         |       file-system access
                       |   memory dir loaded               |       MCP fleet wired
                       +---+--------------------------+----+
                           |                          |
                  interactive lane         scheduled-tasks lane
                           v                          v
                +---------------------+   +-----------------------------+
                | SKILL LIBRARY       |   | MCP FLEET                   |
                |   11 skills         |   |   read + write per scope    |
                |   transcript-to-*   |   |   Granola · Jira ·          |
                |   notes-to-*        |   |   Confluence · GDrive ·     |
                |   user-stories-to-* |   |   SF metadata · Airtable    |
                |   project-plan-     |   |   graph base · Gmail draft  |
                |     update          |   |                             |
                |   decision-log-     |   |                             |
                |     entry           |   |                             |
                |   action-item-      |   |                             |
                |     extract         |   |                             |
                |   (5 more · see     |   |                             |
                |    table below)     |   |                             |
                +---------+-----------+   +---------------+-------------+
                          |                               |
                          | every skill writes back atomically
                          | to the cross-reference graph
                          v                               v
                +---------------------------------------------------+
                | CROSS-REFERENCE GRAPH (Airtable base, project-owned)|
                |   tables:                                          |
                |     Transcripts  T-YYYY-MM-DD                      |
                |     Notes        NOTE-XXX  (linked: T-, PF-, DEC-) |
                |     ProcessFlows PF-XXX    (linked: NOTE-, US-)    |
                |     UserStories  US-XXX    (linked: PF-, PP-, AI-) |
                |     PlanDeltas   PP-XXX    (linked: DEC-, US-)     |
                |     Decisions    DEC-XXX   (linked: T-, NOTE-)     |
                |     ActionItems  AI-XXX    (linked: DEC-, US-, PP-)|
                |   every row holds a status + a back-ref set        |
                |   integrity check runs on every skill write        |
                +---------------------------------------------------+

----------------- inside the consultant's workstation -----------------

+----------------------------------+  +----------------------------------+
| HITL GATE (universal floor)      |  | MEMORY DIRECTORY                 |
+----------------------------------+  +----------------------------------+
|   drafts-only everywhere         |  |   memory/                        |
|   write-back to Jira /           |  |     engagement_profile.md        |
|     Confluence / SF only after   |  |     client_voice.md              |
|     consultant approval in       |  |     stakeholder_map.md           |
|     Cowork                       |  |     rules.md (SOW guardrails,    |
|   Granola auto-ingest produces   |  |       compliance posture)        |
|     a transcript row + a         |  |     id_space_rules.md            |
|     fan-out draft set; nothing   |  |     (the naming + linking rules) |
|     lands in any system of       |  |                                  |
|     record until consultant      |  |   memory updates are explicit -  |
|     reviews                      |  |     skill writes, never silent   |
+----------------------------------+  +----------------------------------+
```

### Skill library

11 skills organised around the fan-out shape:

| Skill | Layer | What it does |
| --- | --- | --- |
| `transcript-to-fanout` | entry | Reads a Granola transcript, runs the full fan-out pass — produces notes, candidate process-flow updates, candidate user-stories, plan deltas, decision-log entries, action items, all cross-referenced. Opus 4.6. |
| `transcript-to-notes` | sub-stage | Standalone — converts a transcript to NOTE-XXX entries in Confluence, with cross-refs back to T-YYYY-MM-DD and forward to whatever DEC-XXX entries the notes implicate |
| `notes-to-user-stories` | sub-stage | Drafts US-XXX entries from a NOTE-XXX, populates acceptance criteria, links back to PF-XXX where the story participates in a process flow |
| `user-stories-to-process-flow` | sub-stage | Generates or revises a PF-XXX process-flow diagram (Mermaid-in-Confluence) from a set of US-XXX stories; flags stories whose flow position is ambiguous |
| `project-plan-update` | sub-stage | Drafts a PP-XXX plan delta from a DEC-XXX or a set of US-XXX; surfaces downstream artifacts (process flows, status reports) that the delta invalidates |
| `decision-log-entry` | sub-stage | Captures a DEC-XXX from a transcript or an ad-hoc consultant prompt; enforces the decision template (context · alternatives · decision · rationale · consequences) |
| `action-item-extract` | sub-stage | Pulls AI-XXX action items from a transcript, links them to DEC-XXX / US-XXX / PP-XXX they depend on, drafts a Jira task for each |
| `status-report-weekly` | composite | Friday-cadence skill — reads the cross-reference graph for the week's NOTE-XXX, DEC-XXX, PP-XXX, AI-XXX deltas; drafts a status report grounded in the actual graph state, not in the consultant's memory |
| `sow-language-check` | safety | Before any CommercialAgent-style artifact (scope memo, change order, SOW amendment) is drafted, this skill reads the active SOW + all prior amendments and flags scope language that conflicts |
| `coherence-audit` | safety | Daily 07:00 scheduled task — walks the cross-reference graph, flags artifacts with stale back-refs (parent revised, child not updated), surfaces a triage list for the consultant |
| `cross-ref-integrity-check` | infra | Internal — runs before any skill writes to the graph base; refuses the write if the cross-ref set is incomplete or points to a non-existent ID |

### HITL gates

The HITL floor is universal — every artifact reviewed by the consultant before write-back to any system of record. On top of that floor, three structural gates:

- **Drafts-only floor.** No skill writes directly to Jira, Confluence, Salesforce metadata, or the project plan without consultant approval through Cowork's chat. The consultant taps to commit; the agent never sends. Same discipline as the [Cowork vertical packages](../claude-cowork/) — `# Don'ts / escalation` block in every `SKILL.md`.
- **Cross-reference integrity check is non-negotiable.** Before any skill writes to the Airtable graph base, `cross-ref-integrity-check` runs. If the cross-ref set is incomplete, points to a non-existent ID, or would create a cycle, the write is refused and surfaced to the consultant. This is code-enforced inside the MCP tool wrappers, not relied on as a prompt instruction.
- **SOW-language check on commercial-class artifacts.** Any artifact that touches commercial scope — change memos, scope amendments, SOW addenda — runs through `sow-language-check` before the consultant sees the draft. If the draft conflicts with active SOW language or prior amendments, the conflict is flagged in the draft, the consultant sees both versions side-by-side, and the consultant decides.

### Integration surface

What the copilot reads from and writes to:

| System | Mode | What for |
| --- | --- | --- |
| **[Granola](https://granola.ai)** | Read (transcripts auto-ingested) | Meeting transcripts are the primary input. Granola's local-first model and its export-to-Drive integration makes the ingest path clean — transcripts land in a designated Drive folder, the copilot picks them up |
| **Jira** | Read + draft (consultant taps to commit) | Issues, epics, sprints, worklogs read for context. AI-XXX action items drafted as issues; user-story drafts proposed as new issues with full cross-ref metadata in the description |
| **Confluence** | Read + draft (consultant taps to commit) | Specs, ADRs, meeting notes read for context. NOTE-XXX, PF-XXX, DEC-XXX artifacts drafted as Confluence pages with the ID in the page title and a metadata block linking to siblings |
| **Google Drive** | Read | Granola transcript landing zone. Prior-engagement artifacts for reference. Read-only — no write-back |
| **Salesforce metadata API** | Read | Object model, field metadata, validation rules, automation flows — read for grounding user-story drafts and process-flow generation against actual SF configuration |
| **Airtable (cross-reference graph base)** | Read + write (atomic per skill) | The graph is the system of record for how artifacts relate. Every skill writes back to the graph atomically — write succeeds only if the cross-ref integrity check passes |
| **Gmail** | Draft only | Stakeholder communication drafts (status updates, escalations) land as Gmail drafts the consultant taps to send. Never autonomous send |

The discipline rule: **Cowork is the cockpit; Airtable is the graph; everything else is a system of record the consultant taps to write to.** The graph is the only place where the copilot owns the data.

---

## Enterprise topology · per-engagement detail

The full per-engagement reference architecture — every tier, every integration, every boundary. The v2 diagram above shows the *shape*; this one shows what's actually running where, layer by layer.

```text
==============================================================================
CONSULTANT SURFACE
==============================================================================

+----------------------------------+  +----------------------------------+
| COWORK DESKTOP                   |  | GRANOLA AUTO-INGEST              |
+----------------------------------+  +----------------------------------+
|   Anthropic Claude               |  |   Granola records the meeting    |
|     Sonnet 4.6 for routine work  |  |     locally + transcribes        |
|     Opus 4.6 for transcript-to-  |  |   Export to Drive folder         |
|       fanout and process-flow    |  |     designated for engagement    |
|       generation                 |  |   Cowork scheduled task picks    |
|   single window, single context  |  |     up new transcripts hourly    |
|   computer-use enabled           |  |     during work hours            |
|   file system access             |  |   Consultant can also invoke     |
|   skill library auto-load        |  |     transcript-to-fanout         |
|   memory directory loaded        |  |     manually on any transcript   |
|   MCP fleet wired                |  |     in the Drive folder          |
+----------------------------------+  +----------------------------------+
                                |
                                v
==============================================================================
COWORK ORCHESTRATION
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| INTERACTIVE LANE     | | SCHEDULED LANE       | | MEMORY DIRECTORY     |
+----------------------+ +----------------------+ +----------------------+
|   consultant at the  | |   Cowork-native      | |   memory/            |
|   keyboard           | |     scheduled tasks  | |     engagement_      |
|   chat-driven        | |   same Claude,       | |       profile.md     |
|     invocation       | |     same skills,     | |     client_voice.md  |
|   skill picker       | |     same MCPs,       | |     stakeholder_     |
|     by description   | |     same memory      | |       map.md         |
|   ad-hoc workflows   | |                      | |     rules.md         |
|                      | |   hourly:            | |       (SOW           |
|                      | |     Granola Drive    | |        guardrails,   |
|                      | |     poll for new     | |        compliance,   |
|                      | |     transcripts      | |        SF org        |
|                      | |   07:00 daily:       | |        constraints)  |
|                      | |     coherence-audit  | |     id_space_        |
|                      | |     scans graph for  | |       rules.md       |
|                      | |     stale back-refs  | |       (naming +      |
|                      | |   Friday 16:00:      | |        linking       |
|                      | |     status-report-   | |        conventions)  |
|                      | |     weekly drafts    | |     templates/       |
|                      | |     for consultant   | |       (per-artifact  |
|                      | |     review           | |        templates)    |
|                      | |                      | |                      |
|                      | |                      | |   memory updates     |
|                      | |                      | |     are explicit -   |
|                      | |                      | |     skill writes,    |
|                      | |                      | |     never silent     |
+----------------------+ +----------------------+ +----------------------+

+----------------------+ +----------------------+ +----------------------+
| SKILL LIBRARY        | | HITL GATE MODEL      | | AUDIT TRAIL          |
+----------------------+ +----------------------+ +----------------------+
|   .claude/skills/    | |   per-skill          | |   every drafted      |
|   11 skills:         | |     # Don'ts /       | |     artifact lands   |
|                      | |     escalation       | |     on disk under    |
|   entry:             | |     block            | |     drafts/          |
|     transcript-to-   | |                      | |       transcripts/   |
|       fanout (Opus)  | |   universal floor:   | |       notes/         |
|   sub-stages:        | |     drafts-only      | |       user-stories/  |
|     transcript-to-   | |     everywhere       | |       process-flows/ |
|       notes          | |   structural gates:  | |       plan-deltas/   |
|     notes-to-user-   | |     cross-ref        | |       decisions/     |
|       stories        | |       integrity      | |       action-items/  |
|     user-stories-to- | |       check          | |                      |
|       process-flow   | |     SOW-language     | |   every write to the |
|       (Opus)         | |       check          | |     graph base is    |
|     project-plan-    | |                      | |     timestamped +    |
|       update         | |   nothing fires      | |     consultant-      |
|     decision-log-    | |     without          | |     attributed       |
|       entry          | |     consultant tap   | |                      |
|     action-item-     | |                      | |   Cowork conversation|
|       extract        | |                      | |     log is the audit |
|   composites:        | |                      | |     of every action  |
|     status-report-   | |                      | |                      |
|       weekly         | |                      | |                      |
|   safety:            | |                      | |                      |
|     sow-language-    | |                      | |                      |
|       check          | |                      | |                      |
|     coherence-audit  | |                      | |                      |
|   infra:             | |                      | |                      |
|     cross-ref-       | |                      | |                      |
|       integrity-     | |                      | |                      |
|       check          | |                      | |                      |
|                      | |                      | |                      |
|   each SKILL.md:     | |                      | |                      |
|     name + desc      | |                      | |                      |
|     # Purpose        | |                      | |                      |
|     # When to use    | |                      | |                      |
|     # Inputs         | |                      | |                      |
|     # Process        | |                      | |                      |
|     # Output         | |                      | |                      |
|     # Don'ts /       | |                      | |                      |
|       escalation     | |                      | |                      |
+----------------------+ +----------------------+ +----------------------+
                                |
                                v
==============================================================================
MCP FLEET · per-engagement wired
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| READ-ONLY MCPs       | | READ + DRAFT MCPs    | | GRAPH MCP            |
+----------------------+ +----------------------+ +----------------------+
|   Granola Drive      | |   Jira MCP           | |   Airtable MCP       |
|     (transcript      | |     (issues read +   | |     (cross-reference |
|     export folder)   | |      draft-create    | |     graph base,      |
|                      | |      with cross-ref  | |     project-owned,   |
|   Salesforce metadata| |      in description) | |     read + atomic    |
|     read MCP         | |   Confluence MCP     | |     write per skill) |
|     (object model,   | |     (pages read +    | |                      |
|     field metadata,  | |      draft with ID   | |   integrity check    |
|     validation       | |      in title +      | |     middleware:      |
|     rules, flows)    | |      cross-ref       | |     refuses writes   |
|                      | |      metadata block) | |     that fail the    |
|   Google Drive       | |   Gmail MCP          | |     ID-space rules   |
|     (prior-engage-   | |     (draft only,     | |                      |
|     ment artifacts   | |      never send)     | |                      |
|     for reference)   | |                      | |                      |
+----------------------+ +----------------------+ +----------------------+
                                |
                       outbound API calls per
                       MCP tool invocation
                                v
==============================================================================
SYSTEMS OF RECORD · client-owned
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| SALESFORCE ORG       | | PROJECT TOOLS        | | DOCUMENT TOOLS       |
+----------------------+ +----------------------+ +----------------------+
|   client's org       | |   Jira               | |   Confluence         |
|   Service Cloud +    | |     project-side     | |     project space    |
|     Experience Cloud | |     project board    | |     specs, ADRs,     |
|                      | |   subscription on    | |     meeting notes,   |
|   read access via    | |     client's license | |     status reports   |
|     metadata API     | |                      | |   subscription on    |
|     scoped to        | |   the copilot drafts | |     client's license |
|     read-only        | |     issues; the      | |                      |
|     metadata roles   | |     consultant taps  | |   the copilot drafts |
|                      | |     to commit        | |     pages; the       |
|   the copilot never  | |                      | |     consultant taps  |
|     writes to the    | |                      | |     to commit        |
|     org              | |                      | |                      |
+----------------------+ +----------------------+ +----------------------+

==============================================================================
DATA RESIDENCY + COMPLIANCE
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| CONSULTANT'S DEVICE  | | ANTHROPIC API        | | AIRTABLE             |
+----------------------+ +----------------------+ +----------------------+
|   Cowork desktop     | |   Anthropic Claude   | |   EU workspace       |
|   memory dir +       | |     (Sonnet 4.6,     | |     (Frankfurt) for  |
|     skill library    | |      Opus 4.6) via   | |     DACH engagements |
|   drafts/ folder     | |     Cowork's cockpit | |   project-owned base |
|     under engagement | |   consultant's       | |     (the consultancy |
|     project folder   | |     subscription     | |     does not retain  |
|   .env (chmod 600)   | |   data not used for  | |     graph data at    |
|     never committed  | |     training         | |     engagement end - |
|                      | |     (Claude Cowork   | |     base transfers   |
|                      | |      privacy policy) | |     to client)       |
+----------------------+ +----------------------+ +----------------------+
```

**Three things to highlight from this layout:**

**The graph is the only place the copilot owns data.** The artifacts live in the client's Jira, Confluence, Drive, Salesforce — systems of record the client controls and pays for. The Airtable cross-reference graph is the only data structure the copilot itself owns, and it's project-owned, transferred to the client at engagement end. **The copilot can be removed from the engagement without taking the artifacts with it** — every artifact lives where the client's team can read it, and the graph is a portable Airtable base. This is the same discipline as [Cowork vertical packages](../claude-cowork/): the customer keeps their tools.

**The consultant is structurally in the loop.** Drafts-only floor is non-negotiable and code-enforced through the MCP wrappers, not prompt-enforced. The cross-reference integrity check refuses writes that would leave the graph inconsistent. The SOW-language check refuses commercial drafts that conflict with active scope. The coherence-audit runs daily and surfaces a triage list. **The consultant doesn't have to remember to do the coherence work. They have to review the coherence work the system has already surfaced.** That's the leverage.

**Compliance posture follows the engagement, not the consultancy.** DACH pharma-adjacent engagements run on EU-region Airtable, Cowork on the consultant's device, Anthropic API through Cowork's privacy posture. The graph base is the only consultancy-introduced data store, and it's project-owned and EU-resident. **No client data lands on consultancy infrastructure.**

---

## Architecture diagrams

All diagrams in this case study live inline in this README as text-style ASCII inside fenced code blocks — readable on GitHub, version-controlled, diff-able, no rendering tooling required.

- **v1 team topology** — n8n MainAgent + 3 sub-agents over Telegram. See [The team](#the-team) above.
- **v2 single-cockpit topology** — Cowork + skill library + cross-reference graph. See [Architecture](#architecture) under the v2 section above.
- **Enterprise topology · per-engagement detail** — full layered view: consultant surface · Cowork orchestration · skill library · MCP fleet · systems of record · data residency. See [Enterprise topology · per-engagement detail](#enterprise-topology--per-engagement-detail) above.

---

## What I designed vs what the platforms provide

For clear credit attribution:

| Concern | Platform provides | What I designed on top |
| --- | --- | --- |
| **Consultant cockpit** | [Claude Cowork](https://claude.com/cowork) — desktop AI app, Anthropic Claude with MCP-tool access, native scheduled tasks, file-system access, computer-use | Configured Cowork as the *only* consultant surface for the engagement — authored the `CLAUDE.md` operational-context template for Salesforce delivery, the memory-directory seed (`engagement_profile.md` / `client_voice.md` / `stakeholder_map.md` / `rules.md` / `id_space_rules.md`), the per-engagement project folder shape |
| **Reasoning model** | [Anthropic Claude](https://www.anthropic.com/claude) (Sonnet 4.6, Opus 4.6) | Per-skill model routing — Sonnet for the routine sub-stages, Opus for `transcript-to-fanout` and `user-stories-to-process-flow` where the reasoning load is heavier |
| **Skill library** | [Cowork skill format](https://claude.com/cowork) — `SKILL.md` per skill, frontmatter `name` + `description`, agent picks by description | Authored the **11-skill library** organised around the fan-out shape (entry → sub-stages → composites → safety → infra), the canonical 7-section structure (Purpose / When to use / Inputs / Process / Output / Don'ts / escalation), the skill-naming-is-API-design discipline borrowed from the [Cowork vertical packages](../claude-cowork/) |
| **MCP servers** | [Model Context Protocol](https://modelcontextprotocol.io) — Anthropic-published spec, growing public registry | Per-engagement MCP fleet specification (Granola Drive + Jira + Confluence + Salesforce metadata + Airtable graph + Gmail draft), the read-only-by-default rule for Salesforce metadata, the draft-only rule for Gmail, the integrity-check middleware sitting in front of the Airtable graph writes |
| **Transcript source** | [Granola](https://granola.ai) — local-first meeting recorder + transcriber with Drive export | Granola → Drive → Cowork hourly poll integration; the Cowork scheduled task that picks up new transcripts and triggers `transcript-to-fanout`; the consultant override to invoke the fan-out manually |
| **Project tools** | [Jira](https://www.atlassian.com/software/jira) and [Confluence](https://www.atlassian.com/software/confluence) (Atlassian, client subscription) | Cross-reference metadata block convention — every drafted Jira issue and Confluence page carries the artifact's ID and a back-ref set in a structured metadata block; the read patterns for grounding skills against current state |
| **Salesforce metadata** | Salesforce Metadata API (client org) | Read-only scoping of the integration — the copilot reads object model, field metadata, validation rules, and flow definitions to ground user-story drafts and process-flow generation against the actual org config, but never writes to the org |
| **Cross-reference graph** | [Airtable](https://airtable.com) — relational table store | The **graph schema itself** (Transcripts · Notes · ProcessFlows · UserStories · PlanDeltas · Decisions · ActionItems, with linked-record relationships between them), the ID-space convention (NOTE-XXX, PF-XXX, US-XXX, PP-XXX, DEC-XXX, AI-XXX, T-YYYY-MM-DD), the cross-ref integrity check middleware that enforces graph consistency on every write |
| **HITL gate model** | Neither Cowork nor MCP ships a gate model | Three-tier autonomy (drafts-only floor · cross-ref integrity gate · SOW-language gate), enforced through the per-skill `# Don'ts / escalation` block and code-level integrity middleware in the Airtable write path; same operating-model lineage as the [Cowork vertical packages](../claude-cowork/) |
| **Scheduled work** | [Cowork native scheduled tasks](https://claude.com/cowork) | Hourly Granola-Drive poll · daily 07:00 coherence-audit · Friday 16:00 status-report-weekly draft. All inside Cowork's native scheduled-tasks layer; no n8n. |
| **Compliance posture** | Anthropic + Cowork have their own enterprise privacy posture; the consultant runs on their own Cowork subscription | EU-region Airtable base for DACH engagements; project-owned graph base transferred to client at engagement end; no client data on consultancy infrastructure |

The pattern: **I don't reinvent Salesforce, I don't reinvent Cowork, I don't reinvent the project tools. I do design the skill library, the cross-reference model, the fan-out logic, the HITL flow, and the configuration that makes Cowork into a Salesforce-delivery copilot.**

---

## Stack

| Layer | Component |
| --- | --- |
| **Consultant cockpit** | [Claude Cowork](https://claude.com/cowork) desktop app — Anthropic Claude (Sonnet 4.6 routine · Opus 4.6 heavy reasoning) · native scheduled tasks · MCP-tool support · file-system access · computer-use |
| **Skill library** | 11 skills (`transcript-to-fanout` entry + 6 sub-stages + 1 composite + 2 safety + 1 infra) — each `SKILL.md` in the canonical 7-section structure |
| **Transcript source** | [Granola](https://granola.ai) — local-first meeting recorder + transcriber with Drive export |
| **MCP fleet** | Granola Drive read · [Jira MCP](https://github.com/modelcontextprotocol/servers) (read + draft) · [Confluence MCP](https://github.com/modelcontextprotocol/servers) (read + draft) · [Google Drive MCP](https://github.com/modelcontextprotocol/servers) (read) · Salesforce metadata MCP (read-only) · [Airtable MCP](https://github.com/modelcontextprotocol/servers) (graph base · read + atomic write) · [Gmail MCP](https://github.com/modelcontextprotocol/servers) (draft only) |
| **Cross-reference graph** | [Airtable](https://airtable.com) — project-owned base, EU-region for DACH engagements, transferred to client at engagement end |
| **Project tools (client-owned)** | [Jira](https://www.atlassian.com/software/jira) + [Confluence](https://www.atlassian.com/software/confluence) (client subscription) |
| **Platform under delivery** | [Salesforce](https://www.salesforce.com) Service Cloud + Experience Cloud (client org) — metadata read-only |
| **Document tools (client-owned)** | Google Drive (client tenancy) — read-only, prior-engagement artifacts only |
| **HITL gate model** | Drafts-only floor (universal) · cross-ref integrity check (code-enforced in MCP write path) · SOW-language check (composite skill) |
| **Compliance posture** | EU-region Airtable for DACH · Cowork on consultant's device · no client data on consultancy infrastructure · graph base transferred to client at engagement end |

---

## Deeper reading

### Related case studies in this portfolio

- [`claude-cowork`](../claude-cowork/) — same configure-don't-build operating model applied to SMB verticals (Practice Command · Legal Efficiency Suite · Field Service Command). This case study is that argument applied to the platform I came from. The skill-naming-is-API-design discipline and the `# Don'ts / escalation` block convention are inherited directly from the Cowork vertical packages.
- [`hugo`](../hugo/) — productized AI marketing employee where each tenant runs on the customer's own ChatGPT Plus. **Pattern C** there (single agent + MCP tools, summarize the return) is the philosophical cousin of v2 here: deterministic work — including cross-reference integrity — lives in the MCP wrapper, not in the agent's prompt. The Hugo v1 → v2 collapse and this case study's v1 → v2 collapse share the same shape: artifact-quality was never the problem; the topology around the artifacts was.
- [`openclaw-hermes-evolution`](../openclaw-hermes-evolution/) — multi-agent build system shipping Psinest. The v1 → v2 narrative there (autonomous build agents in OpenClaw → single-orchestrator Hermes shape) is the same architecture lesson at a different scale: when the multi-agent design has a coherence problem, collapse to a single cockpit with the right tools.
- [`psinest`](../psinest/) — live RGPD + ERS healthcare CRM. The audit-trail discipline and the consultant-is-in-the-loop principle here borrow from Psinest's centralised tenancy and approval patterns.
- [`skoda-ai-ops`](../skoda-ai-ops/) — content operations for a German automotive referral brand, also run from Cowork. Different domain, same "Cowork is the cockpit and the scheduler" architectural bet.

### External

- [Claude Cowork](https://claude.com/cowork) — Anthropic's desktop AI app, the consultant cockpit
- [Model Context Protocol specification](https://modelcontextprotocol.io) — the integration plane
- [Anthropic Claude](https://www.anthropic.com/claude) — Sonnet 4.6 + Opus 4.6, the reasoning models powering every Cowork session
- [Salesforce Metadata API documentation](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_intro.htm) — the read surface this copilot grounds against
- [Granola](https://granola.ai) — local-first meeting recorder + transcriber, the transcript source
- [Jira REST API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/) + [Confluence REST API](https://developer.atlassian.com/cloud/confluence/rest/v2/intro/) — the project-tool integration surface
- [Airtable](https://airtable.com) — the cross-reference graph data store

### The lineage essays

- The [Cowork vertical packages case study](../claude-cowork/) articulates the configure-don't-build operating model in its general form. This case study is that argument applied to Salesforce delivery — the platform I came from before AI consulting.
- The [Configure-don't-build essay](../writing/configure-dont-build.md) generalises the operating model — the Salesforce-consulting discipline of *configure the platform, don't build the substrate* — across regulated-domain AI consulting practices. This case study is the canonical Salesforce-specific instance of that essay.
- The architectural insight that **the cross-reference graph is the load-bearing piece** is the Salesforce-delivery-specific contribution. It's the answer to a problem the Cowork vertical packages don't have (because SMB owners aren't trying to keep five-month engagements coherent across rewrites), and it's the architectural feature that earns the copilot its place on a Salesforce delivery.

---

## Status

- 🛑 **v1 (n8n multi-agent + Telegram)** — designed, prototyped, never deployed to a live engagement. The coherence failures surfaced during dry-run before the first client transcript landed. Sub-agent template libraries (25 templates total) archived as reference material under `agent-drafts/`; some templates were ported forward into v2 skill outputs.
- ✅ **v2 (Cowork + skill library + cross-reference graph)** — current architecture. **11 skills shipped:** `transcript-to-fanout`, `transcript-to-notes`, `notes-to-user-stories`, `user-stories-to-process-flow`, `project-plan-update`, `decision-log-entry`, `action-item-extract`, `status-report-weekly`, `sow-language-check`, `coherence-audit`, `cross-ref-integrity-check`.
- ✅ **Live engagement** — running on a Salesforce delivery at a DACH pharma-adjacent client (Service Cloud + Experience Cloud, ~9-month rollout, hybrid in-house + partner team). Granola auto-ingest is the primary input path; the consultant invokes ad-hoc fan-outs against meeting transcripts that didn't come through the auto-ingest (steering committee calls recorded on Zoom and exported manually).
- ✅ **Cross-reference graph** — Airtable base running with the full schema (Transcripts · Notes · ProcessFlows · UserStories · PlanDeltas · Decisions · ActionItems · linked-record relationships). EU-region workspace. Daily coherence-audit running. Cross-ref integrity middleware enforced on every write.
- ✅ **Diagrams** — all architecture diagrams shipped as inline text-style ASCII in fenced code blocks (v1 team topology · v2 single-cockpit topology · enterprise per-engagement topology).
- 🚧 **Multi-consultant engagement support** — currently, the cockpit is single-consultant (the engagement runs with one delivery lead and the copilot configured on their device). Multi-consultant pattern (shared graph base, per-consultant Cowork installs, conflict-detection on simultaneous writes) is designed but not yet validated.
- 🚧 **Salesforce config write-back** — the Metadata API integration is read-only today. Write-back for user-story acceptance criteria into custom-object metadata (so the org carries the AC alongside the field) is designed; the HITL gate would treat it as a separate approval class (org-impacting). Not shipped yet.
- 🚧 **Per-vertical Salesforce delivery extensions** — the current skill library is industry-agnostic; vertical extensions for Health Cloud, Financial Services Cloud, and Manufacturing Cloud engagements are scaffolded folders awaiting in-engagement work to land the vertical-specific templates and ID-space rules.
- 🚧 **External transcript-source adapters** — Granola is the primary path; adapters for Fireflies, Otter, and Read.ai are designed and partially built. The integration surface is the same Drive-folder landing zone; only the export pipeline upstream differs.

The sequencing is deliberate: ship v1 first to prove that artifact-quality wasn't the bottleneck (the prototype's artifacts were fine; the cross-reference drift was the failure), then collapse to v2 to fix the topology rather than tune the artifacts. Multi-consultant comes after single-consultant proves out on a full engagement cycle, because the conflict-detection patterns are easier to design from a baseline that's stable than from a baseline that's still moving. Salesforce write-back comes after multi-consultant, because org-impacting writes carry an HITL class that needs the multi-consultant gate model in place first.

---

## Acknowledgements

- **[Claude Cowork](https://claude.com/cowork)** (Anthropic) — the desktop AI app that makes this copilot possible. Same Cowork install, same Anthropic Claude, same MCP-tool support, same native scheduled tasks. The architectural property the entire case study depends on is that Cowork is general enough to be a consultant's Salesforce-delivery cockpit *and* a clinic owner's practice cockpit *and* a marketing-employee cockpit — without changing the app.
- **[Model Context Protocol](https://modelcontextprotocol.io)** (Anthropic, MIT) — the integration plane that lets the per-engagement MCP fleet plug in cleanly. The "naming a skill is API design" insight applies one layer up to MCP server naming as well.
- **[Salesforce](https://www.salesforce.com)** — the platform being configured by the engagement this copilot supports. Twenty years of Salesforce-consulting practice taught the operating model — configure the platform, don't build from scratch — that this copilot now embodies for AI delivery.
- **Adopted MCP servers** — [`jira-mcp`](https://github.com/modelcontextprotocol/servers), [`confluence-mcp`](https://github.com/modelcontextprotocol/servers), [`gdrive-mcp`](https://github.com/modelcontextprotocol/servers), [`airtable-mcp`](https://github.com/modelcontextprotocol/servers), [`gmail-mcp`](https://github.com/modelcontextprotocol/servers), Salesforce metadata MCP. These are the heavy lifting on the integration side — I configure them per engagement, I don't build them.
- **[Granola](https://granola.ai)** — the local-first meeting recorder + transcriber that produces the primary input. The Drive-export integration is what makes the auto-ingest path clean.
- **[Atlassian Jira + Confluence](https://www.atlassian.com)** — the project-tool plane this engagement uses. Drafts land here; the consultant taps to commit.
- **[Airtable](https://airtable.com)** — the cross-reference graph data store. The graph is what makes the artifacts queryable.
- **[Anthropic Claude](https://www.anthropic.com/claude)** (Sonnet 4.6 routine · Opus 4.6 heavy reasoning) — the reasoning model powering every Cowork session, interactive and scheduled.

**What's mine:** the skill library structure (11 skills organised around the fan-out shape — entry + sub-stages + composites + safety + infra), the cross-reference graph schema (Transcripts · Notes · ProcessFlows · UserStories · PlanDeltas · Decisions · ActionItems with linked-record relationships), the ID-space convention (NOTE-XXX, PF-XXX, US-XXX, PP-XXX, DEC-XXX, AI-XXX, T-YYYY-MM-DD), the cross-ref integrity check middleware (code-enforced in the MCP write path), the SOW-language safety skill, the daily coherence-audit and Friday status-report-weekly cadences, the HITL flow (drafts-only floor + structural gates), the configuration of Cowork as a Salesforce-delivery cockpit (`CLAUDE.md` template, memory directory seed, scheduled-tasks cadence), and the sequencing argument (v1 first to prove artifact-quality wasn't the problem → v2 to fix the topology).

The configure-don't-build operating model — build the cockpit once on a reliable platform, then configure it per engagement — is the same discipline Salesforce consultants have lived by for twenty years. **AI consulting works best when it mirrors the practices that already work in productized professional services.** The Salesforce Delivery Copilot is that argument applied to the platform I came from; the bridge between my Salesforce-delivery past and my AI-consulting present holds because the operating model is the same on both sides.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
