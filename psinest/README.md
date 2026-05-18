# Psinest — Healthcare CRM with AI Augmentation Roadmap

**Live healthcare CRM** at [psinest.duckdns.org](https://psinest.duckdns.org) — a Portuguese-language platform for psychologists, clinics, and patients. Built end-to-end (founder + small team) and operated in production for real users. This case study documents the **current architecture** (live today), the **AI augmentation roadmap** (designed, partially in flight), and the **compliance-aware design decisions** that shape how AI lands on top of a healthcare data plane.

🌐 [psinest.duckdns.org](https://psinest.duckdns.org) · 🇵🇹 pt-PT first · 👥 ClinicOwner / Psychologist / Patient / Platform Admin roles · 🏥 RGPD + ERS-aware

> **Note on the URL.** `psinest.duckdns.org` is a **placeholder domain for the pilot phase**. The current footprint is **10 psychologists + 1 clinic** running on this domain as the live test run. Once the pilot stabilises and feature scope is locked, the platform migrates to a permanent domain with a managed cert + production CDN. The architecture documented here is what the pilot runs on; the migration path is short (DNS + cert swap, no architectural change required).

---

## What Psinest is

A CRM tailored for the Portuguese mental-health market:

- **Psychologists** manage their caseload, run sessions, upload patient documents, draft homework
- **Clinics** (ClinicOwner role) manage psychologist rosters, partnerships with independent psys, view aggregate reports — never see patient data directly
- **Patients** see their own session history, upload their own documents, receive psy-attached homework, book sessions
- **Platform Admin** approves new psychologist registrations (verifies OPP / cédula credentials)

Live and operating in pilot today. Free to clinics during pilot; will become paid once feature scope stabilises.

---

## Screenshots

Captured from the live product at [psinest.duckdns.org](https://psinest.duckdns.org) using the four permanent demo accounts (`clinica.teste`, `dr.teste`, `patient.teste`, `patient.preonboard.teste`). All data shown is synthetic — these are fixtures maintained on production specifically for spec/regression and portfolio use.

| | |
|---|---|
| ![Landing + sign-in](./screenshots/01-landing.png) | ![Patient space — sessions](./screenshots/02-patient-sessions.png) |
| **Landing + sign-in** — public marketing landing page with role-specific routing after authentication via Firebase. | **Patient space — sessions** — patient's own session history, with psy-attached homework on `/patient/dynamics/:id` (read-only) and self-upload documents at `/patient/documents`. |
| ![Psychologist — patient detail](./screenshots/03-psy-patient-detail.png) | ![Psychologist — session notes](./screenshots/04-psy-session-notes.png) |
| **Psychologist — patient detail** — caseload view with patient-attached dynamics, document timeline, partnership status. Psy sees only patients in their partnered clinics. | **Psychologist — session notes** — note-taking surface with future-state hooks for AI-augmented transcription + SOAP-note drafting (see roadmap below). |
| ![Clinic Owner — dashboard](./screenshots/05-owner-dashboard.png) | ![Clinic Owner — reports + analytics](./screenshots/06-owner-reports.png) |
| **Clinic Owner — dashboard** — psy roster, partnership status, aggregate metrics. Owner never sees patient data; document access blocked entirely at `RoleScope` layer. | **Clinic Owner — reports + analytics** — Recharts-driven cancellation + turnover analytics with per-clinic threshold (1–5 sessions, default 3) and scope selector. |
| ![Admin — psy verification queue](./screenshots/07-admin-verification.png) | ![Document upload + permissions](./screenshots/08-documents.png) |
| **Admin — psy verification queue** — Platform Admin reviews OPP / cédula credentials before activating new psychologist accounts. ERS compliance gate. | **Document upload + permissions** — `IDocumentStore` → Backblaze B2 storage with consent-aware permissions (patient self-upload vs psy-attached docs vs ClinicOwner blocked). |

---

## Current-state architecture

The full enterprise-level view, organised by tier — network edge, application, data, storage/external services, CI/CD.

```
┌────────────────────────────────────────────────────────────────────────────┐
│ HOSTINGER VPS · 72.62.2.211 · Ubuntu 24.04 · 1 vCPU / 3.8 GB                │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│ ┌─ NETWORK / SECURITY EDGE ────────────────────────────────────────────────┐│
│ │ Internet                                                                  ││
│ │   ▼                                                                       ││
│ │ DNS: psinest.duckdns.org (DuckDNS dynamic DNS)                           ││
│ │   ▼                                                                       ││
│ │ nginx 1.24 (ports 80, 443) · TLS via Let's Encrypt · HTTP→HTTPS redirect ││
│ │   ▼ static SPA bundle ←─/                                                ││
│ │   ▼ reverse-proxy ←─/api → 127.0.0.1:5000                               ││
│ └───────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│ ┌─ APPLICATION TIER ───────────────────────────────────────────────────────┐│
│ │ React 19 SPA (served as static bundle by nginx)                           ││
│ │   · TypeScript · Vite 8 · Tailwind 4 · Recharts                           ││
│ │   · i18n: pt-PT default, en fallback (clinical glossary docs/pt-pt-       ││
│ │     clinical-glossary.md is the canonical source)                         ││
│ │   · Firebase Auth client SDK (browser-side sign-in)                       ││
│ │   · Sentry SPA SDK (env-driven DSN, sendDefaultPii=false)                 ││
│ │   ▼ calls /api over HTTPS · attaches Firebase JWT                         ││
│ │ Kestrel API (5000, localhost) · ASP.NET Core 8 + EF Core 8                ││
│ │   · Firebase Admin SDK (server-side JWT verify)                           ││
│ │   · RoleScope.cs (centralised JWT-derived tenancy)                        ││
│ │   · IDocumentStore abstraction (B2 ↔ filesystem swap)                     ││
│ │   · Controllers: /users  /clinics  /psychologists  /patients              ││
│ │                  /sessions  /documents  /reports  /partnerships           ││
│ │                  /onboarding  /admin                                      ││
│ │   · Sentry .NET SDK (server-side exception capture)                       ││
│ └───────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│ ┌─ DATA TIER ──────────────────────────────────────────────────────────────┐│
│ │ PostgreSQL 16 (5432, localhost) · db: psinest · peer auth (no password)  ││
│ │   Core tables:                                                            ││
│ │     users · clinics · psychologists · patients · sessions                 ││
│ │     documents · partnerships · cancellations · onboarding_state · audit   ││
│ │   Schema convention: PascalCase quoted columns ("Id", "FirebaseUid")     ││
│ │                      lowercase unquoted tables (users, patients)          ││
│ │   ⤷ pg-dump cron · 17 */6 * * * → Backblaze B2 (psinest-backups)        ││
│ │     4 backups/day · retention managed via B2 lifecycle policy            ││
│ └───────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│ ┌─ STORAGE / EXTERNAL SERVICES ─────────────────────────────────────────────┐│
│ │ Backblaze B2 · S3-compatible · two buckets:                               ││
│ │   psinest-uploads  · document storage · accessed via AWS S3 .NET SDK     ││
│ │   psinest-backups  · nightly pg-dump archives                            ││
│ │ Firebase Auth · project psinestv2                                        ││
│ │   · JWT issuance (browser-side sign-in)                                  ││
│ │   · JWT verification (server-side via Admin SDK)                         ││
│ │   · Email + password auth · password reset                               ││
│ │ Sentry · org psinest · 2 projects                                         ││
│ │   · dotnet-aspnetcore (API errors)                                       ││
│ │   · psinest-app (SPA errors)                                             ││
│ │ DuckDNS · dynamic DNS for psinest.duckdns.org                            ││
│ └───────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│ ┌─ CI/CD ─────────────────────────────────────────────────────────────────┐│
│ │ GitHub Actions · workflow_dispatch ONLY (no auto-deploy on merge)         ││
│ │   Repos: jccosta94/psinest-app + jccosta94/psinest-api                    ││
│ │   Lint + build are the only merge gates · e2e is non-blocking smoke      ││
│ │   Cross-repo deploy order: API first → 60s wait → App second              ││
│ │   ⤷ SSH → VPS · systemctl restart psinest-api · rsync /var/www          ││
│ └───────────────────────────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Capability domains

| Domain | What it does | Who has access |
|---|---|---|
| **Sessions** | Schedule, take notes, mark cancellations, track turnover | ClinicOwner (read) · Psychologist (full) · Patient (own only) |
| **Patients** | Onboarding wizard, patient space, document upload | Psychologist (assigned) · Patient (own) |
| **Psychologists** | Registration with OPP/cédula gate, verification by Platform Admin, patient-attached dynamics | Self · ClinicOwner (partnered) · Admin (verification queue) |
| **Clinics** | Owner dashboards, psy roster, clinic ↔ psy partnerships | ClinicOwner |
| **Documents** | Upload/store via `IDocumentStore` → Backblaze B2 with consent-aware permissions | Psychologist · Patient (own) · ClinicOwner blocked entirely |
| **Reports** | Recharts viz, scope selector, cancellation + turnover analytics, per-clinic threshold (1–5 sessions, default 3) | ClinicOwner · Psychologist (own caseload scope) |
| **Auth / Users** | Firebase Auth integration, role assignment on register, password reset | Everyone (sign-in flow) · Admin (role management) |
| **Admin / Ops** | Psychologist verification approval queue, i18n master surfaces, model + cost dashboards (future) | Platform Admin only |

---

## Security architecture — Role-scoped access via `RoleScope.cs`

The single most important architectural decision: **the server never trusts client-supplied filters** on list endpoints. Tenancy is derived entirely from the JWT and applied at the EF Core query layer. `RoleScope.cs` is where this enforcement lives.

```
                  Authenticated request (JWT from Firebase)
                              │
                              ▼
                  ┌──────────────────────────────┐
                  │ ASP.NET middleware            │
                  │   · verifies JWT signature    │
                  │   · loads user row from DB    │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ RoleScope.cs                  │
                  │  · reads user.Role            │
                  │  · resolves tenant identity   │
                  │  · resolves URL claims        │
                  │  · prevents privilege climb   │
                  └──────────────┬───────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
         ClinicOwner        Psychologist          Patient
              │                  │                  │
              ▼                  ▼                  ▼
       /owner/*              /psy/*               /patient/*
       reports + roster      patient space        own data only
       partnerships          + sessions           psy-attached
       NEVER documents       + documents          read-only
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ Controller-level guard        │
                  │  [Authorize(Roles=...)]       │
                  └──────────────┬───────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │ EF Core query                 │
                  │  .Where(x => x.TenantId       │
                  │    == currentTenantId)        │
                  │  · scope applied at query     │
                  │    construction time, not     │
                  │    in client filters          │
                  └──────────────────────────────┘
```

**The patterns this prevents:**
- ClinicOwner accidentally reading patient documents — blocked at `RoleScope` before the controller runs
- Patient querying another patient's sessions via crafted `?patientId=N` — server derives `patientId` from JWT, ignores query string
- Psychologist accessing partnered clinic's patients without partnership row — partnership existence is verified server-side

**Important convention:** user IDs are NOT used as entity IDs. `clinica.teste`'s `user.uid` is not her clinic-entity ID. Cross-referencing requires explicit joins via `RoleScope`-aware queries.

---

## Document permission model

Healthcare documents are the most sensitive surface; permissions are intricate.

```
                       Document access request
                                  │
                                  ▼
                  ┌──────────────────────────────┐
                  │ Document permission resolver  │
                  └─────────────┬────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
         upload path                       read / delete path
              │                                   │
   ┌──────────┴──────────┐              ┌────────┴─────────┐
   │                     │              │                  │
   /patient/documents    /psy/patient/N/  read           delete
   (Patient self-        documents       │               │
    upload at this       (Psy uploads    │            ┌──┴────────────┐
    surface only)        attached docs)  │            │               │
   ─────────────────     ──────────────  │       canDelete         blocked
   Owner: Patient        Owner: Psy      │       (Psy-in-          (ClinicOwner
   Always allowed                        │        patient-space    entirely)
   for self              Verified via    │        OR Patient-on-
                         partnership +   │        own-doc)
                         consent row     │
                                         │
                                         ▼
                                Read scope check:
                                  · Patient: own docs only
                                  · Psy: in partner clinic
                                    AND assigned patient
                                  · ClinicOwner: BLOCKED
                                    (never reads any document)
```

**Rules in plain English:**
- **Patient** can read, upload, and delete their own documents on `/patient/documents`
- **Patient** is read-only on `/patient/dynamics/:id` (psy-attached homework — patient cannot modify)
- **Psychologist** can upload patient documents at `/psy/patient/N/dynamics` after partnership + consent rows exist
- **Psychologist** in patient space can delete anything in that space (their work + the patient's contributions)
- **ClinicOwner** is **blocked entirely from documents** — they don't see any patient data, full stop

---

## Tech stack

| Tier | Technology · Version |
|---|---|
| **Frontend** | React 19 · TypeScript · Vite 8 · Tailwind 4 · Recharts · i18n (pt-PT default) |
| **Backend** | ASP.NET Core 8 · EF Core 8 · Kestrel · Firebase Admin SDK |
| **Database** | PostgreSQL 16 · `psinest` db · peer-auth |
| **Document store** | Backblaze B2 (S3-compatible) · psinest-uploads · accessed via AWS S3 .NET SDK · `IDocumentStore` abstraction |
| **Auth** | Firebase Auth · project `psinestv2` · email+password · JWT |
| **Observability** | Sentry · org `psinest` · 2 projects (dotnet-aspnetcore + psinest-app) |
| **Web server** | nginx 1.24 · Let's Encrypt TLS · HTTP→HTTPS redirect |
| **DNS** | DuckDNS (dynamic DNS) · `psinest.duckdns.org` |
| **Hosting** | Hostinger VPS · Ubuntu 24.04 · 1 vCPU · 3.8 GB RAM |
| **Backups** | `pg_dump` cron · 4×/day · → Backblaze B2 psinest-backups bucket |
| **CI/CD** | GitHub Actions · `workflow_dispatch` (manual) · lint + build merge gates |
| **Built by** | [OpenClaw 7-agent team](../openclaw-hermes-evolution/) → migrated to [Hermes Agent single dispatcher](../openclaw-hermes-evolution/) |

---

## Compliance + regulatory architecture

Mental health data in Portugal is regulated by **RGPD** (EU GDPR national implementation) and **ERS** (Entidade Reguladora da Saúde — Portuguese health regulator). Psinest's architecture has to anticipate these from day one.

**Compliance-aware design decisions baked into the current platform:**

| Concern | Implementation |
|---|---|
| **RGPD — data minimisation** | Each role only sees data it strictly needs (RoleScope.cs enforced) · ClinicOwner blocked from documents entirely |
| **RGPD — right to erasure** | `Patient.delete` cascades through sessions, documents, partnerships · documented as part of deletion playbook |
| **RGPD — data residency** | All persistence (Postgres, B2 uploads, B2 backups) is EU-located · Hostinger VPS in EU · B2 EU region selected |
| **RGPD — audit trail** | Every role-affecting action logged · admin verification flow has audit row per approval |
| **ERS — psychologist credential gate** | Registration requires OPP number / cédula profissional · verified by Platform Admin before activation · grandfathering rule for existing users (default approved + audit-trail comment) |
| **Consent-aware document permissions** | Document operations gated by partnership row + consent row before psy can touch patient data |
| **Backup + restore drill** | pg-dump → B2 nightly · restore drill documented as a required ops practice |

**What's NOT yet in place** (called out honestly):
- Per-patient AI opt-in (only meaningful once AI capabilities land — see roadmap below)
- PII redaction proxy (required before any external-LLM call processes patient data)
- AI Policy Manager centerpiece (consent, redaction, audit gates for AI features specifically)

---

## AI augmentation roadmap (future state)

The architecture's current state covers the data plane. The future-state work is **eight AI capability domains** layered on top, each gated by consent, PII redaction, and audit. Designed but not all shipped.

```
CURRENT STATE                                FUTURE STATE
══════════════════════                       ═══════════════════════

┌──────────────────────┐                     ┌──────────────────────┐
│ Application platform  │                     │ Application platform  │
│  Sessions             │                     │ + AI capability       │
│  Documents            │                     │   augmentation        │
│  Patients             │                     │   ┌─ Transcription &  │
│  Psychologists        │ ─── evolves to ──►  │   │   Notes           │
│  Clinics              │                     │   ├─ Clinical Insights│
│  Reports              │                     │   ├─ Treatment Plan   │
│  Auth/Users           │                     │   ├─ Patient Companion│
│  Admin/Ops            │                     │   ├─ Smart Booking    │
└──────────┬───────────┘                     │   ├─ Document AI      │
           │                                  │   ├─ Ops Intelligence │
           ▼                                  │   └─ Knowledge Search │
   RoleScope.cs                               └──────────┬───────────┘
   (data-access gate)                                    │
           │                                             ▼
           ▼                                  ┌──────────────────────┐
   PostgreSQL 16                              │ RoleScope.cs          │
                                              │ (data-access gate —   │
                                              │  unchanged from v1)   │
                                              └──────────┬───────────┘
                                                         │
                                                         ▼
                                              ┌──────────────────────┐
                                              │ AI Policy Manager     │ ◄── NEW
                                              │  · per-feature opt-in │
                                              │  · consent state      │
                                              │  · PII redaction      │
                                              │    enforcement        │
                                              │  · audit log emission │
                                              │  · cost budget caps   │
                                              │  · model version pin  │
                                              └──────────┬───────────┘
                                                         │
                                                         ▼
                                              ┌──────────────────────┐
                                              │ Data + AI infra       │
                                              │  PostgreSQL 16        │
                                              │   + pgvector          │ ◄── NEW
                                              │  Job queue            │ ◄── NEW (async)
                                              │  Redaction proxy      │ ◄── NEW (sidecar)
                                              │  Audit ledger         │ ◄── NEW (immutable log)
                                              └──────────┬───────────┘
                                                         │
                                                         ▼
                                              External AI services:
                                                Claude / LLM API
                                                Whisper / STT
                                                (+ existing Sentry,
                                                  B2, Firebase Auth)
```

### The 8 AI capability domains

| Domain | What ships | Maturity | Compliance gate |
|---|---|---|---|
| **Transcription & Notes** | Session recording (with consent) · Whisper transcription · auto-SOAP note draft · psy reviews + edits | T1 — foundational | Patient + Psy double opt-in · PII redaction of names/places before LLM call |
| **Clinical Insights** | Pattern detection across sessions · mood trajectory · risk flagging (SI / self-harm) · DSM differentials (advisory only) | T2 — differentiating | Always advisory · psy must explicitly accept any suggestion · risk-flag escalation rules documented |
| **Treatment Planning** | Plan drafting from intake assessment · homework generation · framework alignment (CBT/DBT/ACT) | T2 | Generated artefacts are drafts · psy edits before applying to patient record |
| **Patient Companion** | Mood/journal companion · between-session chatbot (bounded) · guided exercises · TTS | T2 | Strict escalation: crisis content → psy + emergency contacts · never replaces therapist |
| **Smart Booking** | Natural-language scheduling · no-show prediction · capacity-aware suggestions | T1 | No PII leaves the platform · prediction is operational, not clinical |
| **Document AI** | Auto-classification on upload · OCR for scanned/handwritten · PDF extraction · PII redaction · auto-tagging | T1 | Redaction layer is the trust gate for any other external-LLM document processing |
| **Ops Intelligence** | Capacity forecasting · turnover prediction · revenue forecasting · caseload balancing · clinic peer benchmarks (anonymised) | T2 | Operational metrics only, never patient-identifying |
| **Knowledge Search** | Semantic search over own notes (psy's caseload scope) · DSM/ICD-11 lookup · clinical literature search · drug-interaction checker | T2 | Strict per-patient scoping enforced by RoleScope · psy never sees another psy's notes |

The full roadmap with 50+ specific capabilities, maturity tiers (T1 / T2 / T3 frontier), and per-capability compliance gates was drafted in a separate brainstorm artefact used to scope this work.

### Why the AI Policy Manager exists

`RoleScope.cs` handles *data access* (who can read what). AI introduces a parallel concern: **who can run what AI capability against whose data**, and **what evidence does that decision leave behind?** The AI Policy Manager is a separate centerpiece because:

- Consent for AI processing is **per-capability**, not per-data (a patient might consent to mood-journal companion but not to session transcription)
- PII redaction is **always-on** before any external LLM call — needs to be enforceable as middleware, not opt-in per call
- Audit requirements differ from data access (every AI suggestion logged with model + version + input hash + output + downstream action — accept/edit/reject)
- Cost budgets are **per-capability** — Treatment Planning's monthly budget shouldn't be eaten by a runaway Patient Companion session

It's parallel to RoleScope but operates one tier higher in the request flow: RoleScope decides *what data the API can return*; AI Policy Manager decides *what the API can do with that data once retrieved* before it reaches an external service.

---

## Operational practices

Things that aren't visible in the architecture diagram but are essential to operating the system:

- **Demo account fixtures** — 4 permanent accounts maintained on prod for spec/regression purposes: `clinica.teste`, `dr.teste`, `patient.teste`, `patient.preonboard.teste`. Documented provisioning recipes for re-creating any of them if state drifts.
- **VPS access discipline** — direct SSH from owner's Mac (key-based). Hostinger browser console as fallback. Postgres via peer auth (`sudo -u postgres psql -d psinest`).
- **Backup restore drill** — required ops practice. Backups exist if restores work; untested backups are zero.
- **Multi-agent build lane** — the entire system was built (and continues to be built) by the [OpenClaw 7-agent team](../openclaw-hermes-evolution/), now migrated to [Hermes Agent single dispatcher](../openclaw-hermes-evolution/) for cost reasons. Joao reviews every PR and triggers every deploy (three human gates per cycle).
- **Pre-commit branch guard** — every commit verifies `git branch --show-current` matches the expected feature branch. Preventing the kind of cross-agent collision documented in [`openclaw-hermes-evolution/`](../openclaw-hermes-evolution/).
- **Architecture-pause for EF migrations** — two human review points before any DB migration: review entity diff *before* generating, review generated migration file *after*. Bit us once when an auto-generated migration silently dropped a column; pause prevents recurrence.

---

## Related case studies in this portfolio

- [**openclaw-hermes-evolution**](../openclaw-hermes-evolution/) — the **build pipeline** that ships Psinest. Where this repo is the *product*, that one is the *factory*. Two generations of agentic build systems documented with full architecture + cost analysis.
- [**hugo**](../hugo/) — same OpenClaw platform underneath, configured very differently (single marketing agent vs 7-agent dev team). Useful comparison for understanding when single-agent beats multi-agent.
- [**shared-architecture-patterns**](../shared-architecture-patterns/) *(in progress)* — RoleScope.cs-style centralised tenancy, compliance-gated capability rollout, AI Policy Manager centerpieces — patterns that show up across multiple projects.

---

## What's next on Psinest

Roughly in priority order:

1. **Pilot readiness** — finishing the production-readiness sprint (already in flight). Validation, full e2e coverage, full Sentry rules.
2. **Document AI** — first AI capability to land. PII redaction layer is the unblocker for everything else.
3. **AI Policy Manager** — once Document AI is in, every subsequent capability lands behind the Policy Manager gates.
4. **Smart Booking** — operational AI (no clinical risk), good next step after Document AI to validate the AI Policy Manager pattern at low-stakes.
5. **Transcription & Notes** — highest-value clinical capability but also highest-risk; needs the policy manager + redaction layer rock-solid first.

The sequencing here isn't accidental — **lowest-clinical-risk AI capabilities first** so the policy/audit machinery hardens before clinical-grade AI capabilities (transcription, insights, treatment planning) land on top.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com) · 🌐 [psinest.duckdns.org](https://psinest.duckdns.org) (live product)
