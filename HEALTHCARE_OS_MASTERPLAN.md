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

### Module B: Provider Workflow Engine
- **Ambient AI Documentation**: Audio capture -> speech-to-text -> clinical NLU -> note generation -> provider review
- Proven results: JAMA study showed burnout dropped from 51.9% to 38.8%, saves 30 min/day
- Smart scheduling with ML-predicted no-shows and overbooking optimization
- CDS Hooks integration for clinical decision support

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
- Remote patient monitoring (BLE devices, Apple Health, Google Health Connect)
- Telehealth integration (Twilio Video)

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

### Recommended: Orthopedics or Dermatology First
High volume, procedure-heavy, lots of prior auth pain, tech-forward physicians.

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

## 13. THE 90-DAY PITCH TO DR. DI REZE

### The Framework (4 minutes total)

**1. The Problem (30 seconds)**
"Healthcare practices run on software from the 2000s. Staff spends 60-70% of time on admin. Burnout is driving physicians out. The technology to fix this exists today, but nobody has built it from scratch with AI at the core."

**2. The Vision (60 seconds)**
"We're building the operating system for healthcare practices. AI-native from the ground up. One platform that handles scheduling, documentation, coding, billing, prior auth, and analytics -- with AI doing the heavy lifting and humans handling exceptions. We're giving every practice the operational capability of the Mayo Clinic."

**3. Why Now (30 seconds)**
"Three things converged: AI models are reliable enough. CMS mandated open APIs, breaking lock-in. And the staffing crisis means practices will pay for automation they'd have resisted three years ago."

**4. The Plan (90 seconds)**
"Phase 1: Two of us build core platform in 90 days. Target mid-size specialty practices in Florida. 3-5 free pilots, prove ROI, convert to paid.
Phase 2: Hire to 10, get SOC 2, scale to 50-100 practices, $5-10M ARR.
Phase 3: Hire to 50, HITRUST, national expansion, marketplace. $25-40M ARR by month 36."

**5. Why This Team (30 seconds)**
"I know how to set up the systems that let a small team move at 5x speed. I'll build the infrastructure, the AI pipeline, the compliance framework. Combined with your healthcare domain expertise and relationships, we can outexecute teams 10x our size."

**6. The Ask (15 seconds)**
"Give me 90 days. I'll build the infrastructure, ship an MVP, and get it in front of real practices. You'll see exactly what I can do."

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
