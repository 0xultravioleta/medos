---
type: master-plan
date: "2026-02-27"
status: active
tags:
  - master-plan
  - strategy
  - phase-1
  - healthcare
aliases:
  - Master Plan
  - MedOS Plan
---

# Healthcare OS - Master Pl hi lan
## AI-Native Operating System for U.S. Healthcare
### Family Office | $100M | Miami, FL

> Prepared: 2026-02-27
> For: Dr. Di Reze - Technical Operations Pitch

---

## EXECUTIVE SUMMARY

**The Opportunity:** U.S. healthcare is a $5.6 TRILLION market with $496 BILLION in annual administrative waste. AI in healthcare is growing 41.5% YoY ($56B in 2026). No one has built an AI-native platform from the ground up -- every incumbent is bolting AI onto 20-year-old architectures.

**The Vision:** Build the "Stripe of Healthcare" -- not a better EHR, but the AI-native infrastructure layer that connects providers, payers, and patients. Every workflow powered by AI agents. Natural language as the primary interface. A system that gets smarter with every interaction.

**The Bet:** $100M of patient family office capital, deployed over 36 months, targeting $36M ARR and 300-400 practices by month 36. Valuation trajectory: $250-500M+.

**The Edge:** AI-native architecture (not AI bolted on), patient capital (not VC pressure), and a 2-person founding team that can ship like 15 using frontier AI development tools.

---

## 1. WHY NOW - THE CONVERGENCE

### Regulatory Tailwinds
- **CMS Interoperability Rules** mandate FHIR APIs -- breaking incumbent lock-in
- **21st Century Cures Act** makes information blocking illegal ($1M/violation)
- **HIPAA Security Rule Update (Jan 2025)** forces modernization -- AES-256, MFA mandatory, no more "addressable" vs "required"
- **CMS Prior Auth Reform** mandates electronic PA by 2026-2027
- **TEFCA** creating national health data exchange network

### Technology Maturity
- LLMs now reliable enough for clinical-adjacent tasks (JAMA-validated)
- AI inference costs dropped 90%+ since 2023
- FHIR R4 widely adopted -- integration finally feasible
- MCP (Model Context Protocol) standardizing AI-to-tool connections
- FHIR-MCP Server already exists open source

### Market Demand
- Post-COVID digital adoption permanent
- Staff shortages acute -- practices CANNOT hire enough people
- Burnout crisis: for every hour with patients, doctors spend 2 on paperwork
- Payer-Provider AI arms race: payers use AI to deny faster, providers NEED AI to fight back
- 41% of providers report denial rates >10% (up from 30% in 2022)

---

## 2. WHAT "HEALTHCARE OS" MEANS

### The Analogy
- **iOS/Android** abstracted phone hardware, provided shared services, enabled an app ecosystem
- **Healthcare OS** abstracts healthcare complexity, provides shared data/AI services, enables a health app ecosystem

### NOT Another EHR
| Epic (Product) | Healthcare OS (Platform) |
|---|---|
| $500M implementations | API-first, self-service onboarding |
| 18-month deployments | Days to integrate |
| Locked-in customers | Open, interoperable |
| AI bolted on | AI is the kernel |
| Human types, AI suggests | AI generates, human reviews |
| Static rules updated quarterly | Continuous learning |
| Form-based input | Natural language input |

### The "Stripe for Healthcare" Model
Stripe didn't build a better bank. It built the payment infrastructure layer. We build the healthcare operations infrastructure layer that lets every practice operate like the Mayo Clinic.

---

## 3. ARCHITECTURE - THE 8 MODULES

```
APPLICATION LAYER
  Provider Workspace | Patient Portal | Payer Portal | Admin Console | 3rd Party Apps

AI ORCHESTRATION LAYER
  Agent Framework | Clinical NLP | Predictive Models | Decision Support

SERVICE MODULES
  A. Patient Identity    E. Population Health
  B. Workflow Engine     F. Patient Engagement
  C. Revenue Cycle       G. Compliance Engine
  D. Payer Integration   H. Integration Layer

DATA LAYER
  FHIR Resource Store (PostgreSQL + JSONB) | Event Store (Kafka) | Analytics (Snowflake)

INTEGRATION LAYER
  FHIR R4 Gateway | HL7v2 Adapter | X12 EDI Engine | Device Bridge | MCP Servers
```

### Module A: Patient Identity & Data Layer (The Kernel)
- Universal patient identity via probabilistic matching (no national ID exists)
- FHIR-native data model -- NOT "translatable to FHIR" but natively FHIR
- Consent management as machine-executable policy (evaluated at query time)
- PostgreSQL 17 + JSONB + pgvector (one DB for relational + FHIR + vectors)
- **Context Rehydration Engine** (see [[ADR-006-patient-context-rehydration]]): Event-driven context refresh when patient data changes — ensures AI agents always operate on fresh data

### Module B: Provider Workflow Engine
- **Ambient AI Documentation**: Audio capture -> speech-to-text -> clinical NLU -> note generation -> provider review
- Proven results: JAMA study showed burnout dropped from 51.9% to 38.8%, saves 30 min/day
- Smart scheduling with ML-predicted no-shows and overbooking optimization
- CDS Hooks integration for clinical decision support
- **Context Freshness Monitor**: Ensures all AI agent contexts stay fresh — staleness detection (score < 0.75 triggers rehydration), exponential time decay, cosine similarity validation against golden source (EMR)

### Module C: Revenue Cycle (WHERE THE MONEY IS)
- AI coding engine: ICD-10 + CPT from clinical notes (10-20% error rate currently)
- Real-time eligibility (X12 270/271)
- Prior auth automation (X12 278 + Da Vinci PAS)
- Claims scrubbing with AI-predicted denial probability
- AI denial management: 65% of denials are never appealed, 50-70% of appeals win
- Revenue impact: $1-3M/year recovered per 200-provider practice

### Module D: Payer Integration
- Built-in clearinghouse (eliminates $0.25-$1.00/claim third-party fees)
- Full X12 EDI support (270/271, 837, 835, 276/277, 278)
- Contract modeling and underpayment detection
- Da Vinci FHIR-based payer exchange

### Module E: Population Health & Analytics
- HCC risk scoring (CMS-HCC v28)
- Readmission prediction (LACE+ with SDOH factors)
- Care gap identification (HEDIS, MIPS measures)
- Network-wide benchmarking

### Module F: Patient Engagement
- Multi-channel: web portal, mobile app, SMS, email, voice
- AI chat agent (triage, scheduling, billing Q&A -- NOT diagnosis)
- Telehealth integration (Twilio Video)

#### Wearable & IoT Device Integration (see [[ADR-007-wearable-iot-integration]])
- **Device Bridge**: Unified ingestion layer for consumer health devices
  - Apple Watch (HealthKit API): HR, HRV, SpO2, ECG, fall detection, activity
  - Oura Ring (Oura Cloud API): HR, HRV, sleep stages, temperature, readiness
  - Dexcom CGM (Dexcom API): Continuous glucose monitoring, trend arrows
  - Google Health Connect: Aggregated Android device data
  - Withings (Health Mate API): Blood pressure, weight, sleep
  - Fitbit (Web API): Activity, sleep, HR
- **FHIR Observation mapping**: Every device reading stored as FHIR R4 Observation with LOINC codes
- **Device MCP Server**: 8 tools for agent access (register, list, ingest, query, alerts, deregister)
- **Patient consent**: Explicit opt-in per device, granular data sharing, right to disconnect
- **Alert thresholds**: Configurable per-patient (HR >120/<45, SpO2 <92, glucose >180/<70)
- **Revenue impact**: RPM (CPT 99453-99458) generates $100-150/patient/month

### Module G: Compliance & Security
- SMART on FHIR OAuth2
- RBAC + ABAC + break-the-glass with justification
- Immutable audit trail (FHIR AuditEvent, 6-year retention)
- AES-256 at rest, TLS 1.3 in transit, field-level encryption for SSN
- Celebrity patient snooping detection

### Module H: Integration & Interoperability
- FHIR R4 API Gateway (CRUD, search, bulk export, subscriptions)
- HL7v2 adapter (95% of hospitals still use HL7v2)
- Third-party app marketplace (SMART on FHIR)
- Developer portal + SDK (Python, JavaScript, C#)

#### A2A Protocol (Inter-Agent Communication)
- **Agent-to-Agent (A2A) Protocol** adopted for inter-agent communication (see [[ADR-008-a2a-agent-communication]])
- **MCP = tool access** (agent -> FHIR data, billing, scheduling tools), **A2A = agent communication** (agent <-> agent coordination)
- Every MedOS agent exposes an **A2A Agent Card** at a dedicated endpoint, enabling capability discovery by internal and external agents
- **A2A Gateway** handles auth, PHI screening, tenant isolation, and audit logging for all inter-agent messages
- **External agent marketplace**: Third-party healthcare AI agents (EHR vendors, payers, health tech) connect to MedOS via A2A Agent Cards
- **HIPAA-compliant PHI screening**: All A2A messages pass through minimum necessary enforcement -- agents only receive data matching their PHI access level
- **Task lifecycle**: Inter-agent requests are A2A Tasks with states (submitted, working, input-required, completed, failed, canceled) providing full auditability
- A2A complements the existing event bus (Redis Streams): event bus for fire-and-forget notifications, A2A for workflows requiring acknowledgment, status tracking, or multi-turn interaction

#### Device Bridge Architecture
- **Inbound adapters**: One adapter per device API (Apple HealthKit, Oura Cloud, Dexcom, Google Health Connect)
- **Normalization layer**: Raw device data → FHIR R4 Observation with LOINC coding
- **Event publishing**: Every device reading → event bus → context rehydration triggers
- **Webhook receivers**: For push-based APIs (Dexcom real-time glucose, Withings webhooks)
- **Batch sync**: For pull-based APIs (Apple HealthKit via mobile app sync)

#### Patient Context Rehydration (see [[ADR-006-patient-context-rehydration]])
- **Event-driven refresh**: When ANY patient data changes, publish event → downstream contexts refresh
- **Context Dependency Graph**: Maps which AI contexts depend on which data sources
- **Tiered cache**: Redis (hot, TTL 15min) → Vector embeddings (warm) → PostgreSQL JSONB (cold/golden source)
- **Freshness monitoring**: Scores 0.0 (stale) to 1.0 (fresh), threshold 0.75, cosine similarity check
- **Auto-refresh policies**: Immediate (active encounter), Soon (background), Batch (analytics), Lazy (dormant)
- **Golden source pattern**: EMR is ALWAYS truth — everything in Redis/vector is disposable cache

---

## 4. TECH STACK

| Layer | Technology | Why |
|---|---|---|
| Backend | FastAPI (Python 3.12+) | AI/ML ecosystem, FHIR libraries, async, auto-docs |
| Frontend | Next.js 15 | Server Components for PHI safety |
| Primary DB | PostgreSQL 17 + pgvector | Single DB: relational + FHIR JSONB + vectors |
| Time-series | TimescaleDB | Vital signs, lab trends |
| Cache | Redis 7+ | Sessions, consent matrix, pub/sub |
| Event Bus | Kafka / EventBridge | Integration events, AI pipeline |
| LLM | Claude API (HIPAA BAA) | Best clinical reasoning, safety |
| Agent Framework | LangGraph | State machine agents, observability |
| Integration | MCP (Model Context Protocol) | FHIR-MCP Server available |
| Speech-to-Text | Whisper v3 (self-hosted GPU) | Cost, latency, data sovereignty |
| Cloud | AWS (HIPAA BAA signed) | Most mature HIPAA services |
| IaC | Terraform | Never CloudFormation |
| Multi-tenancy | Schema-per-tenant + per-tenant KMS | Strong isolation for healthcare |
| Compliance Automation | Vanta/Drata | SOC 2 continuous monitoring |
| CI/CD | GitHub Actions | AI-augmented pipeline |

### Monthly Tool Budget (2-Person Team)
| Tool | Cost/month |
|---|---|
| Claude Code Max (2 seats) | $400 |
| Cursor Pro (2 seats) | $40 |
| Claude API (production) | $200-500 |
| AWS Infrastructure | $500-1,500 |
| Vanta (compliance) | $800 |
| GitHub Team | $8 |
| Langfuse (LLM observability) | $0-59 |
| Auth0 (HIPAA BAA) | $0-240 |
| **TOTAL** | **$2,000-3,500** |

---

## 5. AI-NATIVE DIFFERENTIATORS

### What Makes This Different from "Epic + ChatGPT"

| Dimension | Epic + AI Plugins | Healthcare OS (AI-Native) |
|---|---|---|
| Primary input | Form fields, templates | Natural language (voice, text) |
| Documentation | Human types, AI suggests | AI generates, human reviews |
| Coding | Human codes, AI checks | AI codes, human validates |
| Scheduling | Rule-based | ML-optimized (predicted demand) |
| Billing | Batch processing | Real-time, predictive |
| Decision support | Alert-based (interruptive) | Ambient (contextual) |
| Learning | Static rules, quarterly updates | Continuous learning per interaction |
| Data model | Structured-first (1990s design) | Multimodal (structured + unstructured + temporal) |

### AI Agent Architecture (Bounded Autonomy)
Every agent has:
- **Defined authority** (what it can do)
- **Hard constraints** (what it cannot do)
- **Escalation paths** (when to ask a human)
- **Confidence thresholds** (below 0.85 = human review)
- **Full audit trail** (every action logged as FHIR Provenance)

Agents:
1. **Prior Auth Agent** - auto-submits with clinical justification
2. **Denial Management Agent** - drafts appeals, resubmits claims
3. **Patient Communication Agent** - reminders, FAQ, triage
4. **Referral Coordination Agent** - finds specialists, checks networks
5. **Quality Reporting Agent** - care gaps, measure tracking

### Data Moat & Flywheel
```
More providers -> More clinical data -> Better AI models
-> Better outcomes -> More providers sign up -> REPEAT

Cross-network effects:
- Benchmarking ("your readmission rate is 18%, top quartile is 11%")
- Coding intelligence ("payer X denies code Y 34%, use code Z instead")
- Referral optimization ("this specialist has best outcomes + shortest wait")
```

---

## 6. GO-TO-MARKET STRATEGY

### Target: Mid-Size Specialty Practices (5-30 Providers) in Florida

| Segment | Deal Size | Sales Cycle | Verdict |
|---|---|---|---|
| Solo/small (1-4) | $500-2K/mo | 2-4 weeks | Too small, high churn |
| **Mid-size specialty (5-30)** | **$3-15K/mo** | **4-8 weeks** | **SWEET SPOT** |
| Large multi-specialty (30-100) | $15-50K/mo | 3-6 months | Phase 2 |
| Health systems (100+) | $100K-500K+/mo | 6-18 months | Phase 3 |

### Why Florida First
- Home market (Miami office, relationships)
- 22M+ residents, huge Medicare population, 400+ hospitals
- Less competitive than CA/Northeast for health tech
- No state income tax (recruiting advantage)
- Proximity to Latin American markets for future expansion

### Why Specialty Practices
- Underserved by Epic/Cerner (who focus on hospitals)
- Practice manager is the decision-maker (moves fast)
- Pain points are acute (prior auths, coding, staffing)
- Willing to pay for time savings (staff is #1 expense)

### Recommended First Verticals

**Vertical 1: Mid-Size Specialty (Orthopedics or Dermatology)**
High volume, procedure-heavy, lots of prior auth pain, tech-forward physicians.

**Vertical 2: Post-Acute & Value-Based Care (SNFs, Assisted Living)**
Theoria Medical-style operations: distributed telemedicine, CCM/RPM billing, ACO REACH contracts.
See [[EPIC-016-theoria-medical-pilot]] and Section 16 for full Theoria strategy.

### Pricing
**Tier 1 - Core:** $2,000-5,000/mo (scheduling, patient comms, basic analytics)
**Tier 2 - Advanced:** $5,000-15,000/mo (+ AI documentation, coding, prior auth, RCM)
**Tier 3 - Enterprise:** Custom $15,000-50,000+/mo (multi-location, custom integrations, API access)

**Transaction add-ons:**
- Claims: $1-3/claim
- Prior auth: $5-15/auth
- Payment processing: 2.5-3%

### Pilot Program (The Key to Winning)
- 90 days, free of charge
- One department or workflow (e.g., prior auth for ortho)
- Clear success metrics upfront:
  - Time saved: 30+ min/provider/day
  - Prior auth approval rate: +10-15%
  - Denial rate reduction: -20-30%
  - Revenue per encounter: +5-10%
- Present interim results at day 45
- Full ROI presentation at day 75
- Target: 60-70% pilot-to-paid conversion

---

## 7. CAPITAL DEPLOYMENT ($100M / 36 Months)

### Family Office Advantage (vs VC)
| Dimension | VC-Backed | Family Office |
|---|---|---|
| Timeline | 7-10 year fund, need exits | Generational, can wait |
| Growth | Hypergrowth or die | Sustainable growth OK |
| Burn rate | "Spend fast to win" | Capital preservation matters |
| Pivot flexibility | Constrained by thesis | Freedom to iterate |
| Exit pressure | IPO by Year 7 | Can build profitable business |

### Phase 1: Foundation (Months 0-12) -- $8-12M
| Category | Budget |
|---|---|
| Team (2 -> 8 people) | $2.5-3.5M |
| Infrastructure & tooling | $500-800K |
| Office & operations | $300-500K |
| Legal & compliance | $500K-1M |
| Clinical advisory board | $200-300K |
| Customer pilots | $300-500K |
| Contingency | $500K-1M |

### Phase 2: Product-Market Fit (Months 12-24) -- $25-35M
| Category | Budget |
|---|---|
| Team (8 -> 25-30) | $8-12M |
| SOC 2 Type II + HITRUST | $1-2M |
| Sales & marketing | $3-5M |
| Product development | $3-5M |
| Customer onboarding | $2-3M |
| Infrastructure scale | $2-3M |
| Partnerships | $1-2M |
| Contingency | $2-3M |

### Phase 3: Market Expansion (Months 24-36) -- $30-45M
| Category | Budget |
|---|---|
| Team (30 -> 50) | $12-18M |
| Market expansion | $5-8M |
| Enterprise sales | $5-8M |
| Product platform | $3-5M |
| Strategic acquisitions | $5-10M |
| Contingency | $3-5M |

### Strategic Reserve: $15-25M
The family office advantage -- extend runway indefinitely, never look desperate, opportunistic acquisitions.

### Burn Rate Targets
- Months 0-6: $400-600K/month
- Months 6-12: $600K-1M/month
- Months 12-24: $1.5-2.5M/month
- Months 24-36: $2.5-3.5M/month

**Rule: Always maintain 18+ months of runway at current burn.**

---

## 8. REVENUE PROJECTIONS

| Month | MRR | ARR | Customers | Team |
|-------|-----|-----|-----------|------|
| 6 | $30K | $360K | 5 pilots | 6-8 |
| 12 | $100K | $1.2M | 15-20 | 8-10 |
| 18 | $300K | $3.6M | 40-50 | 20-25 |
| 24 | $700K | $8.4M | 80-100 | 25-30 |
| 30 | $1.5M | $18M | 150-200 | 35-40 |
| 36 | $3M | $36M | 300-400 | 45-55 |

### Revenue Streams (Layered Over Time)
1. **SaaS Subscriptions** (Day 1) - Foundation, predictable
2. **Transaction Fees** (Month 6+) - Claims, prior auth, payments
3. **Marketplace Revenue** (Month 18+) - 20% take-rate on third-party apps
4. **Data Insights** (Month 24+) - De-identified analytics, high-margin
5. **Value-Based Care Shared Savings** (Month 24+) - Aligned incentives

### Unit Economics Target
- ACV: $60-120K/year per mid-size practice
- Gross margin: 75-85%
- Logo churn: <10% annually (healthcare is sticky)
- Net revenue retention: 110-130%

---

## 9. HIRING STRATEGY

### Team of 2 -> AI Multiplier Effect
A 2-person team with AI tools (Claude Code + Cursor) produces what 10-15 traditional engineers can. The "studio engineer" model: one person runs the board (ops, CI/CD, QA, infra), the other creates (architecture, core product, AI).

### Hiring Order (After the First 2)
| # | Role | When | Why |
|---|---|------|-----|
| 3 | Healthcare Domain Expert / Clinical Advisor | Month 3-4 | Can't build for healthcare without someone who lived it |
| 4 | Senior Full-Stack Engineer | Month 5-6 | Second builder |
| 5 | Compliance & Security Lead | Month 6-8 | HIPAA is non-negotiable |
| 6 | Product Designer (Healthcare UX) | Month 8-10 | Healthcare UX is specialized |
| 7 | Sales Engineer | Month 10-12 | Bridges tech and customers |
| 8 | Data Engineer (FHIR/HL7) | Month 12-14 | Integration complexity |
| 9 | Customer Success Lead | Month 14-16 | Retention is everything |
| 10 | ML Engineer | Month 16-18 | Scale AI capabilities |

### Miami Talent Strategy
- First 5-6 from network or relocate (tax advantage helps)
- Remote-first engineering, Miami HQ for clients + compliance
- Recruit aggressively from LatAm (Colombia, Argentina, Brazil -- 40-60% US costs)
- Core team: in-office 3-4 days/week
- Scale-up engineers: remote with quarterly gatherings

---

## 10. REGULATORY & COMPLIANCE ROADMAP

### What to Do When
| Timeline | Action | Cost |
|---|---|---|
| Day 1 | BAA with AWS, encryption everywhere, MFA, audit logging | Included in infra |
| Day 1 | Cyber liability insurance ($1-5M coverage) | $5-15K/year |
| Day 1 | Healthcare attorney on retainer | $15-25K/year |
| Month 1-3 | HIPAA risk assessment, policies, training | $50-100K |
| Month 3-6 | Third-party pen testing | $30-50K |
| Month 6 | Begin SOC 2 Type I | $50-100K |
| Month 6-8 | Hire full-time Compliance Lead | $150-200K salary |
| Month 8-14 | SOC 2 Type II observation period | Part of audit |
| Month 12 | Begin HITRUST assessment | $150-300K |
| Month 18-24 | HITRUST certification | Part of assessment |

### FDA: Likely NOT Required (Initially)
- Administrative AI (billing, scheduling, prior auth) = NOT a medical device
- Clinical documentation where clinician reviews = NOT a medical device
- Autonomous clinical decisions = IS a medical device
- **Strategy: Stay administrative. Add clinical later with FDA 510(k).**

### State-by-State AI Regulation (47 states introduced bills in 2025)
- Texas (Jan 2026): Written disclosure for AI in care
- California: CCPA/CPRA + companion chatbot rules
- Illinois: Prohibits AI from independent therapeutic decisions
- 5 states restrict AI-based coverage denials
- **This complexity is itself a moat** for whoever solves it first

---

## 11. COMPETITIVE LANDSCAPE

### Incumbents (Vulnerable)
- **Epic**: 38% hospital market, not AI-native, closed ecosystem, expensive
- **Oracle Health (Cerner)**: Losing ground (lost 74 hospitals in 2024), post-acquisition chaos
- **athenahealth**: Cloud-native but PE-owned, innovation slowed

### AI-Native Point Solutions (Narrow)
- **Ambience Healthcare**: AI scribe only
- **Abridge**: Clinical documentation only
- **Cohere Health**: Prior auth only
- **AKASA**: Revenue cycle only

### The Gap
Nobody is building the **AI-native platform** -- the operating system where AI is the foundation. Point solutions solve one problem. Incumbents bolt AI onto old architectures. We build the connective tissue across ALL of healthcare operations.

### Positioning
**SAY:** "We give every 10-person practice the operational capability of the Mayo Clinic."
**DON'T SAY:** "We're replacing Epic." (We're not. We integrate alongside it.)

---

## 12. RISK ANALYSIS

| Risk | Severity | Mitigation |
|---|---|---|
| Regulatory changes | HIGH | Stay administrative (no FDA), compliance from Day 1, industry group membership |
| AI reliability (hallucinations in billing/coding) | HIGH | Human review, confidence scoring, model versioning, multi-model architecture |
| Incumbent response (Epic adds AI) | MEDIUM-HIGH | Don't replace EHR, integrate alongside. Target underserved mid-market |
| Key person dependency (2-person team) | HIGH | Documentation from Day 1, equity retention, clinical advisory board |
| Long sales cycles | MEDIUM | Pipeline 3x target, founder-led sales, standardize offering |
| Capital risk | LOW | Family office patience, 18+ months runway always, path to revenue |

---

## 13. THE PITCH TO DR. DI REZZE: THEORIA MEDICAL

### Strategic Context

Dr. Justin Di Rezze is not a typical prospect. He is:
- **CEO & Founder of Theoria Medical** -- a 21-state multispecialty physician services company focused on SNFs, Assisted Living, and hospital care
- **PE-Backed** -- recently partnered with Amulet Capital Partners for national scale
- **SPAC Principal** -- Praetorian Acquisition Corp (M&A mindset, not just operations)
- **Family Office** -- Di Rezze Family Office (capital allocation, long-term thinking)
- **VBC Leader** -- Active in ACO REACH value-based care contracts

**Why Theoria is the ideal MedOS customer:**
Theoria operates exactly the kind of distributed, high-complexity healthcare system that MedOS was built for. Their 24/7 telemedicine model across hundreds of facilities creates data fragmentation, billing leakage, and coordination challenges that only an AI-native OS can solve at scale.

### The Reframed Pitch (10 minutes)

**ACT 1: The Recognition (60 seconds)**
"You didn't just start a practice -- you diagnosed a systemic problem in post-acute care and built a company to fix it. Theoria grew from one SNF to a 21-state enterprise. The question now: how do you go from 21 states to 50 without operational complexity growing linearly?"

**ACT 2: The 4 Pain Points (90 seconds)**
1. **Data Fragmentation** -- Doctors covering 50+ facilities per shift, switching between PointClickCare and MatrixCare silos
2. **Revenue Leakage** -- CCM billing (CPT 99490) requires 20+ min tracking per patient per month. At scale, millions left on the table.
3. **Shift Handoff Risk** -- When the night doctor logs off, how does the next one know about the pending critical potassium at Sunrise SNF?
4. **PE Reporting Burden** -- Amulet needs readmission rates by state with variance analysis. Currently takes weeks of manual work.

**ACT 3: The Live Demo (3-4 minutes)**
Walk through https://medos-platform.vercel.app: Dashboard → AI Scribe → Claims → Approvals → Admin Hub → Devices → Context Freshness
Kill line: *"This platform -- 53 routes, 522 tests, 44 AI tools -- was built in 15 days by 2 people."*

**ACT 4: The Theoria Kill Shots (2-3 minutes)**
1. **Post-Acute Guardian** -- CHF patient gains 3lbs → Agent sends P2 alert to nurse → $20K rehospitalization prevented
2. **Zero-Touch CCM Revenue** -- Automatic time tracking → CPT 99490 auto-drops → $60-150/patient/month captured
3. **Shift Handoff** -- Priority-ranked briefing: "Patient Jones, pending potassium 5.8, trending up"
4. **PE-Ready Reporting** -- Natural language query → 60-second board report with charts and root cause analysis

**ACT 5: The Financial Argument (90 seconds)**
"Theoria at 10x EBITDA is a physician services company. Theoria + MedOS at 20x EBITDA is a technology platform. On $50M EBITDA, that's $500M in additional enterprise value."

**The Ask:**
"Give us 90 days. Read-only deployment in one Michigan facility. Zero disruption. If we don't find $500K+ in uncaptured annual revenue, we walk away."

### Detailed Pitch Strategy
See [[theoria-medical-pitch-strategy]] for full objection handling, pre-meeting checklist, and follow-up plan.

---

## 14. WHAT TO BUILD IN THE FIRST 90 DAYS

### Week 1-2: Foundation
- AWS infrastructure (Terraform, HIPAA BAA, encryption)
- CI/CD pipeline with AI-assisted testing
- Dev environment with Claude Code + Cursor
- HIPAA policies, risk assessment started
- FastAPI skeleton + FHIR resource models + PostgreSQL schema
- Auth system (Auth0/Clerk with HIPAA BAA)

### Week 3-6: Core MVP
- FHIR data ingestion pipeline + event bus
- Patient identity matching engine
- First AI agent: clinical documentation from audio
- Provider review interface (Next.js)
- AI coding engine (ICD-10 + CPT suggestions)
- Integration with 2-3 EHRs via FHIR APIs

### Week 7-10: Revenue Module
- Prior auth status tracking + basic automation
- Eligibility verification (X12 270/271)
- Basic claims generation
- AI denial prediction
- Analytics dashboard

### Week 11-13: Pilot Ready
- Security hardening + pen testing
- Pilot onboarding workflow
- Training materials
- Demo environment
- Integration testing

**Deliverable at 90 days:** A functional platform that 3-5 pilot practices can use for documentation, coding assistance, and basic revenue cycle -- with AI features that demonstrate the vision.

---

## 15. CURRENT PROGRESS (as of 2026-02-28)

### What's Been Built (Day 0-2)

#### Day 0: Research & Knowledge Base

Before writing a single line of code, we invested in understanding the domain deeply -- research first, decide second, build third.

- **Obsidian vault** with 44 documents totaling 121,860 words of structured healthcare knowledge
- **8 domain deep dives** researched in parallel using AI agents:
  - [[FHIR R4]] -- 14 core resources, JSON examples, PostgreSQL storage patterns (6,667 words)
  - [[HIPAA Compliance]] -- the 5 rules, 18 PHI identifiers, technical safeguards (6,182 words)
  - [[Revenue Cycle Management]] -- full claims lifecycle, $755K/year ROI model (5,458 words)
  - [[X12 EDI]] -- annotated 270/837P transactions (4,074 words)
  - [[Prior Authorization]] -- Da Vinci PAS, AI automation (2,761 words)
  - [[Clinical Workflows]] -- 12-step patient journey mapping (5,393 words)
  - [[AWS HIPAA Infrastructure]] -- Terraform modules, cost estimates (2,400 words)
  - [[SOC 2 HITRUST]] -- compliance certification roadmap (2,800 words)
- **4 Architecture Decision Records** documenting key technical decisions:
  - [[ADR-001-fhir-native-data-model]] -- FHIR-native JSONB storage (not relational translation)
  - [[ADR-002-schema-per-tenant]] -- Schema-per-tenant multi-tenancy with Row Level Security
  - [[ADR-003-langgraph-claude-agents]] -- LangGraph + Claude for AI agents with confidence scoring
  - [[ADR-004-fastapi-async-backend]] -- FastAPI backend with async-first architecture
- **117-task execution plan** across 7 sprints (90 days) with day-by-day granularity, explicit dependencies, and testable acceptance criteria
- **6 EPICs** with detailed task breakdowns: AWS Infrastructure, Authentication & Identity, FHIR Data Layer, AI Clinical Documentation, Revenue Cycle MVP, Pilot Readiness

#### Day 1-2: Platform Development

##### Backend (FastAPI)
- 28 Python modules following [[ADR-004-fastapi-async-backend]]
- FHIR Patient CRUD with in-memory store + [[ADR-001-fhir-native-data-model|FHIR-native JSONB]] repository layer ready
- JWT auth middleware (dev mode HS256, prod RS256 with JWKS)
- FHIR AuditEvent builder for immutable compliance audit trail
- PHI filter that automatically redacts all 18 HIPAA identifiers from logs
- Structured logging (JSON for prod/CloudWatch, console for dev)
- 7 REST API endpoints (`/api/v1/*`): patients, appointments, dashboard stats/activity, claims, AI notes
- Alembic migrations with [[ADR-002-schema-per-tenant|schema-per-tenant]] support and `provision_tenant()` SQL function
- 56 tests passing in 0.86s, 99% code coverage
- GitHub Actions CI pipeline (lint, test, security scan with Trivy + pip-audit, Docker build)

##### Frontend (Next.js 15)
- 11 routes (login + 10 dashboard views) with App Router, TypeScript, and Tailwind CSS
- Professional design system (MedOS brand colors, responsive from mobile to desktop)
- Interactive AI Scribe simulation at `/ai-notes/new` with 3 stages:
  1. Recording -- patient/visit type selectors, animated waveform, pulsing red dot, timer
  2. AI Processing -- 5 animated steps (transcribe, entities, SOAP, ICD-10, CPT)
  3. Generated Note -- SOAP note with typewriter effect, confidence badge (94%), suggested codes
- Dashboard with personalized greeting, 4 KPI stat cards, today's schedule, activity feed
- Patient list with search/sort and patient detail with clinical timeline + AI-generated SOAP note
- Claims management with CPT/ICD-10 codes, 4 KPI cards, payer tracking
- Analytics with charts, patient volume trends, top procedures by CPT code
- Settings with profile/practice configuration and preference toggles
- API client (`src/lib/api.ts`) with typed fetch functions and mock data fallback

##### Infrastructure
- Docker + docker-compose (PostgreSQL 17 with pgvector, Redis 7)
- Deployed to Vercel: https://medos-platform.vercel.app
- GitHub repo: https://github.com/0xultravioleta/medos-platform

### Masterplan Alignment

| Masterplan Section | Status | Notes |
|---|---|---|
| Section 3: 8 Modules | Module A (partial), B (demo), C (demo) | Patient Identity, Workflow Engine, Revenue Cycle have demo UIs |
| Section 4: Tech Stack | FastAPI, Next.js 15, PostgreSQL 17, Redis 7 implemented | Bedrock, Kafka, Snowflake pending |
| Section 5: AI Differentiators | AI Scribe simulated (3-stage interactive demo) | Production: Claude via Bedrock (HIPAA BAA) |
| Section 6: GTM | Research complete | Florida, orthopedics/dermatology target identified |
| Section 7: Capital Deployment | Phase 1 budget framework ready | Pending job confirmation |
| Section 10: Compliance | HIPAA baked in from Day 1 (PHI filter, audit events, RLS) | SOC 2 / HITRUST timeline on track |
| Section 14: 90-Day Build Plan | Week 1-2 (Foundation) ~60% complete | Backend + Frontend done, AWS infra pending |

### What's Next (Priority Order)

1. **AWS Infrastructure** (Terraform) -- Sprint 0 remaining tasks per [[03-projects/EPIC-001|EPIC-001]]
2. **Auth0 real integration** -- SMART on FHIR OAuth2 per [[ADR-002-schema-per-tenant]]
3. **Claude via Bedrock** -- real AI Scribe pipeline (LangGraph + Claude, HIPAA BAA)
4. **FHIR Encounter/Observation resources** -- expand beyond Patient CRUD
5. **Revenue cycle real integration** -- X12 EDI 270/271 eligibility, 837P claims
6. **Pilot practice onboarding** -- first 3-5 Florida specialty practices

### New Discovery: Claude for Healthcare (JPM26 Announcement)

On January 11, 2026, Anthropic announced at JPM26 (J.P. Morgan Healthcare Conference) that Claude is now HIPAA-ready for healthcare:

- **HIPAA BAA available** via AWS Bedrock, Google Vertex AI, and Microsoft Azure -- only frontier model on all 3 clouds with HIPAA compliance
- **EMR connectors** for direct integration with electronic medical records
- **Native support** for FHIR, ICD-10, CPT coding, and prior authorization workflows
- **Clinical reasoning** validated by leading health systems

This confirms Claude as our production AI backend. The architecture decision in [[ADR-003-langgraph-claude-agents]] to use Claude + LangGraph is directly aligned with Anthropic's healthcare strategy. We get enterprise-grade, HIPAA-compliant AI without building our own models.

---

## CURRENT PROGRESS

> Last updated: 2026-03-01 (Day 14 of development)

### What's Built

| Component | Status | Details |
|-----------|--------|---------|
| **Knowledge Base** | Complete | 86+ documents, 150K+ words across all healthcare domains |
| **Backend (FastAPI)** | Sprint 5 complete | 50+ Python modules, 522+ tests passing, JWT auth, FHIR CRUD, audit logging, PHI filter, field encryption, rate limiting |
| **Frontend (Next.js 15)** | 53 routes | Login, dashboard, patients, appointments, AI notes, claims, approvals, analytics, pilot, docs (5), settings (6), project, admin hub (13 sections), Theoria pilot (13 pages) |
| **Admin Hub** | Complete | 13 admin sections across 4 groups (Operations, Practice, Platform, Compliance), 9,732 lines, collapsible sidebar |
| **AI Scribe (Demo)** | Complete | Interactive 3-stage simulation + MCP tool-based pipeline with confidence scoring |
| **MCP Layer** | Complete | 6 MCP servers: FHIR (12), Scribe (6), Billing (8), Scheduling (6), Device (8), Context (4) = 44 tools |
| **LangGraph Agents** | 3 active, 5 designed, 9+ brainstormed | Clinical Scribe, Prior Authorization, Denial Management + Patient Communication, Quality Reporting designed |
| **Security Pipeline** | Complete | MCP Gateway, field-level encryption (Fernet, per-tenant PBKDF2), PHI filter, rate limiting, HIPAA 94/100 |
| **A2A Protocol** | Complete | Agent Cards, A2A Gateway, inter-agent tasks, PHI screening, external marketplace |
| **Approval Workflow** | Complete | Human-in-the-loop review for high-stakes agent actions |
| **Device Integration** | Complete | Device MCP Server (8 tools), Oura Ring, Apple Watch, Dexcom CGM, FHIR Observation mapping |
| **Context Rehydration** | Complete | Event-driven refresh (17 change types, 13 context types), freshness scoring, tiered cache |
| **E2E Demo** | Complete | 15-ACT Playwright test covering all 53 pages, auto-generates MP4 video |
| **CI/CD** | Configured | GitHub Actions with lint, test, security scan, Docker build |
| **Deployment** | Live | Frontend on Vercel, backend Docker-ready |
| **Architecture** | Complete | 8 ADRs, 16 EPICs, Terraform module plan, agent architecture, MCP plan, A2A reference |

### Sprint Status

| Sprint | Phase | Status |
|--------|-------|--------|
| **S0** | Foundation (AWS, CI/CD, Auth, DB, FastAPI, FHIR CRUD) | Complete |
| **S1** | FHIR resources, patient matching, event bus | Complete |
| **S2** | MCP SDK refactoring, 32 tools, 3 agents, HIPAAFastMCP | Complete |
| **S2.5** | Device MCP (8 tools), Context Rehydration, Freshness Monitor | Complete (522 tests) |
| **S3** | Demo polish: Approvals UI, WebSocket, agent runner, intake | Complete |
| **S3F** | Frontend: Settings pages, Project Tracker, Admin Hub (13 sections) | Complete (40 routes, 0 TS errors) |
| **S4** | Revenue cycle: X12 837P, scrubbing, 835 parser, PA, denials, analytics | Complete |
| **S5** | Security: field encryption, tenant onboarding, EHR bridge, monitoring | Complete |
| **S6** | Launch: Terraform, demo data, pilot metrics, runbooks | In progress |
| **S4F** | Theoria Medical Pilot Sprint: 13 new pages (4 domains), 7 agents designed | Complete (53 routes, 0 TS errors) |

### Key Metrics

- **53** frontend routes, 0 TypeScript errors
- **522+** backend tests, all passing, ruff clean
- **44** MCP tools across 6 servers
- **3** LangGraph agents active (5 designed, 9+ brainstormed)
- **9,732** lines of admin hub code built in a single night (4 parallel AI agents)
- **8** Architecture Decision Records
- **16** EPICs (EPIC-001 through EPIC-016)
- **86+** knowledge base documents
- **15-ACT** E2E demo covering every route (including Theoria pilot)
- **$402/mo** estimated dev environment cost (AWS)
- **$3,508/mo** estimated production cost (AWS)
- **14 days** total development time (2-person team + AI tools)

### Live Demo

- **URL:** https://medos-platform.vercel.app
- **Credentials:** `dr.direze@sunshinemedical.com` / `demo123`
- **GitHub:** https://github.com/0xultravioleta/medos-platform
- **Knowledge Base:** https://github.com/0xultravioleta/medos

### Documentation Index

- [[theoria-medical-pitch-strategy]] -- Pitch strategy for Dr. Di Rezze meeting
- [[EPIC-016-theoria-medical-pilot]] -- Theoria Medical pilot sprint (10 pages, 4 agents)
- [[demo-script-monday]] -- Demo walkthrough for Monday meeting
- [[terraform-module-plan]] -- 9 AWS Terraform modules with cost estimates
- [[agent-architecture]] -- 5 AI agents with bounded autonomy framework
- [[bedrock-claude-setup]] -- AWS Bedrock + Claude HIPAA setup
- [[mcp-integration-plan]] -- MCP integration strategy
- [[MOC-Agent-Architecture]] -- Agent documentation index
- [[ADR-006-patient-context-rehydration]] -- System-wide context rehydration architecture
- [[ADR-007-wearable-iot-integration]] -- Wearable/IoT device integration
- [[ADR-008-a2a-agent-communication]] -- A2A protocol adoption for inter-agent communication
- [[a2a-protocol-reference]] -- A2A protocol reference and MedOS integration guide
- [[EPIC-014-admin-system-monitoring]] -- Admin Phase 1 (Settings sub-pages)
- [[EPIC-015-admin-hub]] -- Admin Hub Phase 2 (13 sections, 9,732 lines)

---

## 16. STRATEGIC OPPORTUNITY: THEORIA MEDICAL & POST-ACUTE CARE

### The Thesis

Theoria Medical represents a transformative partnership opportunity. Dr. Di Rezze has built the clinical model (24/7 telemedicine across 21 states); MedOS provides the technology platform to take it national. This is not a customer sale -- it's a platform adoption that fundamentally changes Theoria's category from physician services (8-12x EBITDA) to tech-enabled platform (15-25x EBITDA).

### What MedOS Brings to Theoria

| Theoria Pain Point | MedOS Solution | ADR/Module |
|-------------------|----------------|------------|
| Multi-facility data fragmentation | FHIR-native data lake + Context Rehydration | [[ADR-001]], [[ADR-006]] |
| CCM/RPM revenue leakage ($60-150/patient/month) | CCM Revenue Agent + EventBridge time tracking | Module C, New Agent |
| 24/7 shift handoff risk | Shift Summary Agent (priority-ranked briefings) | Module B, New Agent |
| Rehospitalization penalties (ACO REACH) | Post-Acute Guardian Agent (wearable monitoring) | [[ADR-007]], New Agent |
| PE reporting burden (Amulet Capital) | PE-Readiness Agent + Executive Dashboard | Module E, New Agent |
| M&A integration speed | Schema-per-tenant + automated tenant provisioning | [[ADR-002]] |
| Clinician administrative burden | AI Scribe + Prior Auth Agent + Denial Management Agent | [[ADR-003]] |
| Care gap detection at scale | Quality Reporting Agent + ACO REACH measures | Module E |

### The M&A Flywheel

MedOS enables Theoria to become the **acquirer of choice** in post-acute care:
- **Current integration timeline:** 6-12 months per acquired facility
- **With MedOS:** 60-90 days (automated tenant provisioning, FHIR data ingestion, standardized workflows)
- **Result:** 3-4x more acquisitions per year with lower integration cost

A "Diligence Agent" can analyze a target practice's anonymized claims data and predict revenue lift, cost savings, and clinical outcome improvements -- giving Theoria a data-driven edge in deal sourcing.

### The Financial Case

| Metric | Without MedOS | With MedOS |
|--------|--------------|------------|
| Valuation multiple | 8-12x EBITDA | 15-25x EBITDA |
| CCM revenue capture | Manual, ~40% capture rate | Automated, ~90% capture rate |
| M&A integration time | 6-12 months | 60-90 days |
| Readmission penalty exposure | Reactive | Predictive (wearable monitoring) |
| PE reporting turnaround | Weeks (manual) | Minutes (AI agent) |

**If Theoria's EBITDA is $50M:** The difference between 10x and 20x is **$500M in enterprise value**. MedOS is the bridge.

### 90-Day Pilot Roadmap for Theoria

See [[EPIC-016-theoria-medical-pilot]] for full technical plan and [[theoria-medical-pitch-strategy]] for pitch preparation.

**Phase 1 (Days 1-30):** Read-only data connection to one Michigan facility. Shadow operations. Deliver VBC Opportunity Report.
**Phase 2 (Days 31-60):** Deploy Post-Acute Guardian for 50-100 high-risk patients with wearables. Weekly outcome reports vs control group.
**Phase 3 (Days 61-90):** Turn on CCM Revenue Agent + ACO REACH Quality Agent with human approval dashboard. Full ROI analysis.

### Critical Positioning: MedOS + Theoria's Existing Tech

Theoria is the "only physician group in the country that creates its own technology in-house" — they have:
- **ChartEasy** -- Specialized EHR for long-term care
- **ChatEasy** -- Secure messaging for clinical decision support
- **ProphEasy** -- Clinical decision support and analytics

**MedOS does NOT replace these tools.** MedOS is the AI-native infrastructure layer underneath:
- ChartEasy stores data → MedOS normalizes to FHIR + adds pgvector semantic search + enables AI agents
- ChatEasy handles messaging → MedOS adds A2A agent-to-agent coordination
- ProphEasy does analytics → MedOS adds predictive AI agents with bounded autonomy

### New Agent Architecture (7 additions)

See [[EPIC-016-theoria-medical-pilot]] for detailed agent specifications.

1. **Post-Acute Guardian Agent** -- Continuous wearable monitoring for CHF/post-discharge patients
2. **CCM Revenue Agent** -- Automatic non-face-to-face time tracking and CPT 99490 billing
3. **Shift Summary Agent** -- Priority-ranked briefings during telemedicine shift handoffs
4. **ACO REACH Quality Agent** -- Proactive care gap closure for VBC quality scores
5. **SNF-to-Hospital Semantic Data Bridge** -- NLP on discharge summaries, semantic reconciliation (solves Dr. Di Rezze's founding insight)
6. **Generative Care Plan Optimizer** -- AI "super-consultant" scaling clinical expertise network-wide
7. **Dynamic Staffing & Resource Allocation Agent** -- Optimizes hybrid workforce across national footprint (addresses Amulet's margin expansion mandate)

### New Frontend Pages (13 additions)

| Page | Route | Purpose |
|------|-------|---------|
| Facility Console | `/facility` | SNF-level dashboard for Director of Nursing |
| Shift Handoff | `/shift-handoff` | Priority briefings for telemedicine shifts |
| Post-Acute Guardian | `/guardian` | Live wearable monitoring + alert queue |
| Readmission Risk | `/readmission` | Predictive scoring + interventions |
| CCM Time Tracker | `/ccm` | 20-minute threshold tracking per patient |
| RPM Revenue | `/rpm` | CPT 99453-99458 auto-capture |
| Care Gap Scanner | `/care-gaps` | Population care gap detection + outreach |
| ACO REACH | `/aco-reach` | Quality measure performance (Empassion Health metrics) |
| PE Executive | `/executive` | Board-level KPIs for Amulet Capital ($2.7B AUM) |
| Credentialing | `/credentialing` | Multi-state licensing management (21 states) |
| Discharge Reconciliation | `/discharge` | Hospital→SNF semantic data bridge |
| Care Plan Optimizer | `/care-plans` | AI clinical recommendations with evidence citations |
| Staffing Optimizer | `/staffing` | Dynamic provider scheduling across facilities |

---

## SOURCES & REFERENCES

### Market Data
- CMS National Health Expenditure: $5.6T (2025)
- Healthcare AI Market: $37-39B (2025), projected $56B (2026)
- Administrative waste: $496B annually (Health Affairs)
- AI in healthcare spending: $1.4B in 2025, ~3x from 2024 (Menlo Ventures)

### Clinical Evidence
- JAMA Network Open: Ambient AI reduced burnout 51.9% -> 38.8%
- Documentation time savings: 30 min/day per provider
- Validated at Mass General Brigham, Yale, Emory, UChicago Medicine

### Regulatory
- HIPAA Security Rule update: January 6, 2025 (final expected May 2026)
- FDA: 1,250+ AI-enabled devices authorized (July 2025)
- 47 states introduced 250+ AI-in-healthcare bills in 2025
- CMS FHIR mandate: USCDI v3 by January 1, 2026

### Competitive Intelligence
- Epic: 38%+ hospital market, added 176 facilities in 2024
- Oracle Health: Lost 74 hospitals, 17,232 beds in 2024
- Tempus AI: $1.24B revenue (2025), growing 85% YoY
- Cohere Health: $200M funding, 12M+ annual PA requests, 90% auto-approval
