---
type: epic
date: "2026-02-28"
status: planning
priority: 5
tags:
  - project
  - epic
  - phase-1
  - billing
  - rcm
  - module-c
  - module-d
owner: ""
target-date: "2026-05-01"
---

# EPIC-005: Revenue Cycle MVP

> **Timeline:** Week 7-10 (2026-04-17 to 2026-05-01)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-003-fhir-data-layer]] (FHIR CRUD, search, claims resources), [[EPIC-004-ai-clinical-documentation]] (AI coding suggestions, encounter data)
> **Blocks:** [[EPIC-006-pilot-readiness]] (billing must be functional for pilot launch)

## Overview

This epic delivers the minimum viable revenue cycle management (RCM) pipeline for MedOS, covering the full lifecycle from eligibility verification through claims submission and remittance processing. The system integrates with the [[EPIC-004-ai-clinical-documentation]] AI coding engine to auto-populate charges from clinical encounters, generates compliant X12 EDI transactions for clearinghouse submission, and provides a denial tracking and analytics dashboard. By automating the charge capture-to-payment loop, MedOS aims to reduce days in A/R, decrease claim denial rates, and eliminate manual coding bottlenecks for pilot practices. All billing data is persisted as FHIR resources (Claim, ClaimResponse, Coverage, ExplanationOfBenefit) per [[ADR-001-fhir-native-data-model]], ensuring a unified clinical-financial data model.

---

## Goals

- [ ] Verify patient insurance eligibility in real-time before encounters
- [ ] Auto-capture charges from AI-generated clinical documentation
- [ ] Generate compliant X12 837P claims for professional services
- [ ] Process remittance (X12 835) and post payments automatically
- [ ] Track and analyze claim denials with AI-powered prediction
- [ ] Provide revenue analytics dashboard for practice administrators

---

## Success Metrics

| Metric | Target | Measurement |
|---|---|---|
| Eligibility verification turnaround | < 15 seconds real-time | Measure API response time from payer X12 270/271 exchange |
| Clean claim rate | > 95% (first submission) | Track claims accepted without rejection on first submission |
| Auto charge capture rate | > 80% of encounters | Percentage of encounters with auto-populated charges from AI coding |
| Days in A/R | < 35 days | Average time from service date to payment posting |
| Denial rate | < 5% of claims | Track denied claims as percentage of total submitted |
| Denial appeal success rate | > 50% | Track overturned denials from AI-assisted appeals |

---

## Tasks

### T1: Eligibility Verification (X12 270/271)
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** [[EPIC-003-fhir-data-layer]] T4 (FHIR CRUD for Coverage resource)
**References:** [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]], [[Prior-Authorization-Deep-Dive]]

**Description:**
Implement real-time insurance eligibility verification using X12 270 (inquiry) and 271 (response) transactions via clearinghouse integration. Results are stored as FHIR Coverage and CoverageEligibilityResponse resources.

**Subtasks:**
- [ ] Build X12 270 eligibility inquiry message generator:
  - Subscriber demographics (from FHIR Patient)
  - Payer identification (from FHIR Organization/InsurancePlan)
  - Service type codes (medical, mental health, pharmacy, etc.)
  - Date of service range
- [ ] Integrate with clearinghouse API for 270/271 exchange (e.g., Availity, Change Healthcare)
- [ ] Parse X12 271 eligibility response:
  - Active/inactive coverage status
  - Copay, deductible, coinsurance amounts
  - In-network vs out-of-network benefits
  - Prior authorization requirements
  - Remaining deductible and out-of-pocket
- [ ] Persist eligibility results as FHIR resources:
  - FHIR Coverage (insurance plan details)
  - FHIR CoverageEligibilityResponse (benefit details)
- [ ] Build eligibility check UI component:
  - One-click verification from patient chart
  - Display benefit summary with copay/deductible amounts
  - Alert on inactive coverage or missing authorization
- [ ] Implement batch eligibility checking (verify all patients for next day's schedule)

**Acceptance Criteria:**
- [ ] Real-time eligibility response in < 15 seconds
- [ ] Correct parsing of major payer 271 responses (Medicare, Medicaid, top 10 commercial)
- [ ] FHIR Coverage and CoverageEligibilityResponse resources created
- [ ] Batch eligibility runs nightly for next-day scheduled patients
- [ ] UI displays actionable benefit information

---

### T2: Prior Authorization Tracking
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1
**References:** [[Prior-Authorization-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Track prior authorization requirements and status for procedures, medications, and referrals. Integrate with payer portals where APIs are available, and provide manual tracking workflow otherwise.

**Subtasks:**
- [ ] Build prior auth requirement detection:
  - Parse 271 eligibility responses for auth-required service types
  - Maintain payer-specific auth requirement rules database
  - Alert providers when ordered services require authorization
- [ ] Implement auth request tracking workflow:
  - Create auth request from ServiceRequest/MedicationRequest
  - Track status: Submitted -> Pending -> Approved/Denied/Partial
  - Store auth reference numbers linked to FHIR Claim
  - Expiration date tracking with renewal alerts
- [ ] Build prior auth dashboard:
  - Pending authorizations with days waiting
  - Expiring authorizations (next 30 days)
  - Denied authorizations with appeal deadlines
- [ ] Implement FHIR ClaimResponse for auth decisions
- [ ] Create auth document attachment support (clinical notes, lab results)

**Acceptance Criteria:**
- [ ] Auth requirements detected from eligibility data for major payers
- [ ] Auth status tracked through full lifecycle
- [ ] Expiration alerts sent 14 days before auth expires
- [ ] Auth reference numbers linked to claims for clean submission

---

### T3: AI Coding Engine v1
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** [[EPIC-004-ai-clinical-documentation]] T6 (AI coding suggestions)
**References:** [[Revenue-Cycle-Deep-Dive]], [[ADR-003-ai-agent-framework]]

**Description:**
Extend the AI coding suggestions from [[EPIC-004-ai-clinical-documentation]] into a full coding engine that validates code combinations, checks medical necessity, and generates charge entries ready for claims submission.

**Subtasks:**
- [ ] Build code validation layer:
  - ICD-10-CM code validity check (active codes, correct format)
  - CPT code validity and modifier requirements
  - ICD-10 / CPT compatibility (diagnosis supports procedure medical necessity)
  - NCCI edits check (National Correct Coding Initiative bundling rules)
  - Gender and age-specific code restrictions
- [ ] Implement E/M level calculator:
  - 2021 E/M guidelines (time-based and MDM-based)
  - Documentation element scoring (HPI, ROS, PFSH, exam)
  - Medical Decision Making (MDM) complexity assessment
  - Time-based coding with time documentation
- [ ] Build charge entry generation:
  - Map approved ICD-10 + CPT codes to charge items
  - Apply fee schedule (configurable per payer contract)
  - Calculate expected reimbursement
  - Link charges to Encounter and Practitioner
- [ ] Implement coding compliance alerts:
  - Upcoding risk detection (E/M level vs documentation support)
  - Unbundling detection (separate billing for bundled services)
  - Frequency limits (e.g., annual wellness visit once per year)
- [ ] Store charges as FHIR ChargeItem resources

**Acceptance Criteria:**
- [ ] Code validation catches > 95% of coding errors before claim submission
- [ ] E/M level calculator matches manual coder determination > 85% of time
- [ ] NCCI edit checks prevent bundling violations
- [ ] Compliance alerts flag potential upcoding with clinical justification
- [ ] Charges auto-generated from AI coding suggestions for > 80% of encounters

---

### T4: Charge Capture Workflow
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T3, [[EPIC-004-ai-clinical-documentation]] T5 (provider review UI)
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
Build the charge capture workflow that collects, reviews, and finalizes charges from clinical encounters before claims generation.

**Subtasks:**
- [ ] Implement auto charge capture from approved clinical notes:
  - When provider signs note (EPIC-004 T5), auto-generate charges
  - Include all diagnosis codes, procedure codes, and modifiers
  - Apply provider-specific fee schedules
- [ ] Build charge review interface:
  - List pending charges by date of service
  - Show AI-suggested codes with confidence indicators
  - Allow add/edit/remove of charge line items
  - Batch approval for high-confidence charges
- [ ] Implement charge hold rules:
  - Hold charges missing required fields (diagnosis, modifier, etc.)
  - Hold charges flagged by compliance alerts
  - Hold charges for encounters missing provider signature
- [ ] Build charge submission workflow:
  - Individual charge finalization
  - Batch charge finalization (end-of-day charge review)
  - Charge status tracking: Pending -> Reviewed -> Finalized -> Billed

**Acceptance Criteria:**
- [ ] Auto-generated charges appear within 1 minute of note signing
- [ ] Charge review interface loads < 2 seconds with day's charges
- [ ] Hold rules prevent incomplete charges from proceeding to claims
- [ ] Batch approval workflow processes day's charges in < 5 minutes

---

### T5: Claims Generation (X12 837P)
**Complexity:** L
**Estimate:** 4 days
**Dependencies:** T3, T4
**References:** [[X12-EDI-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Generate HIPAA-compliant X12 837P (Professional) claim transactions from finalized charges. Claims must conform to the X12 5010 standard and pass clearinghouse validation.

**Subtasks:**
- [ ] Build X12 837P claim generator:
  - ISA/GS envelope segments (interchange/functional group headers)
  - ST/SE transaction set headers
  - Loop 2000A (Billing Provider)
  - Loop 2000B (Subscriber/Patient)
  - Loop 2300 (Claim Information) -- diagnosis codes, dates, amounts
  - Loop 2400 (Service Lines) -- CPT codes, modifiers, charges, units
- [ ] Implement claim data assembly from FHIR resources:
  - Patient demographics -> subscriber information
  - Coverage -> payer information, subscriber ID
  - Encounter -> dates of service
  - ChargeItem -> service lines
  - Practitioner -> rendering/billing provider NPI
  - Organization -> billing entity
- [ ] Create FHIR Claim resource from assembled data
- [ ] Implement claim validation before submission:
  - Required fields present (NPI, tax ID, diagnosis codes)
  - Code format validation (ICD-10, CPT, modifier format)
  - Date range validation (service date within coverage period)
  - Duplicate claim detection (same patient, same date, same codes)
- [ ] Integrate with clearinghouse submission API
- [ ] Handle claim acknowledgments:
  - X12 999 (Functional Acknowledgment) -- syntax validation
  - X12 277 (Claim Status) -- payer acceptance/rejection

**Acceptance Criteria:**
- [ ] Generated 837P passes clearinghouse syntax validation > 99% of time
- [ ] All required X12 segments and loops populated correctly
- [ ] FHIR Claim resource created and linked to Encounter/ChargeItem
- [ ] Duplicate claim detection prevents resubmission
- [ ] Claim submission status tracked through acknowledgment processing

---

### T6: Claims Scrubbing Engine
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T5
**References:** [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]]

**Description:**
Pre-submission claims scrubbing to catch errors before clearinghouse submission, maximizing the clean claim rate.

**Subtasks:**
- [ ] Implement scrubbing rules engine:
  - Payer-specific requirements (Medicare vs commercial vs Medicaid)
  - Missing/invalid data detection (NPI, tax ID, member ID format)
  - Code-level validation (deleted codes, gender/age restrictions)
  - Modifier requirements (certain CPT codes require specific modifiers)
  - Place of service validation
  - Timely filing deadline check (claim must be filed within payer deadline)
- [ ] Build scrubbing results interface:
  - Error list with severity (fatal, warning, info)
  - One-click fix for common issues (e.g., missing modifier)
  - Skip to claim line with error highlighted
- [ ] Implement auto-fix for common issues:
  - Add required modifiers automatically
  - Correct common code format errors
  - Suggest replacement for deleted/retired codes
- [ ] Track scrubbing metrics:
  - Error rate by type
  - Most common errors by provider
  - Clean claim rate trend

**Acceptance Criteria:**
- [ ] Scrubbing catches > 98% of errors that would cause clearinghouse rejection
- [ ] Auto-fix resolves > 50% of scrubbing errors without manual intervention
- [ ] Scrubbing completes in < 5 seconds per claim
- [ ] Clean claim rate > 95% after scrubbing

---

### T7: Clearinghouse Integration
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T5, T6
**References:** [[X12-EDI-Deep-Dive]]

**Description:**
Build the integration layer with one or more clearinghouses for claim submission and response processing.

**Subtasks:**
- [ ] Implement clearinghouse API integration (primary: Availity or Change Healthcare)
- [ ] Build claim submission queue:
  - Batch submission (configurable: real-time or scheduled)
  - Retry logic for failed submissions with exponential backoff
  - Submission tracking (submitted timestamp, confirmation number)
- [ ] Implement response processing:
  - X12 999 -- Functional Acknowledgment (syntax OK/error)
  - X12 277 -- Claim Status Response (accepted/rejected/pending)
  - X12 835 -- Remittance Advice (payment details, handled in T8)
- [ ] Build clearinghouse dashboard:
  - Submission queue status
  - Acceptance/rejection rates
  - Pending claims by age
- [ ] Implement secondary clearinghouse failover

**Acceptance Criteria:**
- [ ] Claims submitted to clearinghouse within 24 hours of finalization
- [ ] Acknowledgment responses processed within 1 hour of receipt
- [ ] Retry logic handles transient failures without data loss
- [ ] Dashboard shows real-time submission status

---

### T8: Remittance Processing (X12 835)
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T7
**References:** [[X12-EDI-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Parse X12 835 Electronic Remittance Advice (ERA) files to post payments, adjustments, and denials automatically.

**Subtasks:**
- [ ] Build X12 835 parser:
  - CLP (Claim Payment) segment -- claim-level payment info
  - SVC (Service Payment) segment -- line-level payment info
  - CAS (Claim Adjustment) segment -- adjustment reason codes
  - PLB (Provider Level Balance) segment -- provider-level adjustments
- [ ] Implement payment posting:
  - Match 835 payments to submitted claims
  - Post payment amounts at claim and line level
  - Post contractual adjustments
  - Post patient responsibility (copay, deductible, coinsurance)
  - Handle partial payments and split payments
- [ ] Create FHIR ExplanationOfBenefit resource from 835 data
- [ ] Build payment posting review interface:
  - Auto-posted payments (high confidence matching)
  - Manual review queue (ambiguous matching)
  - Payment summary by date, payer, provider
- [ ] Implement patient statement generation:
  - Calculate patient balance after insurance payment
  - Generate patient-friendly statement
  - Track patient payment status

**Acceptance Criteria:**
- [ ] 835 parser handles all major payer ERA formats
- [ ] Auto-posting rate > 90% (matched without manual intervention)
- [ ] Payment amounts reconcile with bank deposits
- [ ] FHIR ExplanationOfBenefit resources created for all processed remittances
- [ ] Patient statements generated within 48 hours of ERA processing

---

### T9: Denial Tracking and Management
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T8
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
Track claim denials, categorize by reason, and manage the appeal workflow.

**Subtasks:**
- [ ] Build denial detection from 835 remittance:
  - Parse CAS adjustment reason codes (CARC) and remark codes (RARC)
  - Categorize denials: eligibility, authorization, coding, medical necessity, timely filing
  - Link denied claims to original submission
- [ ] Implement denial workflow:
  - Auto-create denial work items from 835 processing
  - Assign to billing staff based on denial category
  - Track status: Identified -> Under Review -> Appeal Submitted -> Resolved
  - Appeal deadline tracking with escalation alerts
- [ ] Build denial analytics dashboard:
  - Denial rate by payer, provider, denial reason
  - Top denial reasons with trend analysis
  - Average days to resolve by category
  - Financial impact (denied dollars, recovered dollars)
- [ ] Implement denial appeal document generation:
  - Pull supporting clinical documentation from FHIR resources
  - Generate appeal letter template
  - Attach relevant records (notes, labs, auth numbers)

**Acceptance Criteria:**
- [ ] All denials from 835 automatically tracked
- [ ] Denial categorization accuracy > 95%
- [ ] Appeal deadline alerts prevent missed filing windows
- [ ] Analytics dashboard shows denial trends with actionable insights

---

### T10: AI Denial Prediction
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T9, T3
**References:** [[ADR-003-ai-agent-framework]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Use historical denial data and claim characteristics to predict denial risk before submission, allowing proactive correction.

**Subtasks:**
- [ ] Build denial prediction model:
  - Features: payer, CPT code, ICD-10 code, E/M level, provider, prior denials
  - Training data: historical claims and their outcomes
  - Prediction: probability of denial + predicted denial reason
- [ ] Integrate prediction into claims scrubbing workflow (T6):
  - Flag high-risk claims before submission
  - Suggest corrective actions based on predicted denial reason
  - Display risk score in claim review interface
- [ ] Implement feedback loop:
  - Track prediction accuracy against actual outcomes
  - Retrain model monthly with new denial data
- [ ] Build denial prevention alerts:
  - Real-time alerts during charge capture for high-risk code combinations
  - Provider education suggestions for repeat denial patterns

**Acceptance Criteria:**
- [ ] Denial prediction accuracy > 75% on test set
- [ ] High-risk claims flagged before submission
- [ ] Corrective action suggestions reduce actual denial rate by > 20%
- [ ] Model retraining pipeline operational

---

### T11: Revenue Analytics Dashboard
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T8, T9
**References:** [[Revenue-Cycle-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Build a practice administrator dashboard showing key revenue cycle metrics, trends, and actionable insights.

**Subtasks:**
- [ ] Implement core revenue metrics:
  - Total charges, payments, adjustments by period
  - Days in A/R (aging buckets: 0-30, 31-60, 61-90, 90+)
  - Clean claim rate and trend
  - Denial rate and trend
  - Collection rate (payments / charges)
  - Net collection rate (payments / allowed amounts)
- [ ] Build visualization components:
  - Revenue trend chart (monthly, by payer)
  - A/R aging waterfall chart
  - Denial rate by category (pie/bar chart)
  - Provider productivity (charges per provider per day)
- [ ] Implement drill-down capability:
  - Click metric to see underlying claims
  - Filter by date range, payer, provider, location
  - Export to CSV for external analysis
- [ ] Build automated reporting:
  - Weekly revenue summary email to administrators
  - Monthly financial report generation
  - Custom report builder (stretch)

**Acceptance Criteria:**
- [ ] Dashboard loads < 3 seconds with 6 months of data
- [ ] All core metrics calculated correctly against source data
- [ ] Drill-down from metrics to individual claims functional
- [ ] Weekly automated reports delivered on schedule

---

## Dependencies Map

```
T1 (Eligibility) ──> T2 (Prior Auth)
                                        T3 (AI Coding) ──> T4 (Charge Capture) ──> T5 (Claims 837P)
                                                                                        │
                                                           T5 ──> T6 (Scrubbing) ──> T7 (Clearinghouse)
                                                                                        │
                                                           T7 ──> T8 (Remittance 835) ──> T9 (Denials)
                                                                                           │
                                                           T9 ──> T10 (AI Denial Prediction)
                                                           T8 + T9 ──> T11 (Analytics Dashboard)
```

---

## Cross-Epic Dependencies

| This Epic Requires | Provided By |
|---|---|
| FHIR CRUD API (Claim, Coverage, EOB) | [[EPIC-003-fhir-data-layer]] T4 |
| FHIR search (find claims, encounters) | [[EPIC-003-fhir-data-layer]] T5 |
| AI coding suggestions (ICD-10 + CPT) | [[EPIC-004-ai-clinical-documentation]] T6 |
| Provider-signed clinical notes | [[EPIC-004-ai-clinical-documentation]] T5 |
| Encounter data (dates, providers) | [[EPIC-003-fhir-data-layer]] T4 |
| Patient demographics | [[EPIC-003-fhir-data-layer]] T4 |
| JWT authentication and RBAC | [[EPIC-002-auth-identity-system]] T2, T9 |

| This Epic Provides | Required By |
|---|---|
| Billing functionality for pilot | [[EPIC-006-pilot-readiness]] (working billing is launch requirement) |
| Revenue analytics | [[EPIC-006-pilot-readiness]] (Go/No-Go criteria) |
| Claims submission pipeline | [[EPIC-006-pilot-readiness]] (end-to-end workflow demo) |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Clearinghouse integration delays (API access, testing environments) | High | High | Start clearinghouse enrollment early (Week 1); have backup clearinghouse option; build mock clearinghouse for testing |
| X12 parsing complexity (payer-specific variations in 835/271) | High | Medium | Start with top 5 payers; build flexible parser with payer-specific overrides; maintain payer quirks database |
| Payer-specific claim requirements not documented | Medium | High | Build payer rules database incrementally; start with Medicare/Medicaid (well-documented); add commercial payers as enrolled |
| AI coding engine suggests incorrect codes leading to compliance risk | Medium | Critical | Mandatory human review for all codes; compliance alert system; regular coding accuracy audits; coding education alerts |
| Low clean claim rate during initial deployment | High | Medium | Aggressive scrubbing rules; start with simple claim types (E/M only); add complexity incrementally |
| Timely filing deadlines missed during system transition | Low | High | Track filing deadlines with buffer alerts (7 days); daily aging report; escalation for claims approaching deadline |

---

## Timeline

```
Week 7 (Apr 17 - Apr 23)
|----- T1: Eligibility Verification (270/271) ------|
|----- T2: Prior Authorization Tracking -------------|
|----- T3: AI Coding Engine v1 (start) -------------|

Week 8 (Apr 24 - Apr 30)
|----- T3: AI Coding Engine v1 (complete) ----------|
|----- T4: Charge Capture Workflow ------------------|
|----- T5: Claims Generation 837P (start) ----------|

Week 9 (May 1 - May 7)
|----- T5: Claims Generation 837P (complete) -------|
|----- T6: Claims Scrubbing Engine ------------------|
|----- T7: Clearinghouse Integration ----------------|

Week 10 (May 8 - May 14)
|----- T8: Remittance Processing (835) --------------|
|----- T9: Denial Tracking and Management ------------|
|----- T10: AI Denial Prediction --------------------|
|----- T11: Revenue Analytics Dashboard --------------|
```

---

## Definition of Done

- [ ] Eligibility verification returns real-time benefits from major payers
- [ ] Charges auto-captured from AI-coded clinical encounters
- [ ] X12 837P claims generated and submitted via clearinghouse
- [ ] X12 835 remittances parsed and payments auto-posted
- [ ] Denial tracking with categorization and appeal workflow
- [ ] AI denial prediction flags high-risk claims pre-submission
- [ ] Revenue analytics dashboard operational with core metrics
- [ ] Clean claim rate > 95% in testing with synthetic data
- [ ] All billing data persisted as FHIR resources (Claim, ClaimResponse, EOB, Coverage)
