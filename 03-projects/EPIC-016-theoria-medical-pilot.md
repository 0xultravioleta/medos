---
type: epic
date: "2026-03-01"
status: done
tags:
  - epic
  - frontend
  - backend
  - agents
  - theoria
  - post-acute
  - phase-1
  - vbc
---

# EPIC-016: Theoria Medical Pilot Sprint

> Build Theoria Medical-specific capabilities that demonstrate MedOS as the AI-native OS for distributed post-acute and value-based care. Every feature maps directly to Dr. Di Rezze's operational pain points and Theoria's growth trajectory.

**Sprint:** S4F (Frontend Phase 3 + Agent Architecture)
**Owner:** A + AI
**Related:** [[HEALTHCARE_OS_MASTERPLAN]] | [[ADR-006-patient-context-rehydration]] | [[ADR-007-wearable-iot-integration]] | [[ADR-008-a2a-agent-communication]] | [[agent-architecture]]

---

## Context: Why Theoria Medical

### Dr. Justin Di Rezze — Profile

- **Education:** Wayne State University School of Medicine, Class of 2014
- **Specialty:** Internal Medicine (Board-certified Internist)
- **Career Arc:** Hospitalist → Chief Hospitalist at Ascension Genesys Hospital → SNF Medical Director → Founded Theoria Medical
- **Pivot Moment:** As SNF medical director, discovered the "false sense of security" hospitalists have when discharging to post-acute care — the dangerous void of physician presence in SNFs
- **Scaling Speed:** 1 SNF → largest post-acute group in Michigan in **14 months**
- **Current:** CEO of Theoria Medical (21 states), Principal at Praetorian Acquisition Corp (SPAC)
- **Family Office:** DiRezze Family Office LLC (registered Feb 2025, Sunny Isles FL, managed by Arghavan Di Rezze)
- **Key Contact:** Kevin Pezeshkian, CSO (Chief Strategy Officer)
- **NPI:** 1558775700 | Michigan License: 4301105658
- **Clinical Focus:** Cardiovascular/Internal Medicine (prescribes Amlodipine, Clopidogrel, Simvastatin, Spironolactone, Metoprolol, Apixaban)

### Theoria Medical — Deep Profile

- **What:** Tech-enabled MSO (Management Service Organization) for post-acute care
- **Where:** 21 states, hundreds of SNFs/ALFs/hospitals
- **Model:** 24/7 telemedicine + VBC (ACO REACH via Empassion Health, largest ACO REACH participant in the US)
- **PE Partner:** Amulet Capital Partners ($2.7B AUM, healthcare-focused, Dec 2024 investment)
- **Advisors:** Houlihan Lokey (Amulet), Cain Brothers (Theoria), McDermott Will & Emery (legal)
- **Proprietary Tech Stack:**
  - **ChartEasy** — Specialized EHR for long-term care clinical/regulatory framework
  - **ChatEasy** — Secure messaging platform for clinical decision support
  - **ProphEasy** — Clinical decision support and analytics tool
- **Self-description:** "Only physician group in the country that creates its own technology in-house"

### Critical Positioning: MedOS + ChartEasy/ChatEasy/ProphEasy

**MedOS does NOT replace Theoria's proprietary tools.** MedOS is the **AI-native infrastructure layer underneath** that supercharges them:
- ChartEasy stores clinical data → MedOS normalizes it to FHIR, adds pgvector search, enables AI agent access
- ChatEasy handles messaging → MedOS adds A2A protocol for agent-to-agent coordination on top
- ProphEasy does analytics → MedOS adds predictive AI agents with confidence scoring and bounded autonomy

**Analogy:** ChartEasy/ChatEasy/ProphEasy are the apps. MedOS is the operating system.

### The Pitch Thesis

MedOS transforms Theoria from a **physician services company** (8-12x EBITDA) into a **technology platform** (15-25x EBITDA), directly increasing enterprise value for Amulet Capital's exit strategy. MedOS doesn't compete with Theoria's internal tools — it makes them 10x more powerful.

---

## Scope

13 frontend pages + 7 new agent architectures organized into 4 domains:

### Clinical Operations
| Section | Route | Description | Status |
|---------|-------|-------------|--------|
| Facility Console | `/facility` | SNF-level dashboard for Director of Nursing — all patients, acuity, staffing | done |
| Shift Handoff | `/shift-handoff` | Priority-ranked patient briefing for telemedicine shift changes | done |
| Post-Acute Guardian | `/guardian` | Live wearable monitoring for high-risk post-discharge patients | done |
| Readmission Risk | `/readmission` | Predictive readmission scoring with intervention recommendations | done |

### Revenue Capture
| Section | Route | Description | Status |
|---------|-------|-------------|--------|
| CCM Time Tracker | `/ccm` | Automated tracking of non-face-to-face clinical time (CPT 99490) | done |
| RPM Revenue Dashboard | `/rpm` | RPM device billing with CPT 99453-99458 auto-capture | done |
| Care Gap Scanner | `/care-gaps` | Population-level care gap detection with auto-outreach | done |

### Data Intelligence
| Section | Route | Description | Status |
|---------|-------|-------------|--------|
| Discharge Reconciliation | `/discharge` | SNF-to-Hospital semantic data bridge — discrepancy reports | done |
| Care Plan Optimizer | `/care-plans` | AI-generated evidence-based clinical recommendations | done |
| Staffing Optimizer | `/staffing` | Dynamic provider scheduling across facilities | done |

### Enterprise & Governance
| Section | Route | Description | Status |
|---------|-------|-------------|--------|
| ACO REACH Performance | `/aco-reach` | Quality measure tracking for value-based care contracts | done |
| PE Executive Dashboard | `/executive` | Board-level KPIs for Amulet Capital ($2.7B AUM) reporting | done |
| Credentialing Center | `/credentialing` | Multi-state physician licensing and CAQH management (21 states) | done |

---

## Agent Architecture Additions

### Agent 4: Post-Acute Guardian Agent (NEW)
- **Module:** F (Patient Engagement) + B (Provider Workflow)
- **Purpose:** Continuous monitoring of post-discharge high-risk patients via wearables
- **Trigger:** Incoming FHIR Observation from Device Bridge (weight gain, SpO2 drop, HRV change)
- **Flow:** ADR-007 (device data) → ADR-006 (context rehydration) → ADR-001 (FHIR record pull) → ADR-003 (LangGraph triage) → ADR-008 (A2A alert to nurse)
- **Kill Shot Example:** CHF patient gains 3lbs in 48 hours + 15% sleep quality drop → Agent sends P2 alert to on-site nurse with Furosemide recommendation
- **Revenue Impact:** Prevents $15-25K rehospitalization cost, protects ACO REACH shared savings

### Agent 5: CCM Revenue Agent (NEW)
- **Module:** C (Revenue Cycle) + F (Patient Engagement)
- **Purpose:** Automatically track all non-face-to-face clinical time and generate CCM/RPM billing claims
- **Flow:** EventBridge (API actions on patient chart) → Redis timeseries → Monthly aggregation → When 20min threshold crossed → A2A to Revenue Cycle Agent → Drop CPT 99490 claim
- **Revenue Impact:** Captures $60-150/patient/month in high-margin revenue typically left on the table

### Agent 6: Shift Summary Agent (NEW)
- **Module:** B (Provider Workflow)
- **Purpose:** Generate priority-ranked patient briefings during telemedicine shift handoffs
- **Flow:** Dr. A logs off → Agent scans all active patients across 50+ facilities → Generates briefing with pending labs, degrading vitals, medication changes → Presents to Dr. B at login
- **Clinical Impact:** Prevents critical information loss during shift changes, a leading cause of adverse events

### Agent 7: ACO REACH Quality Agent (NEW)
- **Module:** E (Population Health)
- **Purpose:** Proactively close care gaps to maximize VBC quality scores
- **Flow:** Continuous FHIR population scan → Identify care gaps (overdue HbA1c, missing BP readings) → A2A to Patient Communication Agent → Automated outreach (SMS/call) → Schedule appointment
- **Revenue Impact:** Directly impacts shared savings in ACO REACH contracts
- **Dossier Detail:** "Patient Jane Doe at Facility X is overdue for her Advance Care Plan discussion. Schedule a 15-minute telemedicine visit via ChatEasy to complete CMS Measure ID #123. Failure to complete by Friday will negatively impact shared savings by an estimated $1,200."

### Agent 8: SNF-to-Hospital Semantic Data Bridge (NEW — from Dossier)
- **Module:** H (Integration) + A (Patient Identity)
- **Purpose:** Solve the original sin Dr. Di Rezze discovered — the data chasm at patient transitions between hospitals and SNFs
- **Trigger:** Patient discharge from hospital back to SNF
- **Flow:** Secure data connection established → NLP ingests unstructured discharge summary (PDF/TXT/scanned) → Semantic reconciliation against pre-hospitalization ChartEasy record → Generates "Discrepancy Report" (new meds, changed dosages, new diagnoses, follow-ups)
- **Clinical Impact:** Prevents medication errors and ensures safe transitions — the exact problem that inspired Theoria's founding
- **Integration:** Reads from hospital FHIR endpoints → normalizes to MedOS FHIR store → compares with ChartEasy data

### Agent 9: Generative Care Plan Optimizer (NEW — from Dossier)
- **Module:** B (Provider Workflow) + E (Population Health)
- **Purpose:** AI "super-consultant" that scales the expertise of top clinicians across the entire Theoria network
- **Flow:** Analyzes longitudinal patient data (vitals, labs, nursing notes, RPM data) → Synthesizes with latest evidence-based clinical guidelines → Generates proactive clinical recommendations phrased as consults
- **Kill Shot Example:** "Patient John Smith's weight is up 4 lbs in 48 hours, and RPM data shows nocturnal O2 desaturation. This pattern is 85% predictive of a CHF exacerbation within 72 hours. Recommend: 1. Administering 40mg Lasix IV Push. 2. Scheduling a stat telemedicine check-in. 3. Initiating a daily weight protocol."
- **Confidence:** >= 0.90 for evidence-matched recommendations, ALWAYS human approval for medication changes

### Agent 10: Dynamic Staffing & Resource Allocation Agent (NEW — from Dossier)
- **Module:** B (Provider Workflow) + F (Patient Engagement)
- **Purpose:** Optimize Theoria's hybrid workforce (on-site + telemedicine) across national footprint
- **Flow:** Integrates scheduling + census + patient acuity data → Predicts patient surge needs at specific facilities → Identifies most cost-effective provider to dispatch (travel time + patient needs) → Dynamically adjusts telemedicine schedules for after-hours call volume
- **Business Impact:** Maximizes workforce efficiency, reduces provider burnout, controls operational costs — directly addresses Amulet Capital's mandate for margin expansion
- **Revenue Impact:** Reduces locum tenens costs ($2-5K/day) through optimized scheduling

---

## FHIR Resource Additions

| Resource | Use Case | Priority |
|----------|----------|----------|
| `CarePlan` | Required for CCM billing (CPT 99490 requires active care plan) | HIGH |
| `Device` / `DeviceMetric` | RPM device registration and metrics | HIGH |
| `Communication` | Track async clinician-patient interactions for CCM time tracking | HIGH |
| `CareTeam` | Multi-provider team management across facilities | MEDIUM |
| `Location` | Facility-level data (SNF, ALF, hospital identification) | MEDIUM |
| `Organization` | Facility organization hierarchy for multi-site operations | MEDIUM |

---

## Infrastructure Additions

| Component | Purpose | ADR Link |
|-----------|---------|----------|
| Twilio/WhatsApp Integration | Patient communication channel for Guardian/Care Gap agents | Module F |
| Payer Rules Vector DB | pgvector namespace for ingesting payer clinical guidelines (RAG) | ADR-001 |
| Facility-Level RLS | Row-Level Security at facility level within tenant schemas | ADR-002 |
| Agent Evaluation Framework | Clinical safety grading before agent actions reach clinicians | ADR-003 |
| WebRTC Audio Pipeline | Real-time telemedicine transcription via Whisper sidecar | Module B |
| EventBridge CCM Rules | API action tracking → Redis timeseries → CCM billing threshold | Module C |

---

## Tasks

| ID | Task | Estimated Lines | Status |
|----|------|-----------------|--------|
| S4F-T01 | Facility Console page (SNF dashboard, patient grid, acuity indicators) | ~807 | done |
| S4F-T02 | Shift Handoff page (priority briefing, pending items, vitals timeline) | ~739 | done |
| S4F-T03 | Post-Acute Guardian page (live monitoring, wearable feeds, alert queue) | ~768 | done |
| S4F-T04 | Readmission Risk page (predictive scoring, risk factors, interventions) | ~906 | done |
| S4F-T05 | CCM Time Tracker page (patient list, time aggregation, billing threshold) | ~633 | done |
| S4F-T06 | RPM Revenue Dashboard (device billing, CPT tracking, monthly revenue) | ~658 | done |
| S4F-T07 | Care Gap Scanner page (population grid, gap types, outreach status) | ~792 | done |
| S4F-T08 | ACO REACH Performance page (quality measures, benchmarks, trends) | ~955 | done |
| S4F-T09 | PE Executive Dashboard (board KPIs, facility comparison, financial roll-up) | ~924 | done |
| S4F-T10 | Credentialing Center page (multi-state licensing, CAQH, expiration alerts) | ~1184 | done |
| S4F-T11 | Discharge Reconciliation page (hospital-SNF bridge, discrepancy reports, med changes) | ~711 | done |
| S4F-T12 | Care Plan Optimizer page (AI recommendations, evidence citations, approval flow) | ~636 | done |
| S4F-T13 | Staffing Optimizer page (provider scheduling, facility census, cost analysis) | ~753 | done |
| S4F-T14 | Agent architecture docs (7 new agents: Guardian, CCM, Shift, Quality, Bridge, CarePlan, Staffing) | ~600 | done |
| S4F-T15 | FHIR resource additions (CarePlan, Device, DeviceMetric, Communication, CareTeam, Location schemas) | ~300 | done |
| S4F-T16 | E2E test update for all 13 new pages | ~100 | done |
| S4F-T17 | E2E video recording with all new pages | ~50 | planned |
| S4F-T19 | Demo flow script for Dr. Di Rezze pitch (5-ACT stage directions) | ~200 | done |
| S4F-T20 | Pitch strategy update with live demo URLs and deployment status | ~100 | done |
| S4F-T18 | Master plan & vault documentation updates | ~200 | done |

---

## Acceptance Criteria

- [x] All 13 new pages render with Theoria-specific mock data (SNF names, post-acute patients, VBC metrics)
- [x] Facility Console shows multi-facility view with real SNF-style data
- [x] Shift Handoff generates priority-ranked briefings
- [x] Guardian monitoring shows live wearable data streams with alert thresholds
- [x] CCM tracker accurately models 20-minute billing threshold
- [x] RPM dashboard shows CPT 99453-99458 revenue tracking
- [x] Care Gap Scanner identifies gaps per HEDIS/ACO REACH measures
- [x] ACO REACH dashboard shows quality measure performance
- [x] PE Executive Dashboard provides Amulet Capital-grade reporting
- [x] Credentialing Center manages multi-state licensing
- [x] TypeScript compiles with 0 errors
- [x] E2E test covers all new routes (ACT 14, 15-ACT demo)
- [x] Discharge Reconciliation shows hospital-SNF transition with discrepancy detection
- [x] Care Plan Optimizer generates evidence-based AI recommendations
- [x] Staffing Optimizer shows dynamic provider scheduling across facilities
- [ ] Agent architecture for 7 new agents fully documented (Camino 2 — planned)
- [x] Mock data includes Theoria-specific details: ChartEasy references, Michigan SNF names, Empassion Health/ACO REACH metrics

---

## Estimated Size

~10,150 lines across 16 files (13 pages + agent docs + FHIR schemas + E2E test)

---

## Strategic Impact

### For the Pitch
This sprint transforms MedOS from a "generic healthcare OS" into a **purpose-built engine for Theoria Medical's exact operational model**. When Dr. Di Rezze sees his facility names, his patient types, his billing codes, and his PE reporting needs reflected in a working platform — that's the moment the deal closes.

### For the Platform
Everything built here is reusable. The Facility Console works for any multi-site practice. The CCM Tracker works for any practice doing chronic care. The Guardian Agent works for any RPM program. Theoria is the wedge; the platform is the moat.

### Financial Narrative
| Theoria Category | Current Multiple | With MedOS | Value Delta |
|-----------------|-----------------|------------|-------------|
| Physician Services | 8-12x EBITDA | — | Baseline |
| Tech-Enabled Services | — | 12-18x EBITDA | +50-100% |
| Platform-as-a-Service | — | 15-25x EBITDA | +87-200% |

If Theoria's EBITDA is $50M, the difference between 10x and 20x is **$500M in enterprise value**. MedOS is the bridge.
