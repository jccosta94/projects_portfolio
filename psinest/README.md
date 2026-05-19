# Psinest — Healthcare CRM with AI Augmentation Roadmap
**A live, RGPD-aware healthcare CRM for the Portuguese mental-health market — production traffic today, AI augmentation roadmap designed and partially in flight.**

Psinest is the healthcare CRM I built end-to-end. It's live in pilot at [psinest.duckdns.org](https://psinest.duckdns.org) today: 10 psychologists + 1 clinic running on the platform as a real test deployment. The case study covers two things — the production platform that's already serving users, and the AI augmentation roadmap that's been designed but not all shipped. Both share one load-bearing decision: in mental-health software, RGPD + ERS aren't add-ons. They're the architecture.

Four user roles share one platform (ClinicOwner / Psychologist / Patient / Platform Admin), each with strictly different read scopes, write scopes, and clinical-data exposure. The constraint that triggered the architecture work: mental-health data under EU and Portuguese regulation isn't forgiving of leaky tenancy, and you can't bolt compliance on after the fact. It has to live at the access-enforcement layer, or it doesn't live anywhere.

> **Note on the URL.** `psinest.duckdns.org` is a placeholder domain for the pilot phase. Once the pilot stabilises and feature scope is locked, the platform migrates to a permanent domain with a managed cert + production CDN. The architecture documented here is what the pilot runs on; the migration path is short (DNS + cert swap, no architectural change required).

### What I architected

**The product platform.** A React 19 + TypeScript SPA on top of an ASP.NET Core 8 + EF Core 8 API, talking to PostgreSQL 16 and Backblaze B2 (S3-compatible) for document storage. Firebase Auth handles identity. Sentry handles observability. Hosted on a single Hostinger VPS that also hosts the build pipeline (see [openclaw-hermes-evolution](../openclaw-hermes-evolution/)). The full system shipped in pilot via an AI build pipeline I designed in parallel: Psinest is the *product*, OpenClaw-Hermes is the *factory*.

**The security architecture.** `RoleScope.cs` is the single centralised point where JWT-derived tenancy gets applied at the EF Core query layer. The server never trusts client-supplied filters on list endpoints (`/sessions`, `/patients`, `/documents`); scope is derived from the JWT and enforced at query construction time. ClinicOwner is blocked entirely from documents. Patient can only read their own. Psychologist can only see partnered clinics' patients with valid partnership + consent rows. The patterns this prevents are the patterns that show up in healthcare data breaches: privilege climb via crafted query strings, role confusion, missing tenancy on the wrong endpoint.

**The document permission model.** `IDocumentStore` abstraction over Backblaze B2 with consent-aware permissions. Patient self-uploads at `/patient/documents`; Psychologist uploads at `/psy/patient/N/dynamics` after partnership + consent rows exist; ClinicOwner can never read any document, ever. Read/delete scopes are computed per-document, not per-role. A Psy in their patient's space can delete anything in that space, but the same Psy cannot delete the patient's own self-uploaded docs in a different surface.

**The AI augmentation roadmap.** Eight capability domains layered on top of the existing data plane (transcription, clinical insights, treatment planning, patient companion, smart booking, document AI, ops intelligence, knowledge search), each gated by the **AI Policy Manager** — a separate centerpiece that operates one tier higher than `RoleScope.cs`. RoleScope decides *what data the API can return*; the AI Policy Manager decides *what AI can do with that data before it reaches an external service*. Per-capability consent, always-on PII redaction, per-capability cost budgets, immutable audit log emission, model-version pinning. Designed, partially in flight; document-AI lands first because PII redaction is the unblocker for everything else.

### The architectural insight

Compliance is the concern everything else hangs off. Healthcare data plus EU regulation plus a Portuguese-specific sectoral regulator (ERS) means you don't get to retrofit. `RoleScope.cs` exists as a centralised access gate because the cost of getting tenancy wrong on even one endpoint is "patient data leaks across the clinic boundary," and the only way to make that cost zero is to ensure no controller can opt out of the gate. Centralising the enforcement at the query layer (not the controller layer, not the client layer) is what makes the compliance promise auditable: if `RoleScope.Apply()` runs, scope is enforced; if it doesn't run, no data comes back at all.

**The AI Policy Manager is the same idea, applied one tier up.** When AI capabilities arrive, the concern shifts from "who can read what" to "who can run what AI capability against whose data, and what evidence does that decision leave behind." That's a separate concern with separate consent semantics (per-capability, not per-data), separate cost shape (per-capability budgets), and separate audit requirements (every suggestion logged with model + version + input hash + output + downstream action). Treating AI access as a parallel tier — rather than overloading `RoleScope.cs` — is what keeps the compliance posture coherent as capability surface grows.

**The sequence — current platform first, AI roadmap second, lowest-clinical-risk AI capabilities before highest-risk.** The current platform is live and serving pilot users; the AI roadmap is designed and partially in flight. When AI capabilities land, they land in clinical-risk order — Document AI (low risk, PII-redaction unblocker), Smart Booking (operational, no clinical risk), then upward through Treatment Planning and finally Transcription & Notes (highest clinical value, also highest risk). The Policy Manager pattern gets validated at low-stakes before clinical-grade AI rides on it.

---

## TL;DR

| | **Current state — live pilot** | **Future state — AI-augmented** |
|---|---|---|
| **What's shipping** | React 19 SPA + ASP.NET Core 8 API + Postgres 16 · document storage on Backblaze B2 · Firebase Auth · Sentry observability · multi-tenant role-scoped access | All of the above + 8 AI capability domains layered on top, gated through an AI Policy Manager |
| **Users** | ClinicOwner · Psychologist · Patient · Platform Admin | Same, plus consent state per AI capability per user |
| **Data-access enforcement** | `RoleScope.cs` — JWT-derived tenancy applied at the EF Core query layer | `RoleScope.cs` unchanged (data access is one concern) |
| **AI-capability enforcement** | n/a — no AI yet | **AI Policy Manager** (NEW) — operates one tier up from `RoleScope`; gates per-capability consent, PII redaction, cost budget, audit emission, model version |
| **Document storage** | `IDocumentStore` abstraction over Backblaze B2 · consent-aware read/delete · ClinicOwner blocked entirely | Same store · PII redaction proxy before any external-LLM call processes a document |
| **Compliance posture** | RGPD-aware design baked in (data minimisation, residency, audit trail, right-to-erasure cascades) · ERS-aware (psy credential gate via OPP/cédula) | Same baseline + per-capability AI consent · immutable audit ledger for AI suggestions · cost budget caps per capability |
| **Build pipeline** | Shipped by the [OpenClaw 7-agent team](../openclaw-hermes-evolution/), now migrated to [Hermes single dispatcher](../openclaw-hermes-evolution/) | Same pipeline · AI Policy Manager + capability domains land via the same workflow |
| **Pilot footprint** | 10 psychologists + 1 clinic on `psinest.duckdns.org` | Lands inside the same pilot before moving to a permanent domain |
| **Operating state today** | Production-readiness sprint in flight | Document AI is the first capability scheduled to land; everything else gated behind it |
| **Reversibility** | Domain swap is a DNS + cert change · no architectural change required | AI capabilities are opt-in per-feature — disabling the AI Policy Manager turns the platform back into the current state |

**The headline insight: same data plane, compliance gate already in place, AI layered as a parallel tier rather than retrofitted into the access tier.** Everything that's true today about who-can-read-what stays true tomorrow about who-can-run-what-AI-against-what — they're separate concerns enforced at separate tiers.

---

## Why this matters

Most product teams shipping AI features into regulated domains in 2026 hit two walls in sequence: **compliance first, AI second.**

**Wall 1 — Compliance.** Mental-health data under RGPD + ERS is not forgiving of accidental cross-tenant leakage. A ClinicOwner accidentally reading a patient's session note isn't a UX bug — it's a regulatory exposure that, depending on jurisdiction and scale, ends the product. Bolt-on tenancy ("we'll add the filter in the controller") loses by construction, because every new endpoint is a new opportunity to forget. **Psinest solves this by centralising tenancy enforcement at the query layer**: every list query passes through `RoleScope.cs`, scope is derived from the JWT and applied at query construction time, no controller can opt out, no client filter can override.

**Wall 2 — AI without losing compliance.** Once compliance works, the second wall is layering AI capabilities without breaking it. The naive move is to extend `RoleScope.cs` with AI permissions — but that conflates two concerns. *Who can read this patient's document* and *what AI can do with this document once it's read* are different decisions with different consent semantics, different cost shapes, and different audit requirements. **Psinest solves this with the AI Policy Manager** — a parallel tier that operates one level higher than `RoleScope`, enforcing per-capability consent, always-on PII redaction, per-capability cost budgets, and immutable audit log emission for every AI invocation.

The lesson is **the sequence**: build the compliance gate first, prove the platform can serve real users in a regulated domain, *then* layer AI on top — and when you do, give AI its own enforcement tier rather than overloading the data-access tier. Trying to ship AI features before the compliance gate exists ends with a product that can't pass a privacy audit; trying to bolt AI consent onto a data-access layer that wasn't designed for it ends with a tangled gate where neither concern is enforced cleanly.

---

## Screenshots

Captured from the live product at [psinest.duckdns.org](https://psinest.duckdns.org) using the four permanent demo accounts (`clinica.teste`, `dr.teste`, `patient.teste`, `patient.preonboard.teste`). All data shown is synthetic — these are fixtures maintained on production specifically for spec/regression and portfolio use.

| | |
|---|---|
| ![Public landing page](./screenshots/01-landing.png) | ![Patient — home & activities](./screenshots/02-patient-sessions.png) |
| **Public landing page** — Portuguese marketing surface with role-specific routing after authentication via Firebase. The product positions itself as the platform that unites clinics, psychologists, and patients. | **Patient — home & activities** — patient dashboard with next session, session count, pending activities, and psy-attached dynamics (homework). Sidebar nav exposes Sessions / Chat / Activities / Documents / Premium Content / Profile. |
| ![Psychologist — patient detail](./screenshots/03-psy-patient-detail.png) | ![Psychologist — weekly calendar](./screenshots/04-psy-session-notes.png) |
| **Psychologist — patient detail** — per-patient profile, patient-journey link (session history + case formulation + clinical documentation), cancellation-risk surface. Psy sees only patients in their partnered clinics — enforced server-side at `RoleScope`, no client filtering. | **Psychologist — weekly calendar** — week view with completed (green), scheduled (blue), and cancelled (red strikethrough) sessions across multiple consultórios. Session-notes drill-down lives one click in; the calendar is the workflow anchor. |
| ![Clinic Owner — dashboard](./screenshots/05-owner-dashboard.png) | ![Clinic Owner — reports & analytics](./screenshots/06-owner-reports.png) |
| **Clinic Owner — dashboard** — psychologists / patients / sessions / partnerships / locations stat cards + recent sessions log. Owner never sees patient clinical data; document access blocked entirely at the `RoleScope` layer. | **Clinic Owner — reports & analytics** — operational vs financial split, time-range selector, session-status donut breakdown (cancelled / completed / in-progress / no-show / scheduled), and mode breakdown. "Volume only — no clinical content" is the explicit policy. |
| | ![Patient — documents](./screenshots/08-documents.png) |
| *Admin — psy verification queue (ERS compliance gate where Platform Admin reviews OPP / cédula credentials before activating new psychologist accounts) — captured separately.* | **Patient — documents** — patient's own document space with Upload action. Policy text ("PDF or image, up to 25 MB. Documents attached to a dynamic show their source.") is consent-aware: patient can read/upload/delete own documents; ClinicOwner is blocked entirely; psy sees only after partnership + consent rows exist. |

---

## Current-state architecture

The full v1 platform — SPA on top of a .NET API, talking to Postgres and B2, all running on a single shared VPS that also hosts the build pipeline. The diagram below shows how those pieces fit together; the table beneath it documents each tier in detail.

```text
                    Public users (Psy · Patient · Owner · Admin)
                                  │
                                  │ HTTPS · psinest.duckdns.org
                                  ▼
                       ┌──────────────────────┐
                       │   HOSTINGER VPS      │             ← 72.62.2.211
                       │   single shared host │               Ubuntu 24.04
                       │                      │               1 vCPU · 3.8 GB
                       └──────────┬───────────┘               pilot + build co-located
                                  │ network edge
                                  ▼
                       ┌──────────────────────┐
                       │   nginx 1.24         │             ← Let's Encrypt TLS
                       │   reverse proxy +    │               HTTP → HTTPS redirect
                       │   static SPA serve   │               ports 80 / 443
                       └──────────┬───────────┘
                                  │
                ┌─────────────────┴─────────────────┐
                │                                   │
                ▼                                   ▼
       ┌────────────────┐                  ┌────────────────┐
       │  React 19 SPA  │                  │  Kestrel API   │   ← .NET 8
       │  ────────────  │                  │  ────────────  │     localhost:5000
       │  TypeScript    │                  │  ASP.NET Core  │
       │  Vite 8 ·      │  ── /api ──▶    │  EF Core 8     │
       │  Tailwind 4 ·  │   HTTPS+JWT      │  RoleScope.cs  │     ← JWT-derived
       │  Recharts ·    │                  │  IDocumentStore│       tenancy gate
       │  i18n pt-PT    │                  │  Sentry .NET   │
       └───────┬────────┘                  └────────┬───────┘
               │ Firebase JWT                       │ reads/writes
               │ Sentry SPA                         │
               ▼                                    ▼
                                  ┌──────────────────────┐
                                  │  PostgreSQL 16       │   ← localhost:5432
                                  │  db: psinest         │     peer auth
                                  │  PascalCase columns  │     no password
                                  │  lowercase tables    │
                                  └──────────┬───────────┘
                                             │ pg_dump cron
                                             ▼ (4× / day)
              ┌─────────────────────────────────────────────────────────┐
              │ EXTERNAL SERVICES (outbound from VPS)                   │
              │   Firebase Auth (project psinestv2) · JWT issue+verify  │
              │   Backblaze B2 · psinest-uploads + psinest-backups      │
              │   Sentry · org psinest (dotnet-aspnetcore + psinest-app)│
              │   DuckDNS · dynamic DNS for psinest.duckdns.org         │
              │   GitHub Actions · workflow_dispatch deploys (manual)   │
              └─────────────────────────────────────────────────────────┘
```

**Per-tier detail:**

| Tier                   | What's running                                                                                                                                                                                                                                |
|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Network edge**       | nginx 1.24 on ports 80 + 443 with Let's Encrypt TLS · HTTP → HTTPS redirect · static SPA bundle served from `/var/www` · `/api/*` reverse-proxied to Kestrel at `127.0.0.1:5000`. DuckDNS dynamic-DNS resolves `psinest.duckdns.org`.            |
| **Frontend (SPA)**     | React 19 + TypeScript + Vite 8 + Tailwind 4 + Recharts · i18n pt-PT default with en fallback (canonical glossary: `docs/pt-pt-clinical-glossary.md`) · Firebase Auth client SDK for browser-side sign-in · Sentry SPA SDK with `sendDefaultPii=false`. |
| **Backend (API)**      | Kestrel on `127.0.0.1:5000` · ASP.NET Core 8 + EF Core 8 · Firebase Admin SDK (server-side JWT verify) · `RoleScope.cs` (centralised JWT-derived tenancy at the query layer) · `IDocumentStore` abstraction (B2 ↔ filesystem swap) · Sentry .NET SDK. |
| **Controllers**        | `/users` · `/clinics` · `/psychologists` · `/patients` · `/sessions` · `/documents` · `/reports` · `/partnerships` · `/onboarding` · `/admin`. All list endpoints pass through `RoleScope` — no controller opts out of the gate.                  |
| **Data**               | PostgreSQL 16 on `127.0.0.1:5432` · db `psinest` · peer auth (no password). Core tables: `users` · `clinics` · `psychologists` · `patients` · `sessions` · `documents` · `partnerships` · `cancellations` · `onboarding_state` · `audit`. Schema convention: PascalCase quoted columns (`"Id"`, `"FirebaseUid"`), lowercase unquoted tables. |
| **Backups**            | `pg_dump` cron at `17 */6 * * *` → Backblaze B2 `psinest-backups` bucket · 4 backups / day · retention managed via B2 lifecycle policy · restore drill documented as required ops practice.                                                       |
| **Document storage**   | Backblaze B2 `psinest-uploads` (S3-compatible) accessed via the AWS S3 .NET SDK behind `IDocumentStore`. Consent-aware read/delete permissions enforced server-side — see [Document permission model](#document-permission-model) below.            |
| **Auth**               | Firebase Auth · project `psinestv2` · email + password · server-side JWT verification via Firebase Admin SDK · password reset built-in. Server never trusts client-supplied user IDs.                                                              |
| **Observability**      | Sentry · org `psinest` · 2 projects: `dotnet-aspnetcore` (API) + `psinest-app` (SPA). Both env-driven DSN, both `sendDefaultPii=false`.                                                                                                          |
| **CI/CD**              | GitHub Actions on `jccosta94/psinest-app` + `jccosta94/psinest-api`. **`workflow_dispatch` only** (no auto-deploy on merge). Lint + build are the only merge gates; e2e is non-blocking smoke. Cross-repo deploy order: **API first → 60 s wait → App second** (App-first hits 404s). Deploy runs SSH to the VPS · `systemctl restart psinest-api` · rsync `/var/www`.            |

**Security boundaries:**

| Boundary                        | Enforcement                                                                                                                            |
|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Public ingress**              | Only port 22 (SSH key-only) + 80/443 (nginx + TLS). API on 5000 and Postgres on 5432 bind to `127.0.0.1` only — no network exposure.     |
| **Tenancy enforcement**         | `RoleScope.cs` — JWT-derived, applied at the EF Core query layer, never trusts client filters. Every list endpoint passes through it.    |
| **Document scope**              | ClinicOwner blocked from documents entirely. Patient reads own only. Psy reads partnered clinic + assigned patient only. See [Document permission model](#document-permission-model). |
| **Postgres isolation**          | `listen_addresses = 'localhost'` + peer auth — no network exposure, no password auth on the local socket.                                |
| **Build / prod co-location**    | Build pipeline runs on the same VPS as the production stack. Pre-revenue cost choice, accepted explicitly. Migration path is one host split. |
| **Three human-in-the-loop gates** | (1) CEO proposes priorities from the kanban board, Joao picks. (2) Joao reviews every PR, no agent self-merge. (3) Joao runs `workflow_dispatch` manually for every deploy. (See [openclaw-hermes-evolution](../openclaw-hermes-evolution/) for the build-pipeline detail.) |

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

## Architecture diagrams

All diagrams in this repo are inline text-style ASCII inside fenced code blocks — readable on GitHub, version-controlled, diff-able, no rendering tooling required.

- **Current-state topology** — SPA + Kestrel API + Postgres + B2 + Firebase + Sentry on one VPS. See [Current-state architecture](#current-state-architecture) above.
- **Security architecture (`RoleScope.cs`)** — JWT → middleware → RoleScope → controller guard → EF Core query construction. See [Security architecture — Role-scoped access via `RoleScope.cs`](#security-architecture--role-scoped-access-via-rolescopecs) above.
- **Document permission model** — upload-path vs read/delete-path split, with canDelete policy + read scope rules per role. See [Document permission model](#document-permission-model) above.
- **AI augmentation: current → future** — side-by-side current data plane vs future AI-augmented state with the AI Policy Manager and pgvector / job queue / redaction proxy / audit ledger as new components. See [AI augmentation roadmap (future state)](#ai-augmentation-roadmap-future-state) above.

---

## What I designed vs what the platforms provide

| Layer                             | Adopted platform / framework                                                                             | What I designed on top                                                                                                                                          |
|-----------------------------------|----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Identity**                      | Firebase Auth (email + password, JWT issue/verify)                                                       | The role-on-register flow, the OPP/cédula verification queue (ERS gate), the grandfathering rule for existing users                                              |
| **API + data access**             | ASP.NET Core 8 + EF Core 8 (controllers, middleware, query API)                                          | **`RoleScope.cs`** — the centralised JWT-derived tenancy gate; every list endpoint passes through it · the controller-routing pattern that makes the gate inescapable |
| **Document storage**              | Backblaze B2 (S3-compatible) + AWS S3 .NET SDK                                                           | **`IDocumentStore`** abstraction (B2 ↔ filesystem swap) · the consent-aware document permission model · the upload-path vs read/delete-path policy split        |
| **Database**                      | PostgreSQL 16 (Hostinger-hosted) + peer auth                                                             | The schema convention (PascalCase quoted columns + lowercase unquoted tables) · the audit-row pattern for role-affecting actions · the EF migration architecture-pause discipline |
| **Observability**                 | Sentry (org `psinest`, two projects)                                                                     | The env-driven DSN pattern · `sendDefaultPii=false` baseline · the SPA + API split for separating client vs server exceptions                                     |
| **Web tier**                      | nginx 1.24 + Let's Encrypt + DuckDNS                                                                     | The SPA-bundle + reverse-proxy split · the cross-repo deploy ordering (API first → 60 s wait → App)                                                              |
| **Backups**                       | `pg_dump` + Backblaze B2 + cron                                                                          | The `psinest-pg-dump` wrapper + B2 lifecycle policy · the backup-restore-drill ops practice (untested backups are zero)                                          |
| **CI/CD**                         | GitHub Actions + `workflow_dispatch`                                                                     | The "lint + build are the only merge gates · e2e is non-blocking smoke" policy · cross-repo PR sequencing                                                        |
| **Build pipeline (meta)**         | [OpenClaw](https://openclaw.ai/) (v1, 7 agents) → [Hermes Agent](https://hermes-agent.nousresearch.com/) (v2, single dispatcher) | The dispatch-authorization graph (`subagents.allowAgents`), the three human-in-the-loop gates, the dev/prod co-location pattern. See [openclaw-hermes-evolution](../openclaw-hermes-evolution/) |
| **AI capability gate (future)**   | n/a — purpose-built                                                                                      | **The AI Policy Manager** — per-capability consent · always-on PII redaction · per-capability cost budget · audit ledger · model-version pinning                  |

The pattern: I don't reinvent identity, storage, or transport. I do design **the enforcement gates** that make compliance promises auditable — `RoleScope` for data access, `IDocumentStore` for document scope, the AI Policy Manager (incoming) for AI-capability access.

---

## Stack

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

## Operational practices

Things that aren't visible in the architecture diagram but are essential to operating the system:

- **Demo account fixtures** — 4 permanent accounts maintained on prod for spec/regression purposes: `clinica.teste`, `dr.teste`, `patient.teste`, `patient.preonboard.teste`. Documented provisioning recipes for re-creating any of them if state drifts.
- **VPS access discipline** — direct SSH from owner's Mac (key-based). Hostinger browser console as fallback. Postgres via peer auth (`sudo -u postgres psql -d psinest`).
- **Backup restore drill** — required ops practice. Backups exist if restores work; untested backups are zero.
- **Multi-agent build lane** — the entire system was built (and continues to be built) by the [OpenClaw 7-agent team](../openclaw-hermes-evolution/), now migrated to [Hermes Agent single dispatcher](../openclaw-hermes-evolution/) for cost reasons. Joao reviews every PR and triggers every deploy (three human gates per cycle).
- **Pre-commit branch guard** — every commit verifies `git branch --show-current` matches the expected feature branch. Preventing the kind of cross-agent collision documented in [`openclaw-hermes-evolution/`](../openclaw-hermes-evolution/).
- **Architecture-pause for EF migrations** — two human review points before any DB migration: review entity diff *before* generating, review generated migration file *after*. Bit us once when an auto-generated migration silently dropped a column; pause prevents recurrence.

---

## Deeper reading

### Related case studies in this portfolio

- [**openclaw-hermes-evolution**](../openclaw-hermes-evolution/) — the **build pipeline** that ships Psinest. Where this repo is the *product*, that one is the *factory*. Two generations of agentic build systems documented with full architecture + cost analysis.
- [**hugo**](../hugo/) — same Hermes Agent platform underneath, configured very differently (single marketing agent + MCP tools vs 7-agent dev team). Useful comparison for understanding when single-agent beats multi-agent.
- [**shared-architecture-patterns**](../shared-architecture-patterns/) *(in progress)* — `RoleScope.cs`-style centralised tenancy, compliance-gated capability rollout, AI Policy Manager centerpieces — patterns that show up across multiple projects.

### Live system

- [`psinest.duckdns.org`](https://psinest.duckdns.org) — the live pilot, accessible via the four permanent demo accounts (`clinica.teste@psinest.com` / `dr.teste@psinest.com` / `patient.teste@psinest.com` / `patient.preonboard.teste@psinest.com`, all password `Psinest2026!`).

---

## Status

- ✅ **Current platform** — React 19 SPA + ASP.NET Core 8 API + Postgres 16 + Backblaze B2 + Firebase Auth + Sentry, all on a single Hostinger VPS. Pilot serving 10 psychologists + 1 clinic on `psinest.duckdns.org`.
- ✅ **`RoleScope.cs`** — centralised JWT-derived tenancy at the EF Core query layer. Every list endpoint passes through it. Document permission model wired in.
- ✅ **Document storage** — `IDocumentStore` abstraction over Backblaze B2. Consent-aware permissions. ClinicOwner blocked entirely.
- ✅ **Backups + restore drill** — `pg_dump` every 6 hours to B2 `psinest-backups`. Restore drill documented and exercised.
- ✅ **Observability** — Sentry on both SPA and API. Env-driven DSN, `sendDefaultPii=false`.
- ✅ **Build pipeline** — Psinest ships via the [OpenClaw 7-agent team → Hermes Agent](../openclaw-hermes-evolution/) build system. Joao reviews every PR and triggers every deploy (three human gates per cycle).
- 🚧 **Production-readiness sprint** — full e2e coverage, full Sentry rule set, content validation, permanent-domain migration plan.
- 🚧 **AI Policy Manager** — designed; lands as the gate for the first AI capability (Document AI).
- 🚧 **Document AI** — first AI capability scheduled to land. PII redaction proxy is the unblocker for everything downstream.
- 🚧 **Smart Booking** — second AI capability (operational, no clinical risk) — validates the Policy Manager pattern at low-stakes before clinical-grade AI lands.
- 🚧 **Transcription & Notes** — highest-value clinical capability; lands only after the Policy Manager + redaction layer are battle-hardened.
- 🚧 **Clinical Insights / Treatment Planning / Patient Companion / Ops Intelligence / Knowledge Search** — remaining capability domains; lands progressively behind the same Policy Manager gates, in increasing clinical-risk order.

The sequencing isn't accidental — **lowest-clinical-risk AI capabilities first** so the policy/audit machinery hardens before clinical-grade AI capabilities ride on it.

---

## Acknowledgements

The platforms this case study is built on:

- **Firebase Auth** (Google) — identity layer (JWT issue/verify, password reset, email+password). Cleanly scoped; never had to reach for a custom auth implementation.
- **ASP.NET Core 8 + EF Core 8** (Microsoft, MIT) — the API runtime and ORM. The query-construction API is what makes `RoleScope.cs` possible — scope applied at query construction is strictly safer than scope applied as a client filter.
- **PostgreSQL 16** — the data store. Peer auth + `listen_addresses = 'localhost'` is what makes localhost-only Postgres trivially safe.
- **Backblaze B2** (S3-compatible) — document and backup storage. Order-of-magnitude cheaper than S3 at this scale, EU region available, AWS SDK works as-is.
- **Sentry** — observability. Two-project split (SPA + API) + env-driven DSN + `sendDefaultPii=false` is the baseline I keep coming back to.
- **nginx + Let's Encrypt + DuckDNS** — the public-edge stack. Standard, boring, works.
- **Hostinger VPS** — the host. Pre-revenue cost discipline; the day revenue justifies it, the production stack splits to its own box.
- **[OpenClaw](https://openclaw.ai/)** (by [@steipete](https://x.com/steipete)) and **[Hermes Agent](https://hermes-agent.nousresearch.com/)** (open source, by [Nous Research](https://nousresearch.com)) — the build-pipeline platforms that ship Psinest. Documented separately in [openclaw-hermes-evolution](../openclaw-hermes-evolution/).
- **Claude Code CLI** (Anthropic) — the code-writing executor that does the actual implementation work in both build-pipeline generations.

**`RoleScope.cs`, `IDocumentStore`, the document permission model, the AI Policy Manager pattern**, and the compliance-gated capability rollout (RGPD + ERS gates layered onto the standard data-access tier) are mine — designed for this product, documented here so the patterns are reusable in the next regulated-domain platform.

The four-role product surface (ClinicOwner / Psychologist / Patient / Platform Admin) and the consent-aware document permission model are borrowed from how mental-health practice actually works — informed by working with practicing psychologists during the pilot. **Multi-role healthcare platforms work best when they mirror the actual access boundaries the practice already enforces in the analog world.**

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com) · 🌐 [psinest.duckdns.org](https://psinest.duckdns.org) (live product)
