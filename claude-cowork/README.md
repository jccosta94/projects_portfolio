# Claude Cowork Vertical Workflows
**The same Cowork install, three operator cockpits. Verticalisation done as configuration — MCP fleet, skill library, approval gates — not as code.**

This is a case study in **how to verticalise a horizontal AI desktop app for non-developer professionals** — specifically: how I take [Claude Cowork](https://claude.com/cowork) (Anthropic's desktop AI app with MCP-tool access) and turn the same install into three different operator cockpits — a clinic owner's, a law-firm associate's, and a home-services dispatcher's — by changing only what gets wired in. Same Cowork. Same Claude. Different MCP fleet, different skill library, different human-in-the-loop gates. No custom code per vertical; everything that's "vertical" is configuration.

The three vertical packages — **Practice Command** (Healthcare Clinics), **Legal Efficiency Suite** (Law Firms), **Field Service Command** (Home Services) — sit on top of an 11-skill core library plus per-vertical extensions. Two of them are live in pilot today (Practice Command at an aesthetic clinic in Portugal; Legal Efficiency Suite with two boutique firms — one Portuguese, one cross-border European); the third is fully designed and packaged but waiting for the first signed pilot. The constraint that triggered the architecture work: a one-person integration practice can't credibly sell three different "bespoke AI builds" per vertical and survive the delivery cost. The only way the economics work is if every vertical engagement is the same Cowork environment with a configured fleet on top.

### What I architected

**Cowork as the operator's only cockpit.** Every engagement is the same shape — the owner installs Claude Cowork on their primary device, the cockpit gets a configured `CLAUDE.md` operational context, a seeded memory directory (user profile, brand voice, top clients/accounts, suppliers, exclusion rules), an MCP fleet wired to the tools they already use (Gmail, Google Calendar, Google Drive, Airtable, Google Business Profile, the right CRM, the right accounting tool — read-only — plus WhatsApp Business via an n8n bridge for inbound), an 11-skill library plus the vertical-specific add-ons, and a small number of scheduled tasks for the recurring digests. There is no separate admin dashboard. There is no per-vertical custom UI. There is no second app the owner has to learn. **Cowork is the dashboard, the editor, the orchestrator, and the keyboard for the entire engagement.**

**Three vertical templates on top of a shared core.** The 11 core skills (`admin-inbox-triage`, `admin-meeting-prep`, `admin-owner-dashboard`, `marketing-content`, `marketing-calendar`, `marketing-analytics-summary`, `sales-followup`, `sales-proposal`, `sales-pipeline-review`, `accounting-prep-monthly`, `accounting-handoff-pack`) cover the workflows every SMB owner needs — inbox, meetings, owner dashboard, content, sales, accounting prep. The verticals layer on top: Practice Command adds patient-recall, no-show-reduction, deposit-collection, intake-digitalization, and special-category-data handling; Legal Efficiency Suite adds matter-intake-with-conflict-check, deadline tracking, document-draft-with-review-gate, and privilege-preserving communication logging; Field Service Command adds missed-call-capture, job-intake-and-dispatch, technician-skill-matrix matching, route-optimization briefs, on-site checklists, and review cascades. Each vertical's add-ons reference the same memory files and read from the same MCP fleet — they don't re-implement core capabilities, they extend them.

**Three classes of human-in-the-loop gate, tuned per vertical.** Every skill in the library has a hard `# Don'ts / escalation` block ("never publish autonomously", "never send on the owner's behalf", "never invent numbers", "drafts only"). On top of that floor, each vertical adds a vertical-specific gate: Practice Command treats anything touching clinical data as an automatic owner-eyes-only event (no patient communication can fire without the owner's tap, no recall message can be drafted with patient identifiers the system pulled from the PMS without re-consent verification); Legal Efficiency Suite treats every document the system drafts as a paralegal-grade artifact that requires a partner or supervising associate's review before it touches a client, a court, or opposing counsel (this is the supervision rule that makes OpenClaw — our autonomous-agent tier — strictly off-limits for law firms, in all three markets we operate in); Field Service Command treats every dispatched job assignment as a soft draft that the dispatcher reads in their messaging app and confirms with a single tap before the technician sees it. **Same Cowork. Same Claude. Three different approval shapes.**

### The architectural insight

**Cowork is a horizontal platform; verticalisation is a configuration concern, not a code concern.** The first version of the consulting practice — early 2026, before I'd shipped any vertical templates — was a backlog of "bespoke" engagements where every prospect got a custom-scoped build. The cost structure didn't work. The second version was the inverse: a single horizontal Command Center package that I sold to everyone regardless of vertical. The marketing was clean, but the deliverable was thin in every vertical because the workflows it shipped were either too generic to move the operator's bottleneck or too specific to a vertical other than the one in front of me. **Verticalisation as configuration** is the third version, and it's the one that ships: the platform is fixed, the skill library is fixed, the cockpit shape is fixed; what varies per vertical is the MCP fleet (which PMS, which DMS, which dispatch tool), the skill library composition (which 11 core + which vertical-specific extensions), the memory seeds (what counts as a VIP, what's special-category data, what's privileged), and the HITL gate model. **None of this is code. All of it is configuration.**

**Naming a skill is API design.** A skill's `description` field is what Claude reads to decide whether to invoke it. The line between `admin-inbox-triage` and `admin-meeting-prep` is the line between "what gets pulled up at 09:00" and "what gets pulled up before a 14:00 with a specific person on the calendar"; if either name leaks into the other's territory (an "inbox prep for meetings" skill would compete with both), Cowork picks the wrong one and the owner's first impression is that the system is flaky. I treat skill names like public-API endpoint names: stable, mutually exclusive, descriptive enough that the owner can predict what they'll get without reading the body. Adding a new skill is a deliberate naming decision, not a routine one — the cost of a confusing name is paid every time the owner asks a question that could plausibly invoke two skills and the system picks the wrong one.

**The sequence: Practice Command first, Legal Efficiency Suite second, Field Service Command third.** Practice Command was the first vertical I shipped because the deliverable is concrete and the bottleneck is unambiguous (chair utilization, no-show rate, the recall loop — every owner I talk to can name the number). The compliance constraint (RGPD plus, in CH, FADP for health data; plus ERS-level professional rules for clinical practices in Portugal) forces design discipline early — if the design survives clinical-data scrutiny, it survives most other things. Legal Efficiency Suite was second because the buying motion is different (managing partners are slower to commit, but the contract values are higher and the workflows are durable). Field Service Command is third because the operator persona is the hardest to reach asynchronously (home-services owners are in vans, not at desks), and the design needed to be field-first before I'd take a pilot — which it now is, sitting in the catalog, waiting. **Doing them in a different order — leading with Field Service Command, say — would have meant designing the discipline-imposing constraints later, after I'd already locked in patterns that were too loose to retrofit.**

---

## TL;DR

| | **Practice Command** | **Legal Efficiency Suite** | **Field Service Command** | **Command Center (horizontal)** |
|---|---|---|---|---|
| **Vertical** | Healthcare clinics — dental, physio, aesthetic, chiropractic, psychology | Boutique law firms, solo practitioners, small partnerships | Plumbing, electrical, HVAC, cleaning, landscaping, locksmith, painting | Generic SMB — owner doesn't fit cleanly into the three verticals |
| **Operator role** | Clinic owner / lead practitioner — often the bottleneck chair-time generator | Managing partner / supervising associate | Owner-operator (often also the lead technician) / dispatcher | SMB owner — typically solo or with one ops person |
| **MCP fleet (vertical-specific add)** | Practice management read API (Dentally / Clinicminds / Doctolib / Doctoralia / Optident, depending on market), special-category-data-aware Airtable schema | Document management read (NetDocuments / iManage / SharePoint / file system), conflict-list source, time-tracking export | GBP write (review cascade), Airtable dispatch board, WhatsApp Business inbound via n8n bridge, mapping-API briefs | None added — runs on the core MCP fleet alone |
| **Skill library size** | 11 core + 7 vertical (patient-recall, no-show-reduction, deposit-collection, intake-digitalization, recall-loop-audit, review-cascade-clinical, gbp-clinical-post) | 11 core + 6 vertical (matter-intake-with-conflict, deadline-diary-audit, document-draft-review-gate, privilege-aware-comms-log, billable-time-capture-prompt, client-status-snapshot) | 11 core + 7 vertical (missed-call-capture, job-intake-dispatch, route-brief, on-site-checklist, review-cascade-field, gbp-field-post, reactivation-cadence) | 11 core, that's it |
| **Owner cockpit** | Cowork desktop on the lead practitioner's primary device | Cowork desktop on the managing partner's primary device | Cowork desktop on the owner-operator's primary device + a Telegram bridge for in-van approvals | Cowork desktop on the owner's primary device |
| **HITL gate model** | Owner-eyes-only on anything touching patient identifiers · drafts-only on every external communication · re-consent verification before any recall outreach | Supervising-lawyer review on every drafted document · privilege-aware redaction by default · no agent-driven communication with clients, courts, or opposing counsel · OpenClaw strictly **not available** | Dispatcher confirmation on every job assignment · owner approval on every quote sent · review-request cascade fires only on `job_completed` event | Drafts-only everywhere · standard core gates (see [skill library](https://github.com/jccosta94/oblqai/tree/main/cowork-skills/core)) |
| **Tier** | Claude Cowork Setup ("Command", human-in-the-loop) | Claude Cowork Setup — **OpenClaw is never offered for law firms** | Claude Cowork Setup; OpenClaw available as a later upgrade path | Claude Cowork Setup only |
| **Operating state today** | **LIVE** — first pilot with an aesthetic clinic in Portugal | **LIVE** — two pilots: one Portuguese boutique, one cross-border-European solo practitioner | Designed and packaged — pitch deck, vertical checklist, capability map all shipped; awaiting first signed pilot | **LIVE** — one off-blueprint pilot (an agency + small-construction operator) |
| **Pricing (PT baseline)** | One-off €8,000 build · €950/mo managed | Not yet finalised — defer to Joao | Not yet finalised — defer to Joao | One-off €2,500–3,500 · €1,499/mo managed |

**Headline insight: the cockpit shape is fixed; what's variable per vertical is the wiring on either side of it.** The owner sits in the same Cowork app regardless of vertical. What changes is which tools Cowork can see (MCP fleet), which workflows it offers (skill library), and which actions require an owner tap before they fire (HITL gate model).

---

## Why this matters

Most consultancies trying to bring AI to non-developer professionals in 2026 hit two walls in sequence: **first the productisation wall, then the per-vertical-discipline wall.**

**Wall 1 — Productisation.** A "bespoke AI build for your business" sounds compelling on a call and breaks on delivery. Every engagement starts from zero, every workflow is custom, every memory file is hand-authored, every connector is debugged in production. The cost-of-delivery per engagement swallows the engagement fee, and the second customer's build can't reuse anything from the first because the first was a snowflake. **This is the wall the "horizontal Command Center" version of the practice was supposed to solve** — and it did, but only by going generic enough that the workflows didn't move the operator's bottleneck in any given vertical. Selling the same Command Center to a clinic and to a law firm meant the clinic didn't get a recall loop and the firm didn't get conflict-check intake; both got an inbox triage and a content calendar and felt under-served.

**Wall 2 — Per-vertical discipline.** Each vertical has a non-negotiable discipline you can't paper over with a horizontal package — healthcare has special-category data and clinical-trust expectations, legal has supervision and privilege rules (which is why the autonomous-agent tier is strictly inappropriate for law firms in all three markets we operate in), home services has dispatch-and-route logistics that text-only AI can't reason about without the right inputs. Trying to bolt these on later — after the engagement is sold as horizontal and the owner has discovered the gap — is a customer-trust event. **The vertical-template approach front-loads the discipline into the package**, so the prospect sees a pitch that's actually shaped like their business and a deliverable that respects the constraints their profession imposes.

The lesson is **the sequence**: build a horizontal cockpit and a horizontal skill library first (because that's where the leverage of "configuration not code" actually lives), then verticalise *on top of it* — not under it. Verticalisation as a layer above the horizontal core gives you re-usability per vertical; verticalisation as the foundation forces you to reinvent the cockpit for every package and you're back to bespoke. Doing it in a different order — vertical-first, then trying to extract a horizontal core later — produces three half-shared codebases and no leverage. Vertical-on-top-of-horizontal produces three configured cockpits and one shared library.

The lesson is also **what stays constant.** Across all three verticals, the cockpit is the same (Cowork desktop), the model is the same (Anthropic Claude — Sonnet for routine work, Opus for the heavier reasoning skills), the orchestration is the same (Cowork's native scheduled tasks for the recurring digests; n8n only as the webhook bridge for the things Cowork can't be the inbound endpoint for — WhatsApp Business inbound, primarily), the memory pattern is the same (a `memory/` directory with `user_profile.md`, `brand_voice.md`, `clients_top.md`, `rules.md`, plus vertical-specific files), the cadence is the same (daily digest, weekly owner dashboard, monthly accountant pack, monthly review-and-optimize call), and the principle is the same — agents propose, the human disposes.

---

## Implementation plan · the 4-phase ~30-day arc

Every engagement, regardless of vertical, runs through the same four phases. The vertical changes *what* gets configured; it doesn't change *how* the configuration gets done. Day numbers are nominal — compressed for a simple shop, extended for a complex one — but the gates between phases are fixed: phase N+1 doesn't start until phase N has produced its signed deliverable.

### The phases

```text
                           Free 30-min discovery call
                                       │
                                       ▼
                       ┌───────────────────────────────┐
                       │ Phase 1 — Discovery & Design  │   ← Days 1–3
                       │                               │     paid discovery
                       │  · workbook walk-through      │     & audit
                       │  · stakeholder interviews     │
                       │  · workflow capture + map     │
                       │  · ROI scoring                │
                       └──────────────┬────────────────┘
                                      │ Implementation Blueprint Letter (signed)
                                      ▼
                       ┌───────────────────────────────┐
                       │ Phase 2 — Build & Configure   │   ← Days 4–10
                       │                               │
                       │  · Cowork install + CLAUDE.md │
                       │  · memory directory seed      │
                       │  · MCP fleet wiring per       │
                       │    vertical                   │
                       │  · Airtable schema deploy     │
                       │  · skill library install      │
                       │  · scheduled tasks configured │
                       │  · n8n bridge flows (only the │
                       │    3 exception cases)         │
                       └──────────────┬────────────────┘
                                      │ working cockpit, runbook draft
                                      ▼
                       ┌───────────────────────────────┐
                       │ Phase 3 — Testing & Launch    │   ← Days 11–14
                       │                               │
                       │  · dry-run each skill         │
                       │    surface on live data       │
                       │  · brand voice ratified       │
                       │    (3 sample artifacts)       │
                       │  · accountant signs off on    │
                       │    handoff-pack format        │
                       │  · 90-min owner training      │
                       │  · staged go-live (mktg+sales │
                       │    week 1, admin+acct week 2) │
                       └──────────────┬────────────────┘
                                      │ training recording, final runbook,
                                      │ first usable monthly accountant pack
                                      ▼
                       ┌───────────────────────────────┐
                       │ Phase 4 — Optimization &      │   ← Day 15 onward
                       │           Managed Service     │
                       │                               │
                       │  · monthly review call (60m)  │
                       │  · backlog grooming           │
                       │  · automation health check    │
                       │  · monthly business command   │
                       │    report                     │
                       │  · 1–2 new/upgraded workflows │
                       │    from the backlog per month │
                       └───────────────────────────────┘
```

### What each phase produces

**Phase 1 — Discovery & Design (Days 1–3).** I open the engagement with the canonical discovery workbook (see below) and run it as a structured conversation rather than a form — stakeholder interviews, workflow capture, role cards, scoring. Output: a signed Implementation Blueprint Letter that locks scope, the MCP fleet to enable, the skill library composition, the HITL gate model, dates, price, and the owner's responsibilities for the next phase. Nothing gets built until this is signed.

**Phase 2 — Build & Configure (Days 4–10).** The cockpit goes up: Cowork installed on the owner's primary device, `CLAUDE.md` drafted from discovery inputs and ratified, memory directory seeded, MCP fleet wired (the core fleet plus the vertical add — see the per-vertical tables above), Airtable schema deployed per the vertical's template, skill library installed, Cowork scheduled tasks configured for the recurring cadence, n8n bridge flows added *only* for the three exception cases (WhatsApp Business inbound, no-MCP-tool gaps, external deadman). The runbook draft lands at the end of this phase.

**Phase 3 — Testing & Launch (Days 11–14).** Each skill surface gets dry-run on the operator's real data and the owner's edits flow back into the memory files. Brand voice is ratified across three sample artifacts (LinkedIn post, newsletter, client email) — and is not "done" until the owner accepts them without rewrite. For verticals with an accounting handoff, the accountant has to sign off on the pack format *before* go-live; we don't ship a handoff format the accountant won't accept. Owner training is a 90-minute recorded session. Go-live is staged — marketing and sales surfaces week 1, admin and accounting prep week 2 — so the operator's mental load stays manageable.

**Phase 4 — Optimization & Managed Service (Day 15 onward).** Hand-off into the recurring monthly motion: a 60-minute review-and-optimize call, backlog grooming (the owner keeps a shared list of fixes / tunes / net-new workflows; we ship one or two per month), an automation health check (MCP tokens, n8n flow success rates, connector heartbeats, memory file drift), and a written monthly Business Command report summarising what Claude did, the deltas in each surface, and where the next month's leverage is.

### The canonical discovery workbook

Discovery isn't a slide deck or a generic intake form — it's a **15-sheet Excel workbook** I walk every prospect through during Phase 1. The workbook lives at `Discovery Docs/claude_coworker_requirements_blueprint_with_workflow_map.xlsx` and replaced the old 14-markdown discovery kit on 2026-05-16. Same conversation every time; the vertical-specific checklists (under `Discovery Docs/Vertical Checklists/`) layer on top for Healthcare, Legal, and Home Services.

| Sheet | What it captures |
|---|---|
| **Overview** | Engagement scope, vertical fit confirmation, market (PT / DE / CH), discovery cadence |
| **Master Discovery** | The full structured interview — stakeholders, current toolchain, pain points, KPIs, constraints |
| **Workflow Blueprints** | The 11-core + vertical-add workflow shapes the engagement *could* ship — anchored to the skill library |
| **Workflow Capture** | What workflows the operator runs *today* — who does what, where it lives, how long it takes |
| **Workflow Map** | The mapping layer — every captured workflow either matches a blueprint (configure), needs adaptation (configure + tune memory), or is out-of-scope (flag) |
| **Scoring Model** | ROI scoring per workflow — impact × frequency × time-saved-per-run, used to sequence Phase 2 |
| **Role Cards** | One card per role at the client (owner, dispatcher, paralegal, front-desk, etc.) — what they need from Cowork |
| **Prospect Form** | The pre-call intake the prospect fills before the discovery call — keeps the call focused |
| **Call Structure** | The discovery-call agenda template — 30-min free version and the paid-audit version |
| **Lists** | Reference lists used across the sheets (verticals, packages, markets, tier names, MCP server names) — single source of truth for dropdowns |
| **Skill Definitions** | The 11 core + per-vertical add skills, with their `description` field and HITL gate type — this is what gets installed in Phase 2 |
| **MCP & Connectors** | The MCP fleet to wire per engagement, scopes (read / read-write), auth model, who owns the credential |
| **Actions & Permissions** | What each skill can do autonomously vs what requires an owner tap — the HITL gate model as a per-action matrix |
| **Claude Components** | The Cowork-side components installed (skills, scheduled tasks, memory files, CLAUDE.md sections) per engagement |
| **Plugin Requirements** | Any Cowork plugins the engagement needs (and any not-yet-built MCPs the engagement is blocked on) |

The workbook is the load-bearing artifact of Phase 1. The signed Implementation Blueprint Letter is derived from the workbook; the Phase 2 build checklist is derived from the workbook; the monthly business reports in Phase 4 track deltas against the workbook's Scoring Model. It is the same instrument every engagement uses, regardless of vertical — what changes per vertical is which checklists layer on top and which Workflow Blueprints get selected.

---

## v1 — Practice Command (Healthcare Clinics)

The most mature vertical and the one that taught me the template. Live pilot with an aesthetic clinic in Portugal. The package targets dental, physio, aesthetic, chiropractic, and small psychology / mental-health practices — non-hospital private clinics where the owner is also the lead practitioner.

### Operator cockpit

```text
                       Lead practitioner (clinic owner)
                                  │
                                  │  Mac mini at front desk
                                  ▼
                       ┌──────────────────────┐
                       │  Claude Cowork       │   ← single app, one window
                       │  desktop             │     no separate admin UI
                       └──────────┬───────────┘
                                  │ MCP fleet, configured per clinic
                                  ▼
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
       ┌─────────────┐                   ┌──────────────────────┐
       │ Patient-    │                   │ Operations           │   ← scheduling,
       │  facing     │                   │  surface             │     CRM, files,
       │  surface    │                   │                      │     accounting
       │ (drafts     │                   │                      │     prep
       │  only)      │                   │                      │
       └──────┬──────┘                   └──────────┬───────────┘
              │                                     │
              ▼                                     ▼
       ┌─────────────┐                   ┌──────────────────────┐
       │ Practice    │                   │ Front-desk           │
       │  mgmt API   │                   │  toolchain           │
       │  (PMS read) │                   │  Gmail · GCal ·      │
       │  Dentally · │                   │  Drive · Airtable    │
       │  Clinicminds│                   │  (CRM-lite +         │
       │  · Doctolib │                   │  invoice tracker) ·  │
       │  · Doctoralia                  │  GBP · accounting    │
       │  · Optident │                   │  (read-only) ·       │
       │  (per market)                   │  WhatsApp via n8n    │
       └──────┬──────┘                   └──────────┬───────────┘
              │                                     │
              └──────────────────┬──────────────────┘
                                 │ everything writes back via the same Airtable
                                 ▼
                      ┌──────────────────────┐
                      │ Owner-eyes-only ⏸   │   ← HUMAN GATE 1
                      │  approval inbox      │     anything touching
                      │  inside Cowork chat  │     patient identifiers
                      └──────────────────────┘
```

### Skill library

11 core + 7 vertical add-ons:

| Skill | Layer | What it does |
|---|---|---|
| `admin-inbox-triage` | core | Daily 09:00 triage of unread Gmail; P0–P3 buckets; drafts for P2 only; never sends |
| `admin-meeting-prep` | core | Pre-meeting brief from calendar event + attendee history |
| `admin-owner-dashboard` | core | Weekly state-of-clinic summary: chair utilization, no-show rate, AR aging, content output |
| `marketing-content` | core | Single drafted piece (LinkedIn / newsletter / blog) in clinic voice; drafts only |
| `marketing-calendar` | core | Rolling 30/60/90 content calendar; surfaces gaps |
| `marketing-analytics-summary` | core | Plain-English narrative of channel performance |
| `sales-followup` | core | Personalised follow-up to enquirers who haven't booked |
| `sales-proposal` | core | Treatment-plan quote draft from the firm's template |
| `sales-pipeline-review` | core | Weekly enquiry-to-booking diagnostic |
| `accounting-prep-monthly` | core | Cash-flow narrative + invoice aging; preparation only |
| `accounting-handoff-pack` | core | Monthly accountant pack in the agreed format |
| `patient-recall-loop` | vertical | Weekly audit of patients overdue for recall; drafts in the owner's voice; **never sends** |
| `no-show-reduction-audit` | vertical | Monthly diagnostic of no-show patterns; surfaces booking-policy changes for the owner to decide |
| `deposit-collection-prompt` | vertical | Drafts a deposit-request message for any new booking flagged as high-value or high-no-show-risk |
| `intake-digitalization-brief` | vertical | Converts a paper intake form into a structured online intake spec (for the practice manager to deploy) |
| `recall-loop-audit` | vertical | Monthly KPI — what % of due-recall patients responded; what dropped |
| `review-cascade-clinical` | vertical | Post-treatment review-request draft tuned to the regulator's rules in the operator's market |
| `gbp-clinical-post` | vertical | Drafts a GBP post in the right voice for a regulated clinic; flags anything that would breach advertising rules |

### HITL gates

Three hard gates above the standard drafts-only floor:

- **Patient-identifier gate.** Any output that contains a patient name, phone number, or treatment history is held inside Cowork as an owner-eyes-only artifact and never sent or posted by the agent. The owner taps to send.
- **Re-consent verification.** Before any recall message is drafted with data pulled from the practice management system, the skill checks the Airtable `consent_recall = true` flag on the patient record. If the flag is missing or false, the skill drafts a re-consent message instead and surfaces the gap.
- **Regulator-aware review-request draft.** The `review-cascade-clinical` skill enforces the operator-market's advertising-rules denylist (PT ERS, DE HWG, CH FADP for special-category data) before any review-request goes out. The owner still taps to send.

---

## v2 — Legal Efficiency Suite (Law Firms)

Second-shipped vertical and the one with the strictest discipline. Live with two pilots: a Portuguese boutique partnership and a solo practitioner running cross-border European matters. The package targets small-to-mid law firms, solo practitioners, and boutique practices.

### Operator cockpit

```text
                       Managing partner / supervising associate
                                  │
                                  │  desktop in chambers
                                  ▼
                       ┌──────────────────────┐
                       │  Claude Cowork       │   ← single app, one window
                       │  desktop             │     OpenClaw is NOT
                       └──────────┬───────────┘     available for law firms
                                  │
                                  ▼
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
       ┌─────────────┐                   ┌──────────────────────┐
       │ Matter      │                   │ Practice             │   ← billing,
       │  surface    │                   │  surface             │     calendar,
       │ (drafts     │                   │                      │     intake,
       │  for        │                   │                      │     comms log
       │  partner    │                   │                      │
       │  review)    │                   │                      │
       └──────┬──────┘                   └──────────┬───────────┘
              │                                     │
              ▼                                     ▼
       ┌─────────────┐                   ┌──────────────────────┐
       │ DMS read    │                   │ Practice toolchain   │
       │ NetDocuments│                   │  Gmail · GCal ·      │
       │ iManage ·   │                   │  Drive · Airtable    │
       │ SharePoint ·│                   │  (Matters + Time +   │
       │ file system │                   │  Intake) · CRM ·     │
       │ (read-only) │                   │  accounting (read)   │
       └──────┬──────┘                   └──────────┬───────────┘
              │                                     │
              └──────────────────┬──────────────────┘
                                 │ time + matter + intake state
                                 ▼
                      ┌──────────────────────┐
                      │ Supervising-lawyer ⏸│   ← HUMAN GATE 1
                      │  review queue        │     every drafted
                      │  inside Cowork chat  │     document, every
                      └──────────────────────┘     external comms
```

### Skill library

11 core + 6 vertical add-ons:

| Skill | Layer | What it does |
|---|---|---|
| `admin-inbox-triage` … `accounting-handoff-pack` | core | Same 11-skill core as Practice Command |
| `matter-intake-with-conflict` | vertical | Captures inquiry intake into structured form; runs conflict-check against the firm's matter list; flags any potential conflict for partner review before engagement letter drafts |
| `deadline-diary-audit` | vertical | Weekly audit of every active matter's open limitation / court / regulatory deadline against the calendar; surfaces single-points-of-failure |
| `document-draft-review-gate` | vertical | Drafts a memo / letter / engagement letter from the firm's template; tags the artifact `awaiting-supervisor-review`; refuses to send under any circumstance |
| `privilege-aware-comms-log` | vertical | Logs client communications into the Airtable matter record with privilege-aware redaction by default; partner can un-redact, agent cannot |
| `billable-time-capture-prompt` | vertical | End-of-day prompt that walks the timekeeper through unbilled-time recovery from calendar + email + document touch events |
| `client-status-snapshot` | vertical | Monthly per-client snapshot ready to send (after partner edit): active matters, recent activity, upcoming deadlines |

### HITL gates

Three hard gates above the standard drafts-only floor — all enforced because professional responsibility, supervision, and privilege rules in every market we operate in (PT, DE, CH) say so:

- **Supervising-lawyer review on every drafted document.** No drafted artifact reaches a client, a court, or opposing counsel without a supervising lawyer's explicit tap. The agent surfaces the draft into a review queue; it does not send. Ever.
- **Privilege-aware redaction by default.** The `privilege-aware-comms-log` skill writes redacted summaries into the Airtable matter record. The partner can un-redact when they need to; the agent never writes un-redacted content into any logged surface.
- **No autonomous client communication.** The agent does not send messages, draft replies, or generate communications that go to anyone outside the firm without a partner's tap. This is the constraint that makes **OpenClaw — our autonomous-agent tier — strictly off-limits for law firms.** Across PT, DE, and CH, the supervision rules are clear enough that I never quote OpenClaw to a law-firm prospect.

---

## v3 — Field Service Command (Home Services)

Third-designed vertical. Pitch deck, vertical checklist, capability map all shipped; the first signed pilot is the missing piece. The package targets plumbing, electrical, HVAC, cleaning, landscaping, locksmith, handyman, and painting operators — owner-operator businesses where the owner is often also the lead technician and the dispatcher is often the owner's partner.

### Operator cockpit

```text
                       Owner-operator (often also lead tech)
                                  │
                                  │  desktop in the office
                                  │  + Telegram bridge for the van
                                  ▼
                       ┌──────────────────────┐
                       │  Claude Cowork       │   ← single app at base
                       │  desktop             │     Telegram bridges to
                       └──────────┬───────────┘     the van for approvals
                                  │
                                  ▼
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
       ┌─────────────┐                   ┌──────────────────────┐
       │ Field       │                   │ Office               │   ← intake,
       │  surface    │                   │  surface             │     dispatch,
       │ (job assign │                   │                      │     invoice,
       │  drafts +   │                   │                      │     review
       │  on-site    │                   │                      │     cascade
       │  checklists)│                   │                      │
       └──────┬──────┘                   └──────────┬───────────┘
              │                                     │
              ▼                                     ▼
       ┌─────────────┐                   ┌──────────────────────┐
       │ WhatsApp    │                   │ Operations toolchain │
       │  Business   │                   │  Gmail · GCal ·      │
       │  inbound    │                   │  Drive · Airtable    │
       │  (n8n bridge)│                  │  (Jobs · Technicians │
       │  Mapping    │                   │  · Quotes · Reviews) │
       │  API briefs │                   │  GBP write · CRM ·   │
       │             │                   │  accounting (read)   │
       └──────┬──────┘                   └──────────┬───────────┘
              │                                     │
              └──────────────────┬──────────────────┘
                                 │ dispatch board + review cascade
                                 ▼
                      ┌──────────────────────┐
                      │ Dispatcher ⏸        │   ← HUMAN GATE 1
                      │  confirms every job  │     single-tap from
                      │  assignment via      │     the messaging app
                      │  Telegram before tech │
                      │  sees it             │
                      └──────────────────────┘
```

### Skill library

11 core + 7 vertical add-ons:

| Skill | Layer | What it does |
|---|---|---|
| `admin-inbox-triage` … `accounting-handoff-pack` | core | Same 11-skill core as the other verticals |
| `missed-call-capture` | vertical | n8n-bridged callback when a phone call goes to voicemail outside hours; drafts a WhatsApp callback in the owner's voice for the owner to send |
| `job-intake-dispatch` | vertical | Converts an inbound enquiry (form / WhatsApp / phone-log) into a structured Airtable Job record and drafts a dispatch assignment for the dispatcher to confirm |
| `route-brief` | vertical | Morning brief of the day's jobs by technician; suggests a route order; flags conflicts |
| `on-site-checklist` | vertical | Job-type-aware on-site checklist (HVAC start-up, plumbing tear-out, etc.) the technician opens on their phone |
| `review-cascade-field` | vertical | On `job_completed`, drafts a review-request the owner taps to send; cascade pauses if the previous request was unanswered |
| `gbp-field-post` | vertical | Drafts a GBP post showcasing a completed job (with photos the technician uploaded), tuned for the local-search audience |
| `reactivation-cadence` | vertical | Quarterly reactivation pass on customers who haven't booked in 12+ months; drafts the touch the owner taps to send |

### HITL gates

Three hard gates above the standard drafts-only floor:

- **Dispatcher confirmation on every job assignment.** No technician sees a job assignment that the dispatcher hasn't confirmed with a single tap. The system drafts; the dispatcher dispatches.
- **Owner approval on every quote sent.** The `sales-proposal` skill drafts the quote; the system holds it in Cowork until the owner taps to send. No autonomous quoting, even for jobs that look like template matches.
- **Review cascade fires only on `job_completed`.** The `review-cascade-field` skill is event-driven — it never asks for a review before the job is marked completed in Airtable, and it never fires twice on the same job. The owner taps to send.

---

## v4 — Command Center (the horizontal fallback)

Not a vertical. The horizontal package I sell when a prospect doesn't fit cleanly into the three verticals — the off-blueprint case. Currently running with one pilot (an operator who runs both a marketing agency and a small construction business in parallel, neither of which is in scope for the three vertical packages). The Command Center ships the 11 core skills, no vertical add-ons, and the same cockpit shape. Pricing is in band: €2,500–3,500 one-off, €1,499/mo managed, single device. **The discipline rule:** if a prospect *is* in scope for one of the three verticals (Home Services, Healthcare Clinics, Law Firms), the vertical wins. Command Center never gets quoted to undercut a vertical package.

---

## Enterprise topology · per-vertical detail

The full per-vertical reference architecture — every tier, every integration, every approval boundary. The cockpit diagrams above show the *shape*; this one shows what's actually wired in a Practice Command pod end-to-end. The Legal Efficiency Suite and Field Service Command topologies are structurally identical — same tiers, different MCP fleet and skill library composition per the tables above.

```text
==============================================================================
OPERATOR SURFACE
==============================================================================

+----------------------------------+  +----------------------------------+
| COWORK DESKTOP                   |  | TELEGRAM BRIDGE (optional)       |
+----------------------------------+  +----------------------------------+
|   Anthropic Claude               |  |   Webhook bridge via n8n for     |
|     Sonnet 4.6 for routine       |  |     in-the-field approvals       |
|     Opus 4.6 for heavy skills    |  |     (Field Service Command only) |
|   single window, single context  |  |                                  |
|   computer-use capable           |  |   Approval shape:                |
|   file system access             |  |     ok / reject reply trips      |
|   skill library auto-load        |  |     webhook into Cowork          |
|   memory directory loaded        |  |                                  |
+----------------------------------+  +----------------------------------+

                                |
                                v

==============================================================================
COWORK ORCHESTRATION
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| INTERACTIVE LANE     | | SCHEDULED LANE       | | MEMORY DIRECTORY     |
+----------------------+ +----------------------+ +----------------------+
|   operator at the    | |   Cowork-native      | |   memory/            |
|   keyboard           | |     scheduled tasks  | |     user_profile.md  |
|   chat-driven        | |   same Claude,       | |     brand_voice.md   |
|     invocation       | |     same skills,     | |     clients_top.md   |
|   skill picker       | |     same MCPs,       | |     rules.md         |
|     by description   | |     same memory      | |     vertical/        |
|   ad-hoc workflows   | |                      | |       <patients|     |
|                      | |   daily 09:00:       | |        matters|      |
|                      | |     inbox triage     | |        jobs>.md      |
|                      | |   weekly Mon 08:00:  | |   (per-vertical      |
|                      | |     owner dashboard  | |    extension)        |
|                      | |   monthly 1st 06:00: | |                      |
|                      | |     accounting prep  | |   memory updates     |
|                      | |   monthly 1st 07:00: | |     are explicit -   |
|                      | |     recall/deadline/ | |     skill writes,    |
|                      | |     reactivation per | |     never silent     |
|                      | |     vertical         | |                      |
+----------------------+ +----------------------+ +----------------------+

+----------------------+ +----------------------+ +----------------------+
| SKILL LIBRARY        | | HITL GATE MODEL      | | AUDIT TRAIL          |
+----------------------+ +----------------------+ +----------------------+
|   .claude/skills/    | |   per-skill          | |   every drafted      |
|   11 core skills     | |     # Don'ts /       | |     artifact lands   |
|     (admin / mktg /  | |     escalation       | |     on disk under    |
|      sales / acct)   | |     block            | |     content/drafts/  |
|   + vertical pack:   | |                      | |     accounting-prep/ |
|     Practice:    7   | |   plus vertical:     | |     admin/inbox/     |
|     Legal:       6   | |     Practice =       | |     matters/         |
|     Field Svc:   7   | |       patient-       | |     jobs/            |
|     Command Ctr: 0   | |       identifier     | |                      |
|                      | |       gate           | |   Cowork conversation|
|   each SKILL.md:     | |     Legal =          | |     log is the audit |
|     name + desc      | |       supervising-   | |     of every action  |
|     # Purpose        | |       lawyer review  | |                      |
|     # When to use    | |     Field Svc =      | |   every Cowork       |
|     # Inputs         | |       dispatcher     | |     scheduled-task   |
|     # Process        | |       confirmation   | |     run writes a     |
|     # Output         | |                      | |     dated artifact   |
|     # Don'ts /       | |   nothing fires      | |                      |
|       escalation     | |     without          | |                      |
|                      | |     owner tap        | |                      |
+----------------------+ +----------------------+ +----------------------+

                                |
                                v

==============================================================================
MCP FLEET · per vertical
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| CORE FLEET           | | VERTICAL FLEET       | | EXTERNAL BRIDGE      |
+----------------------+ +----------------------+ +----------------------+
|   Gmail (read/draft) | |   Practice Command:  | |   n8n (self-hosted   |
|     gmail-mcp        | |     PMS read API     | |     on OBLQAI VPS)   |
|   Google Calendar    | |     (Dentally /      | |                      |
|     gcal-mcp         | |      Clinicminds /   | |   used ONLY for:     |
|   Google Drive       | |      Doctolib /      | |     - WhatsApp       |
|     gdrive-mcp       | |      Doctoralia /    | |       Business       |
|   Google Business    | |      Optident)       | |       inbound        |
|     Profile          | |     consent-aware    | |       (no MCP yet)   |
|     gbp-mcp          | |     patient roster   | |     - tools with no  |
|   Airtable           | |   Legal Eff Suite:   | |       MCP server     |
|     airtable-mcp     | |     DMS read         | |       available      |
|   HubSpot (or other  | |     (NetDocuments /  | |       (some accting  |
|     CRM)             | |      iManage /       | |        editions)     |
|   Accounting (read)  | |      SharePoint /    | |     - external       |
|     Moloni / Sage /  | |      file system)    | |       deadman /      |
|     KMU per market   | |     conflict list    | |       heartbeat      |
|   Firebase MCP       | |     time export      | |                      |
|     (for OBLQAI-side | |   Field Svc Command: | |   Cowork scheduled   |
|      website +       | |     mapping API      | |     tasks own        |
|      hosting where   | |     briefs           | |     orchestration -  |
|      relevant)       | |     (no PMS - the    | |     n8n is the       |
|   Gemini Image MCP   | |      dispatch board  | |     bridge, not the  |
|     (internal -      | |      lives in        | |     orchestrator     |
|      gemini-image-   | |      Airtable)       | |                      |
|      mcp wrapper)    | |                      | |                      |
|   Veo3 MCP           | |                      | |                      |
|     (internal -      | |                      | |                      |
|      video gen on    | |                      | |                      |
|      demand)         | |                      | |                      |
+----------------------+ +----------------------+ +----------------------+

                                |
                       outbound API calls per
                       MCP tool invocation
                                v

==============================================================================
DATA + AUDIT PLANE
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| AIRTABLE BASE        | | DRAFTED ARTIFACTS    | | OBLQAI DASHBOARD     |
+----------------------+ +----------------------+ +----------------------+
|   per-engagement     | |   on operator's      | |   internal partner   |
|     workspace        | |     local disk under | |     dashboard at     |
|   tables vary by     | |     project folder   | |     jccosta94.       |
|     vertical:        | |   never auto-deleted | |     github.io/       |
|     Practice -       | |   owner reviews,     | |     oblqai-dashboard |
|       Patients,      | |     edits, sends     | |   read-only feed of  |
|       Bookings,      | |     (the act of      | |     CRM + costs      |
|       Recalls,       | |     sending updates  | |   refresh model:     |
|       Invoices,      | |     the artifact)    | |     snapshot.json    |
|       Consents       | |                      | |     pushed manually  |
|     Legal -          | |   monthly audit:     | |   never live-reads   |
|       Matters,       | |     count drafted    | |     a client's       |
|       Time,          | |     vs sent vs       | |     Airtable         |
|       Intake,        | |     archived per     | |                      |
|       Deadlines,     | |     skill            | |                      |
|       Comms Log      | |                      | |                      |
|     Field Svc -      | |                      | |                      |
|       Jobs,          | |                      | |                      |
|       Technicians,   | |                      | |                      |
|       Quotes,        | |                      | |                      |
|       Reviews        | |                      | |                      |
+----------------------+ +----------------------+ +----------------------+

==============================================================================
DEPLOYMENT BOUNDARIES
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| OPERATOR'S DEVICE    | | OBLQAI VPS           | | CUSTOMER-OWNED       |
+----------------------+ +----------------------+ +----------------------+
|   Cowork desktop     | |   n8n self-hosted    | |   PMS / DMS /        |
|   memory dir +       | |     (webhook bridge  | |     dispatch tools   |
|     skill library    | |     only)            | |     stay on the      |
|   project folder     | |   external deadman / | |     customer's       |
|     under the        | |     heartbeat        | |     subscription     |
|     operator's       | |   no per-customer    | |   accounting tool    |
|     Documents/       | |     state stored     | |     stays on the     |
|   local Airtable     | |     here (Cowork +   | |     accountant's     |
|     credentials      | |     Airtable own     | |     license          |
|     (.env, chmod 600)| |     that)            | |   data residency     |
|                      | |                      | |     respected per    |
|                      | |                      | |     market (PT / DE  |
|                      | |                      | |     /CH per RGPD /  |
|                      | |                      | |     FADP)            |
+----------------------+ +----------------------+ +----------------------+
```

**Three things to highlight from this layout:**

**Cowork is the only orchestrator. n8n is the webhook bridge.** This is the discipline rule I treat as non-negotiable in every vertical: the recurring digests, the daily / weekly / monthly schedule, the skill-picker logic — all of it runs inside Cowork's native scheduled-tasks layer, not in n8n. n8n only earns its place when the work *can't* happen inside Cowork: a public inbound webhook (WhatsApp Business has no MCP yet, so its inbound has to land at an HTTPS endpoint), a tool with no MCP server at all (some regional accounting editions), or an external deadman / heartbeat (so the operator finds out the connector died before they discover it the hard way). One n8n flow per gap, not a workflow tree.

**The customer keeps their tools.** Every vertical's MCP fleet reads from systems the customer already owns (PMS, DMS, CRM, accounting) — those subscriptions stay on the customer's bill. The integration goes one way: Cowork reads, the customer's tools remain authoritative. I never replicate the patient roster into a parallel store the customer can't reach, and I never put the firm's matter list into an OBLQAI-controlled database. Airtable is the only OBLQAI-introduced data store, and it's the operator's workspace, not a backend the customer has to trust.

**The data-residency map is per-market, not per-vertical.** PT runs on Claude API + Airtable (cloud) because the legal posture is settled at that level. DE adds an AVV/DPA addendum to every contract before any data flows. CH gets the option of a local-inference setup (Ollama + Qwen3 32B on a customer-owned box) for healthcare and legal engagements where the FADP posture for special-category data demands it. The vertical determines whether the question gets asked; the market determines what the answer is.

---

## Architecture diagrams

All diagrams in this repo live inline in this README as text-style ASCII inside fenced code blocks — readable on GitHub, version-controlled, diff-able, no rendering tooling required. Anchor links below for the operator-cockpit and enterprise-topology diagrams.

- **Practice Command cockpit** — operator → Cowork → patient-facing surface · operations surface → owner-eyes-only approval. See [Practice Command — Operator cockpit](#operator-cockpit) above.
- **Legal Efficiency Suite cockpit** — operator → Cowork → matter surface · practice surface → supervising-lawyer review queue. See [Legal Efficiency Suite — Operator cockpit](#operator-cockpit-1) above.
- **Field Service Command cockpit** — operator → Cowork + Telegram bridge → field surface · office surface → dispatcher confirmation. See [Field Service Command — Operator cockpit](#operator-cockpit-2) above.
- **Enterprise topology · per-vertical detail** — full layered view: operator surface · Cowork orchestration · skill library · MCP fleet · data + audit plane · deployment boundaries. See [Enterprise topology · per-vertical detail](#enterprise-topology--per-vertical-detail) above.

---

## What I designed vs what the platforms provide

| Concern | Platform provides | What I designed on top |
|---|---|---|
| **Operator cockpit** | [Claude Cowork](https://claude.com/cowork) — desktop AI app, Anthropic Claude with MCP-tool access, file-system access, native scheduled tasks, computer-use | Configured Cowork as the *only* management surface per engagement — authored the `CLAUDE.md` operational-context template, the memory-directory seed pattern (`user_profile.md` / `brand_voice.md` / `clients_top.md` / `rules.md` / vertical-specific files), the project folder shape every engagement adopts |
| **Reasoning model** | [Anthropic Claude](https://www.anthropic.com/claude) (Sonnet 4.6, Opus 4.6) | Per-skill model routing — Sonnet for the routine skills (inbox triage, meeting prep, dashboard), Opus for the heavier reasoning skills (matter intake with conflict, deadline-diary audit, accounting handoff pack) |
| **Skill library** | [Cowork skill format](https://claude.com/cowork) — `SKILL.md` per skill, frontmatter `name` + `description`, agent picks by description | Authored the **11 core skills** (`admin-inbox-triage`, `admin-meeting-prep`, `admin-owner-dashboard`, `marketing-content`, `marketing-calendar`, `marketing-analytics-summary`, `sales-followup`, `sales-proposal`, `sales-pipeline-review`, `accounting-prep-monthly`, `accounting-handoff-pack`), the canonical 7-section structure (Purpose / When to use / Inputs / Process / Output / Don'ts / escalation), the vertical-specific add-on libraries (7 Practice, 6 Legal, 7 Field Service), the skill-naming-is-API-design discipline |
| **MCP servers** | [Model Context Protocol](https://modelcontextprotocol.io) — Anthropic-published spec, growing public registry of community + first-party MCP servers | Per-vertical MCP fleet specification (which servers each vertical needs and what scope to grant), the read-only-by-default rule for accounting and DMS, the Hugo-style env-var discipline pattern (`OBLQAI_*` prefix everywhere, never SDK-default pickup), the OBLQAI-internal MCP wrappers I built where the public registry didn't have what I needed (`gemini-image-mcp`, `veo3-mcp`, the OBLQAI Gmail API integration) |
| **Scheduled work** | [Cowork native scheduled tasks](https://claude.com/cowork) — recurring tasks the operator configures once and forgets about | Configured the recurring cadence per vertical (daily 09:00 inbox triage; weekly Mon 08:00 owner dashboard; monthly 1st 06:00 accounting prep; monthly 1st 07:00 the vertical-specific recall / deadline / reactivation pass), wired every scheduled task through the same skill library and memory directory the operator uses interactively |
| **Webhook bridge** | [n8n](https://n8n.io) — self-hosted workflow tool on the OBLQAI VPS | One flow per integration gap — WhatsApp Business inbound (the big one), no-MCP-tool accounting bridges, external deadman / heartbeat monitor. The discipline: n8n is the bridge, not the orchestrator. Cowork's scheduled tasks own the recurring work. |
| **HITL gate model** | Neither Cowork nor MCP ships an approval-gate model | Three-tier autonomy taxonomy (drafts-only baseline · vertical-specific gate · scheduled-task scoped permissions), enforced through the per-skill `# Don'ts / escalation` block in every `SKILL.md`, plus vertical-specific patterns (patient-identifier gate for Practice, supervising-lawyer review queue for Legal, dispatcher-confirmation-via-Telegram for Field Service) |
| **CRM / operations data** | [Airtable](https://airtable.com) — per-engagement workspace | Per-vertical Airtable schema (Practice: Patients · Bookings · Recalls · Invoices · Consents; Legal: Matters · Time · Intake · Deadlines · Comms Log; Field Service: Jobs · Technicians · Quotes · Reviews) — schemas authored from the vertical checklists, not invented per client |
| **Compliance posture** | Anthropic Claude and Cowork have their own enterprise privacy and security posture; the operator runs on their own subscription | Per-market data-residency pattern (PT default cloud; DE add AVV/DPA addendum; CH optional local-inference setup with Ollama + Qwen3 32B for healthcare and legal where FADP for special-category data demands it), vertical-specific compliance language in every proposal (RGPD + ERS for Practice, supervision + privilege for Legal, RGPD + sectoral for Field Service), the OpenClaw-never-for-law-firms hard rule |
| **Partner-facing dashboard** | Neither platform ships one | [OBLQAI dashboard](https://jccosta94.github.io/oblqai-dashboard/) — Vite + React + Tailwind on GitHub Pages, read-only view of the CRM + costs, snapshot.json pushed manually from a Cowork session whenever the partner needs a refresh (no Airtable PAT in CI, no live reads) |

The pattern: **I don't reinvent the cockpit. I do design the configuration, the skill library, the MCP fleet, and the gate model that make the cockpit useful per vertical.**

---

## Stack

| Layer | Component |
|---|---|
| **Operator cockpit** | [Claude Cowork](https://claude.com/cowork) desktop app — Anthropic Claude (Sonnet 4.6 routine · Opus 4.6 heavy reasoning) · native scheduled tasks · MCP-tool support · file-system access · computer-use |
| **Skill library** | 11 core skills (`admin-*`, `marketing-*`, `sales-*`, `accounting-*`) + per-vertical extensions (7 Practice · 6 Legal · 7 Field Service) — each `SKILL.md` in the canonical 7-section structure |
| **MCP fleet — core** | [Gmail MCP](https://github.com/modelcontextprotocol/servers) · [Google Calendar MCP](https://github.com/modelcontextprotocol/servers) · [Google Drive MCP](https://github.com/modelcontextprotocol/servers) · [Google Business Profile MCP](https://github.com/modelcontextprotocol/servers) · [Airtable MCP](https://github.com/modelcontextprotocol/servers) · HubSpot MCP (or per-engagement CRM) · regional accounting MCP (read-only — Moloni / Sage / KMU per market) · [Firebase MCP](https://firebase.google.com) for the OBLQAI-side hosting · OBLQAI-internal `gemini-image-mcp` (Nano Banana wrapper) · OBLQAI-internal `veo3-mcp` (Veo 3.1 wrapper) · OBLQAI-internal Gmail API integration |
| **MCP fleet — Practice Command add** | PMS read API (Dentally / Clinicminds / Doctolib / Doctoralia / Optident per market) · consent-aware patient roster reads |
| **MCP fleet — Legal Efficiency Suite add** | DMS read (NetDocuments / iManage / SharePoint / file system) · conflict-list source · time-tracking export |
| **MCP fleet — Field Service Command add** | Mapping API briefs · WhatsApp Business inbound via n8n bridge · Airtable dispatch board (canonical job state) |
| **Webhook bridge** | [n8n](https://n8n.io) self-hosted on the OBLQAI VPS — WhatsApp Business inbound only, no-MCP accounting bridges, external deadman / heartbeat monitor. Bridge, not orchestrator. |
| **Operations data store** | [Airtable](https://airtable.com) per-engagement workspace — per-vertical schema authored from the vertical checklists |
| **Local inference (data-residency option)** | [Ollama](https://ollama.com) + [Qwen3 32B](https://qwenlm.github.io/) — optional add-on for CH healthcare and legal engagements where FADP for special-category data demands it |
| **Partner dashboard** | [OBLQAI dashboard](https://jccosta94.github.io/oblqai-dashboard/) — Vite + React + Tailwind on GitHub Pages, read-only feed of CRM + costs, `snapshot.json` pushed manually |
| **OBLQAI website + brand surface** | [Firebase Hosting](https://firebase.google.com) + React + Vite + Tailwind v4 · `oblqai.com` · CTA "Request Free Consultation" · `joao.costa@oblqai.com` |
| **Brand & visual identity** | Editorial monochrome with single copper accent (`C27C4E`) · charcoal `3C3C3C` + cream `E7E3DC` neutrals + gray ramp · Georgia headers / Calibri body for decks · Inter for the wordmark `OBLQ · AI` |

---

## Deeper reading

### Related case studies in this portfolio
- [`openclaw-hermes-evolution`](../openclaw-hermes-evolution/) — the multi-agent build system that ships Psinest. Same Claude doing different work, but autonomous-agent territory — explicitly **not** what Claude Cowork engagements offer to law firms, and the upgrade path home-services and healthcare clients can graduate to when they're ready for the OpenClaw tier (see [oblqai.com](https://oblqai.com) for the two-tier positioning).
- [`hugo`](../hugo/) — productized AI marketing employee where each tenant runs on the customer's own ChatGPT Plus subscription. The Pattern C insight there (single-agent + MCP tools, summarize the return) is the philosophical cousin of the skill-naming discipline used here: deterministic work goes in MCP-served Python, not in the agent's prompt.
- [`psinest`](../psinest/) — live RGPD + ERS healthcare CRM. The compliance architecture for Practice Command engagements borrows directly from the tenancy and access-enforcement patterns shipped in Psinest (`RoleScope.cs`-level centralised tenancy is the model).
- [`skoda-ai-ops`](../skoda-ai-ops/) — content-operations architecture for a German automotive referral brand, also run from Cowork. Different vertical, same "Cowork is the cockpit *and* the scheduler" architectural bet.

### External
- [Claude Cowork](https://claude.com/cowork) — Anthropic's desktop AI app
- [Model Context Protocol specification](https://modelcontextprotocol.io)
- [Anthropic Claude](https://www.anthropic.com/claude) — Sonnet 4.6 + Opus 4.6, the reasoning models powering every Cowork session
- [n8n](https://n8n.io) — self-hosted workflow automation, used as the webhook bridge tier
- [Airtable](https://airtable.com) — operations data store per engagement
- [Ollama](https://ollama.com) + [Qwen3 32B](https://qwenlm.github.io/) — local-inference option for FADP-sensitive engagements

### Vertical references (the regulatory + professional ground these workflows have to respect)
- **Practice Command** — Portuguese ERS clinical-advertising rules · EU RGPD special-category-data handling · German HWG advertising rules for medical services · Swiss FADP rules for healthcare data residency
- **Legal Efficiency Suite** — supervision and privilege rules across PT / DE / CH bar associations; AML / KYC obligations for real estate, corporate, and immigration matters across the same three markets
- **Field Service Command** — Google Business Profile policy on review solicitation · WhatsApp Business policy on initiated messaging · regional consumer-protection rules on quotes and pricing language

---

## Status

- ✅ **Practice Command** — LIVE. First pilot operational at a Portuguese aesthetic clinic. Skill library shipped (11 core + 7 vertical). Airtable schema in production. Daily / weekly / monthly scheduled tasks running. Owner-eyes-only approval pattern enforced for everything touching patient identifiers. Pricing finalised across all three markets (PT €8k/€950 · DE €12k/€1,425 · CH CHF 16k/CHF 1,900). Pitch deck (`Healthcare-Clinics-Practice-Command.pptx`), vertical checklist (`Discovery Docs/Vertical Checklists/Healthcare-Clinics.md`), and implementation blueprint all shipped.
- ✅ **Legal Efficiency Suite** — LIVE. Two pilots: one Portuguese boutique partnership, one cross-border European solo practitioner. Skill library shipped (11 core + 6 vertical). Supervising-lawyer review queue enforced for every drafted document. Privilege-aware redaction by default in the comms log. OpenClaw-never-for-law-firms rule documented and enforced. Pitch decks (`Comercial/Demos/Law-Firms-Legal-Efficiency.pptx`, `Comercial/Demos/Law-Firms-Legal-Efficiency-PT.pptx`) and vertical checklist (`Discovery Docs/Vertical Checklists/Law-Firms.md`) all shipped. Per-vertical pricing not yet finalised — defer to Joao before quoting.
- 🚧 **Field Service Command** — WIP. Full design complete: pitch deck (`Home-Services-Field-Command.pptx`), vertical checklist (`Discovery Docs/Vertical Checklists/Home-Services.md`), capability map, MCP fleet specification, and skill library design (11 core + 7 vertical). Awaiting first signed pilot to validate the Telegram-bridge dispatcher-approval pattern in the field. Per-vertical pricing not yet finalised — defer to Joao before quoting.
- ✅ **Command Center (horizontal fallback)** — LIVE. One pilot running off-blueprint (an agency + small-construction operator). Discipline rule documented: never quoted to undercut a vertical package. Pricing in band: €2,500–3,500 one-off · €1,499/mo managed.
- ✅ **Diagrams** — all architecture diagrams shipped as inline text-style ASCII in fenced code blocks (Practice cockpit · Legal cockpit · Field Service cockpit · enterprise per-vertical topology).
- 🚧 **Per-vertical runbooks** — Command Center runbook (`07-Claude-Cowork-Command-Center-Runbook.md`) is the template. Practice / Legal / Field Service runbook clones with vertical-specific deployment steps are next.
- 🚧 **Per-vertical skill repos** — `cowork-skills/core/` is fully populated. `cowork-skills/practice-command/`, `cowork-skills/legal-efficiency/`, `cowork-skills/field-service/` are scaffolded folders awaiting the vertical-specific `SKILL.md` files to land from in-engagement work.

The sequencing is deliberate: Practice Command first because the discipline imposed by clinical-data handling forces the design to be tight enough that the other verticals inherit good defaults; Legal Efficiency Suite second because the supervising-lawyer gate is structurally compatible with the Practice owner-eyes-only gate (the same pattern, with a different label on the approver); Field Service Command third because the field-first dispatcher-via-Telegram pattern is the one that needed the most live-pilot validation before going to market. Command Center sits below the line as the fallback for prospects that don't fit, never as the entry point for prospects that do.

---

## Acknowledgements

- **[Claude Cowork](https://claude.com/cowork)** (Anthropic) — the desktop AI app that makes all three vertical templates possible. Same Cowork install, same Anthropic Claude, same MCP-tool support, same native scheduled tasks. The fact that Cowork is general enough to be a clinic operator's cockpit *and* a law-firm associate's cockpit *and* a home-services dispatcher's cockpit, without changing the app, is the load-bearing platform property the entire vertical-templating method depends on.
- **[Model Context Protocol](https://modelcontextprotocol.io)** (Anthropic, MIT) — the integration plane that lets the vertical-specific MCP fleet plug in cleanly per engagement. The "naming a skill is API design" insight applies one layer up to MCP server naming as well: stable, mutually exclusive, descriptive enough that Cowork's auto-selection picks the right one.
- **Adopted public MCP servers** — [`gmail-mcp`](https://github.com/modelcontextprotocol/servers), [`gcal-mcp`](https://github.com/modelcontextprotocol/servers), [`gdrive-mcp`](https://github.com/modelcontextprotocol/servers), [`gbp-mcp`](https://github.com/modelcontextprotocol/servers), [`airtable-mcp`](https://github.com/modelcontextprotocol/servers), HubSpot MCP, [Firebase MCP](https://firebase.google.com), regional accounting MCPs (Moloni / Sage / KMU per market). These are the heavy lifting on the integration side — I configure them per engagement, I don't build them.
- **Custom MCP servers I built for the practice** — `gemini-image-mcp` (Nano Banana wrapper, for image generation per content workflows), `veo3-mcp` (Veo 3.1 wrapper, for video on demand), the OBLQAI Gmail API integration (gmail-api). Small, single-purpose, no orchestration — they expose one capability cleanly so Cowork can call them like any other tool.
- **[Anthropic Claude](https://www.anthropic.com/claude)** (Sonnet 4.6 routine · Opus 4.6 heavy reasoning) — the reasoning model powering every Cowork session, interactive and scheduled, across every vertical.
- **[n8n](https://n8n.io)** (Apache-2.0) — self-hosted workflow automation, used as the webhook bridge for WhatsApp Business inbound and the no-MCP-tool accounting gaps. Discipline rule: bridge, not orchestrator.
- **[Airtable](https://airtable.com)** — operations data store per engagement.
- **[Ollama](https://ollama.com) + [Qwen3 32B](https://qwenlm.github.io/)** — local inference option for FADP-sensitive Swiss engagements in healthcare and legal verticals.

**What's mine:** the vertical-templating method (horizontal cockpit + vertical configuration), the per-vertical skill library structure (11 core + N vertical, canonical 7-section `SKILL.md`), the HITL gate patterns per vertical (patient-identifier gate for Practice · supervising-lawyer review queue for Legal · dispatcher-confirmation-via-Telegram for Field Service · drafts-only floor everywhere), the approval-flow design (`# Don'ts / escalation` block in every skill, no autonomous sending or publishing, ever), the per-vertical MCP fleet specification, the per-vertical Airtable schemas authored from the discovery checklists, the per-market data-residency pattern (PT cloud default · DE AVV/DPA addendum · CH local-inference option), the OpenClaw-never-for-law-firms rule, and the sequencing argument (Practice first → Legal second → Field Service third → Command Center as fallback only).

The vertical-templating pattern — build the horizontal platform first, then verticalise as configuration on top — is borrowed from how every successful productized service firm I've watched operates analog. A managed-IT shop doesn't write a different RMM for every customer; it configures the same RMM with different alerting policies per customer. A managed-accounting practice doesn't author a new general ledger per client; it configures QuickBooks differently per client. **AI consulting works best when it mirrors the practices that already work in productized professional services.**

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com) · 🌐 [oblqai.com](https://oblqai.com) · 🐙 [github.com/jccosta94](https://github.com/jccosta94)
