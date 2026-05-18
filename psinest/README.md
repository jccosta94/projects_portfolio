# Psinest — Healthcare CRM with AI Augmentation Roadmap

🚧 **Case study in progress.** Full architecture writeup landing soon.

[Psinest](https://psinest.duckdns.org) is a production healthcare CRM for psychologists, clinics, and patients in the Portuguese market — live and serving real users. This repo will document the full architecture and the AI augmentation roadmap that's being designed on top of it.

**What's documented here when the writeup lands:**
- System architecture — role-scoped access via `RoleScope.cs`, consent-aware document permissions, multi-tenant data isolation, observability + backup topology
- Stack: React 19 + TypeScript + Vite 8 + Tailwind 4 / ASP.NET Core 8 + EF Core 8 / PostgreSQL 16 / Firebase Auth / Backblaze B2 / Sentry / Hostinger VPS
- AI augmentation roadmap — 8 capability domains (transcription, clinical insights, treatment planning, patient companion, smart booking, document AI, ops intelligence, knowledge search) mapped onto a compliance-aware deployment plan
- Compliance gates — RGPD / ERS-aware design, PII redaction proxy, AI Policy Manager centerpiece, advisory-only clinical AI patterns

**In the meantime:**
- 🔗 [Portfolio index](https://github.com/jccosta94) — see all projects
- 🔗 [OpenClaw-Hermes Evolution](https://github.com/jccosta94/openclaw-hermes-evolution) — the multi-agent build system case study (currently the most polished portfolio entry with full architecture + cost analysis)
- 🌐 [psinest.duckdns.org](https://psinest.duckdns.org) — live product

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
