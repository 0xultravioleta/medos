---
type: business
date: "2026-02-28"
status: active
tags:
  - business
  - demo
  - phase-1
---

# MedOS Platform Demo Script — Monday Meeting

> **Audience:** Dr. Di Reze, Family Office (Miami, ~$100M AUM)
> **Role being filled:** Lead of AI Ops — Healthcare OS
> **Demo URL:** https://medos-platform.vercel.app
> **Duration:** 10-12 minutes total (5-7 min demo + 2-3 min architecture + close)

---

## Pre-Demo Setup (30 seconds)

- [ ] Open https://medos-platform.vercel.app in Chrome (incognito recommended)
- [ ] Second tab: https://github.com/0xultravioleta/medos-platform
- [ ] Optional third tab: Obsidian vault open showing [[HEALTHCARE_OS_MASTERPLAN]] and the `_MOCs/` index
- [ ] Screen sharing ready, browser at 100% zoom
- [ ] Close unrelated tabs and notifications

---

## The Narrative — Frame Before You Demo

> **Opening line:**
>
> "Before writing a single line of code, I spent a full day researching the healthcare domain systematically."

Key beats to hit:

- **121,000+ words** of structured research across **44 documents** — see [[MOC-Domain-Knowledge]]
- **Architecture Decision Records** documenting every technical choice — see [[MOC-Architecture]]
- Deep dives into [[FHIR R4]], [[HIPAA]], revenue cycle management, X12 EDI
- A **117-task execution plan** across **7 sprints** — the entire 90-day roadmap is already mapped
- "I treat healthcare engineering the way a physician treats a case — research first, diagnose the problem space, then build the treatment plan."

---

## Demo Flow (5-7 minutes)

### 1. Login (30 seconds)

**URL:** `/`

**Credentials:**
- Email: `dr.direze@sunshinemedical.com`
- Password: `demo123`

**Talking points:**
- "Split-screen design — clean, enterprise-grade UX that physicians will actually use."
- "Notice the HIPAA compliance badges. Compliance is not a checkbox we add later — it is the foundation."
- "This accepts any credentials for the demo. Production uses Auth0 with SMART on FHIR OAuth2 — the healthcare-specific authentication standard."

---

### 2. Dashboard (60 seconds)

**URL:** `/dashboard`

**Show:**
- Personalized greeting with provider name
- KPI cards (patients seen today, revenue, pending tasks)
- Today's schedule with patient names
- Activity feed / recent events
- Revenue overview chart

**Talking points:**
- "Real-time operational intelligence. The practice manager sees revenue and operations. The provider sees their schedule and clinical alerts."
- "Every data point maps to a FHIR resource. This is not a dashboard bolted on top — the data model IS the dashboard."
- Click on a patient name in the schedule to transition naturally to the patient list.

---

### 3. Patient List (45 seconds)

**URL:** `/patients`

**Show:**
- Searchable, sortable patient table
- Risk scores displayed per patient
- Insurance information columns
- Quick filters

**Talking points:**
- "FHIR-native data model. Every patient record is a FHIR R4 Patient resource stored as JSONB in PostgreSQL."
- "Risk stratification is computed from conditions, encounters, and social determinants — not a static field."
- Click on **Robert Chen** (high risk, multiple conditions) to drill into the detail view.

---

### 4. Patient Detail (60 seconds)

**URL:** `/patients/p-004`

**Show:**
- Full patient profile (demographics, insurance, contacts)
- Active conditions list with severity
- Clinical timeline (encounters, labs, referrals)
- AI-generated SOAP note preview

**Talking points:**
- "AI confidence scoring. The system tells you HOW confident it is in every inference. Below 85%, it flags for human review. This is not a black box."
- "The clinical timeline is an event-sourced view of FHIR Encounter resources. Every interaction is immutable and auditable."
- "This is the patient-centric view that physicians actually need — not buried in 47 tabs like legacy EHRs."

---

### 5. AI Scribe — THE STAR (90 seconds)

**Navigation:** AI Notes -> Start AI Scribe
**URL:** `/ai-notes/new`

**Steps:**
1. Select patient: **Robert Chen**
2. Select visit type: **Follow-up**
3. Click **Start Recording** -> let it run 5-10 seconds
4. Click **Stop**
5. Watch the processing steps animate (transcription -> analysis -> note generation)
6. Show the generated SOAP note with typewriter effect

**Talking points:**
- "This simulates our ambient AI documentation pipeline. The physician speaks naturally during the encounter — no typing, no clicking."
- "In production: **Whisper v3** for transcription, **Claude via AWS Bedrock** with a HIPAA BAA for clinical reasoning, then structured SOAP note output."
- "Anthropic launched **Claude for Healthcare** in January 2026 — HIPAA-ready with Business Associate Agreements on all three major clouds. We are building on the most capable medical AI available."
- "Notice the **ICD-10 and CPT code suggestions** with individual confidence scores. This is not just documentation — it is revenue capture. Every missed code is lost revenue."
- "Physicians save **30+ minutes per day**. A JAMA study showed burnout dropped from 52% to 39% with ambient AI documentation. This is the feature that sells itself."

> **Pause here.** Let this sink in. This is the emotional peak of the demo.

---

### 6. Claims / Revenue Cycle (45 seconds)

**URL:** `/claims`

**Show:**
- Claims table with CPT codes, ICD-10 codes, status tracking
- Denial rate metrics
- Claim lifecycle visualization

**Talking points:**
- "Revenue cycle management is WHERE THE MONEY IS. **$496 billion** in annual administrative waste in US healthcare."
- "AI-powered denial management recovers **$1-3 million per year** for a 200-provider practice."
- "We catch coding errors, missing modifiers, and authorization gaps BEFORE the claim goes out — not after the denial comes back 60 days later."
- Reference [[05-domain/billing]] for the deep research behind this module.

---

### 7. Analytics (30 seconds)

**URL:** `/analytics`

**Show:**
- KPI summary cards
- Revenue trend line
- Patient volume over time
- Top procedures by volume and revenue

**Talking points:**
- "Population health analytics, benchmarking against regional averages, and care gap identification."
- "This is the data layer that powers value-based care contracts. Practices that can prove outcomes get better reimbursement."

---

## Architecture Discussion (2-3 minutes)

Switch to the **GitHub repo** tab or the **Obsidian vault**.

### Tech Stack (aligned with [[HEALTHCARE_OS_MASTERPLAN]])

| Layer | Technology | Why |
|-------|-----------|-----|
| Backend | FastAPI (Python 3.12+) | 56 tests, 99% coverage, async-first |
| Frontend | Next.js 15 | 11 routes, React Server Components, Tailwind |
| Database | PostgreSQL 17 + pgvector | FHIR JSONB storage + vector search for AI |
| CI/CD | GitHub Actions | Automated testing, linting, deployment |
| Multi-tenancy | Schema-per-tenant + RLS | Row Level Security for data isolation |
| AI | Claude via AWS Bedrock | HIPAA BAA, structured outputs, tool use |
| Auth (planned) | Auth0 + SMART on FHIR | Healthcare-standard OAuth2 |

### HIPAA Compliance — Built In, Not Bolted On

- **PHI filter** in all logging — no patient data in CloudWatch
- **FHIR AuditEvent** trail for every data access
- **Schema-per-tenant** isolation — a breach in one practice cannot reach another
- **Encryption everywhere** — at rest (AES-256) and in transit (TLS 1.3)
- **Structured logging** — every access is traceable for compliance audits

### Documentation Depth

- **4 Architecture Decision Records** — see [[04-architecture/adr]]
- **117-task execution plan** across 7 sprints — see [[03-projects]]
- **44 research documents** covering FHIR, HIPAA, RCM, X12 EDI, clinical workflows
- "Everything is documented. If I get hit by a bus, the next engineer can pick this up and keep building."

---

## The Close

> "This is **Day 2** of development. The research was Day 0-1. Imagine what **90 days** looks like."

> "Give me 90 days. I will build the infrastructure, ship an MVP, and get it in front of real practices."

### The 90-Day Vision

- **Days 1-30:** Core platform — auth, FHIR engine, multi-tenant database, basic UI
- **Days 31-60:** AI pipeline — ambient scribe, clinical decision support, coding assistant
- **Days 61-90:** Revenue cycle — claims engine, denial management, analytics dashboard
- **Target:** 3-5 pilot practices, mid-size specialty in Florida (orthopedics or dermatology)
- Reference the full plan: [[HEALTHCARE_OS_MASTERPLAN]]

---

## Objection Handling

### "Is this just a mock?"

> "Yes, and that is **deliberate**. I prioritized showing the FULL vision over wiring one feature to a database. Every mock component has a documented production upgrade path in the codebase. The architecture underneath is real — the API routes, the data models, the multi-tenant schema, the CI/CD pipeline. What you are seeing is not a Figma prototype. This is a running Next.js application deployed on Vercel with a real FastAPI backend structure."

### "What about HIPAA?"

> "HIPAA compliance is built into the architecture from Day 1. PHI filter in logging, FHIR AuditEvent audit trail, schema-per-tenant isolation, encryption everywhere. Claude for Healthcare — announced January 2026 — gives us HIPAA-ready AI through AWS Bedrock with a signed BAA. We do not need to build our own model or worry about PHI leaving a compliant boundary."

### "Can one or two people build this?"

> "With Claude Code and Cursor, we ship at 5-10x the speed of a traditional team. I built this entire demo — the research, the backend, the frontend, the CI/CD pipeline, the deployment — in **48 hours**. That is the AI multiplier. A two-person team with these tools operates like a team of ten."

### "Why not just use Epic/Cerner/athenahealth?"

> "Legacy EHRs were built in the 1990s and retrofitted for the internet. They charge $50K-500K for implementation, take 6-18 months, and physicians hate using them. We are building AI-native from scratch — the UX is designed around how physicians actually work, not around how databases were structured 30 years ago."

### "What is the revenue model?"

> "SaaS per-provider pricing, starting at $200-500/provider/month for the core platform. Revenue cycle management is performance-based — we take a percentage of recovered revenue from denied claims. The AI scribe alone justifies the cost in physician time saved."

---

## Post-Demo Follow-Up

- [ ] Send link to the live demo for Dr. Di Reze to explore independently
- [ ] Share the [[HEALTHCARE_OS_MASTERPLAN]] as a PDF export
- [ ] Prepare a 90-day milestone document with specific deliverables and success metrics
- [ ] Schedule a follow-up meeting for deeper technical dive if requested
