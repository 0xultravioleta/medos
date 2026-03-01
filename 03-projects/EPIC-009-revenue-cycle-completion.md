---
type: epic
date: "2026-02-28"
status: in-progress
priority: 1
tags:
  - epic
  - sprint-4
  - billing
  - x12
  - revenue
  - phase-1
owner: ""
target-date: "2026-05-09"
---

# EPIC-009: Revenue Cycle Completion -- X12 Claims Pipeline

> **Timeline:** Weeks 9-10 (Sprint S4)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-007-mcp-sdk-refactoring]] (done), [[EPIC-008-demo-polish]] (done)
> **Blocks:** [[EPIC-005-revenue-cycle-mvp]], [[EPIC-006-pilot-readiness]]

## Objective

Complete the revenue cycle pipeline by building the X12 claims generation, scrubbing, remittance parsing, and payment posting modules. This epic delivers the financial backbone of MedOS: HIPAA-compliant 837P professional claims, a pre-submission scrubbing engine that catches denial-causing errors before they leave the building, 835 remittance parsing for automated payment posting, and 4 new MCP tools that expose the claims pipeline to AI agents. Combined with a claims analytics frontend, this epic enables MedOS to demonstrate end-to-end revenue cycle automation from encounter to payment.

Sprint 2 deliverables: 32 MCP tools, 3 LangGraph agents, HIPAAFastMCP SDK. See [[EPIC-007-mcp-sdk-refactoring]].
Sprint 3 deliverables: Approvals UI, WebSocket events, agent runner, patient intake workflow. See [[EPIC-008-demo-polish]].

---

## Timeline (Gantt)

```
Week 9 (Sprint 4, Part 1)
|----- T1: X12 837P Claims Generator ------------|
|---- T2: Claims Scrubbing Rules Engine ----------|
|---- T3: X12 835 Remittance Parser --------------|

Week 10 (Sprint 4, Part 2)
|---- T4: Payment Posting Module -----------------|
|---- T5: Claims Pipeline MCP Tools --------------|
|------------- T6: Claims Analytics Frontend -----|
```

---

## Tasks

### T1: X12 837P Claims Generator
**Complexity:** L
**Estimate:** 6h
**Dependencies:** None (builds on existing FHIR Claim resources)
**References:** [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]]

**Description:**
Generate HIPAA-compliant X12 837P (005010X222A1) professional claims from FHIR Claim resources. The generator takes a FHIR Claim resource (with linked Patient, Practitioner, Organization, Encounter, and Coverage) and produces a valid X12 837P EDI file ready for clearinghouse submission. Supports single and batch claim generation. All required loops (1000A/B, 2000A/B, 2300, 2400) populated from FHIR data.

**Acceptance Criteria:**
- [x] Generator produces valid 005010X222A1 output
- [x] All required X12 segments populated (ISA, GS, ST, BHT, CLM, SV1, DTP, HI, NM1, etc.)
- [x] FHIR Claim -> X12 837P mapping covers: demographics, insurance, diagnosis codes, procedure codes, modifiers, charges, units, dates of service, place of service, NPI
- [x] Batch mode generates multi-claim 837P files
- [x] Output passes X12 syntax validation
- [x] Tests cover single claim, batch, and edge cases (multiple diagnosis codes, modifiers, secondary insurance)

**Status:** Done (33 tests passing)

---

### T2: Claims Scrubbing Rules Engine
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T1
**References:** [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]]

**Description:**
Build a pre-submission claims scrubbing engine that validates claims against 15+ rules before they leave MedOS. Each rule produces a severity (error, warning, info) and a human-readable explanation. Claims that fail any error-level rule are blocked from submission. The engine also produces a denial risk score (0.0-1.0) based on historical patterns and rule violations.

**Acceptance Criteria:**
- [x] 15+ scrubbing rules implemented:
  - Required fields validation (NPI, Tax ID, subscriber ID, dates)
  - ICD-10-CM code validity for date of service
  - CPT code validity and medical necessity (diagnosis-procedure pairing)
  - Modifier correctness (25, 59, XE, XS, XP, XU)
  - Place of service consistency
  - Age/gender consistency with diagnosis
  - NCCI bundling edits (unbundling detection)
  - Timely filing limit check per payer
  - Duplicate claim detection
  - Missing referring provider when required
  - Invalid date ranges (service date in future, DOS before DOB)
  - Missing or invalid authorization number when PA required
  - Charge amount validation (zero charges, negative charges)
  - Subscriber ID format validation per payer
  - Rendering provider NPI validation
- [x] Each rule returns: rule_id, severity (error/warning/info), description, affected field
- [x] Denial risk score calculated from rule violations and historical patterns
- [x] Error-level violations block claim submission
- [x] Scrub report generated as structured JSON

**Status:** Done (18 rules implemented, denial risk scoring operational)

---

### T3: X12 835 Remittance Parser
**Complexity:** M
**Estimate:** 4h
**Dependencies:** None
**References:** [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]]

**Description:**
Parse X12 835 Electronic Remittance Advice (ERA) files into structured data. The parser extracts claim-level payments (CLP segments), adjustment reason codes (CAS segments with CARC/RARC), service line details (SVC segments), and provider-level adjustments (PLB segments). Output is a structured Python model ready for payment posting.

**Acceptance Criteria:**
- [x] Parser handles 005010X221A1 format
- [x] Extracts: payer info, payment amount, check/EFT number, payment date
- [x] Extracts per-claim: claim ID, status, charged amount, paid amount, patient responsibility
- [x] Extracts per-service-line: CPT code, charged, allowed, paid, adjustment amounts
- [x] Extracts adjustment reason codes (CARC) with human-readable descriptions
- [x] Extracts remark codes (RARC) with human-readable descriptions
- [x] Handles multiple claims per 835 file
- [x] Tests with sample 835 files covering: full payment, partial payment, denial, adjustment

**Status:** Done (CLP/CAS/SVC parsing complete)

---

### T4: Payment Posting Module
**Complexity:** M
**Estimate:** 3h
**Dependencies:** T3
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
Match 835 remittance payments to submitted claims and update claim status. Calculate patient responsibility (copay, coinsurance, deductible) from CAS segments. Detect underpayments by comparing paid amounts against contracted rates. Generate patient statements for remaining balances.

**Acceptance Criteria:**
- [x] Payments matched to original claims by claim ID and service line
- [x] Claim status updated: submitted -> paid / partially paid / denied
- [x] Patient responsibility calculated from CAS segments (PR group)
- [x] Contractual adjustments identified from CAS segments (CO group)
- [x] Underpayment detection: paid amount vs expected amount flagged when variance > threshold
- [x] Payment posting creates FHIR ExplanationOfBenefit resources
- [x] Unmatched payments flagged for manual review

**Status:** Done (balance calculation, status determination operational)

---

### T5: Claims Pipeline MCP Tools
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T1, T2, T3, T4
**References:** [[mcp-integration-plan]], [[agent-architecture]]

**Description:**
Register 4 new MCP tools in the Billing MCP server that expose the claims pipeline to AI agents. These tools allow agents to generate claims, scrub for errors, post payments, and query analytics -- enabling end-to-end revenue cycle automation via the MCP protocol.

**Acceptance Criteria:**
- [x] `billing_generate_claim` -- Generate X12 837P from FHIR Claim resource
  - Input: claim_id or FHIR Claim resource
  - Output: Generated 837P data + validation status
  - Requires human approval for submission
- [x] `billing_scrub_claim` -- Run scrubbing rules against a claim
  - Input: claim_id or FHIR Claim resource
  - Output: Scrub report with rule violations, denial risk score
- [x] `billing_post_payment` -- Post payment from parsed 835 data
  - Input: 835 data (parsed) or raw 835 file
  - Output: Payment posting results, patient responsibility, underpayment flags
  - Requires human approval
- [x] `billing_claims_analytics` -- Query claims analytics
  - Input: date range, optional filters (payer, provider, status)
  - Output: Clean claim rate, denial rate by category, AR aging, revenue summary
- [x] All 4 tools registered via `@hipaa_tool` with PHI access level and required scopes
- [x] Total MCP tool count: 36 (32 existing + 4 new)

**Status:** Done (4 new MCP tools registered, 36 total)

---

### T6: Claims Analytics Frontend
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T1, T2, T4
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
Build a Claims Analytics dashboard section in the Next.js frontend. Displays key revenue cycle metrics: clean claim rate, denial rate breakdown by CARC category, AR aging buckets, and revenue trends. All data from `GET /api/v1/analytics/claims` endpoint. Server Component for data fetching, Client Component only for interactive charts.

**Acceptance Criteria:**
- [x] Clean claim rate displayed as percentage with trend (last 30/60/90 days)
- [x] Denial breakdown by CARC category (pie chart or bar chart)
- [x] AR aging buckets: 0-30, 31-60, 61-90, 91-120, 120+ days
- [x] Revenue summary: charges submitted, payments received, adjustments, patient responsibility
- [x] Filters: date range, payer, provider
- [x] Server Component fetches data; Client Component renders charts
- [x] No PHI displayed on analytics dashboard (aggregated metrics only)

**Status:** Done (GaugeRing KPIs, AR aging chart, denial breakdown chart)

---

## Dependencies Map

```
T1 (837P Generator) ──> T2 (Scrubbing Engine)
T3 (835 Parser) ──> T4 (Payment Posting)
T1 + T2 + T3 + T4 ──> T5 (MCP Tools)
T1 + T2 + T4 ──> T6 (Claims Analytics)
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| X12 837P Generator (T1) | [[EPIC-005-revenue-cycle-mvp]] |
| Claims Scrubbing Engine (T2) | [[EPIC-005-revenue-cycle-mvp]], [[EPIC-006-pilot-readiness]] |
| X12 835 Parser + Payment Posting (T3, T4) | [[EPIC-005-revenue-cycle-mvp]] |
| Claims Pipeline MCP Tools (T5) | All billing agents, [[EPIC-006-pilot-readiness]] |
| Claims Analytics Frontend (T6) | [[EPIC-006-pilot-readiness]] (demo) |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| X12 837P format edge cases across payers | Medium | High | Test against multiple payer profiles, use pyx12 for validation, extensive fixtures |
| Scrubbing rules not covering payer-specific edits | Medium | Medium | Start with CMS/NCCI universal rules, add payer-specific rules iteratively |
| 835 parser handling non-standard payer remittances | Medium | Medium | Test with real 835 samples from multiple payers, graceful fallback for unrecognized segments |
| Payment posting mismatches between 835 and submitted claims | Low | High | Strict matching on claim ID + service date + CPT, flag unmatched for manual review |
| MCP tool count growth impacting gateway performance | Low | Low | Tool registry is in-memory, 36 tools well within capacity |

---

## Definition of Done

- [x] X12 837P generator producing valid 005010X222A1 output
- [x] Claims scrubber catching 15+ denial types with risk scoring
- [x] X12 835 parser extracting payment, adjustment, and patient responsibility data
- [x] Payment posting matching 835 payments to submitted claims
- [x] 4 new MCP tools registered (total: 36 tools)
- [x] Claims analytics on frontend with charts
- [ ] 220+ tests passing, ruff clean

---

## References

- [[EPIC-007-mcp-sdk-refactoring]] -- Sprint 2 scope (MCP SDK + 32 tools)
- [[EPIC-008-demo-polish]] -- Sprint 3 scope (Approvals UI, WebSocket, agent runner)
- [[ADR-005-mcp-sdk-integration]] -- HIPAAFastMCP design decision
- [[mcp-integration-plan]] -- MCP tool catalog and gateway
- [[Revenue-Cycle-Deep-Dive]] -- Revenue cycle business processes
- [[X12-EDI-Deep-Dive]] -- X12 EDI format specifications
- [[Prior-Authorization-Deep-Dive]] -- PA workflow (related, built in Sprint 2)
- [[agent-architecture]] -- Agent framework consuming MCP tools
