---
type: epic
date: "2026-02-27"
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

> **Timeline:** Week 7-10 (2026-04-10 to 2026-05-01)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-003-fhir-data-layer]] (FHIR CRUD, search), [[EPIC-004-ai-clinical-documentation]] (AI coding, FHIR resources from notes)
> **Blocks:** [[EPIC-006-pilot-readiness]]

## Objective

Build the minimum viable revenue cycle management (RCM) system that takes AI-coded encounters from the clinical documentation module and carries them through the billing pipeline: eligibility verification, charge capture, claims generation, submission, remittance processing, and denial tracking. For pilot practices, this must demonstrably reduce claim denials and accelerate collections.

---

## Timeline (Gantt)

```
Week 7 (Apr 10 - Apr 17)
|----- T1: Eligibility Verification (270/271) -----|
|----- T2: Prior Auth Status Tracking --------------|
|----------- T3: Prior Auth Automation v1 ----------|

Week 8 (Apr 18 - Apr 24)
|----- T4: AI Coding Engine v1 ---------------------|
|----- T5: Charge Capture Workflow ------------------|

Week 9 (Apr 25 - May 1)
|----- T6: Claims Generation (837P) -----------------|
|----- T7: Claims Scrubbing Rules Engine -------------|
|----------- T8: Clearinghouse Integration ----------|

Week 10 (May 1 - May 8)
|----- T9: Remittance Processing (835) --------------|
|----- T10: Denial Tracking + AI Prediction ---------|
|----------- T11: Revenue Analytics Dashboard -------|
|----------- T12: Integration Testing --------------|
```

---

## Tasks

### T1: Eligibility Verification Engine (X12 270/271)
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** [[EPIC-002-auth-identity-system]] T6 (API keys for payer integrations)
**References:** [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]]

**Description:**
Build real-time patient insurance eligibility verification using X12 270/271 transactions. Practices need to verify coverage before rendering services to avoid claim denials.

**Subtasks:**
- [ ] Implement X12 270 (Eligibility Inquiry) generator:
  - Map FHIR Coverage resource to X12 270 segments:
    - ISA/GS envelope (sender/receiver IDs, dates)
    - ST/BHT transaction header
    - 2000A Information Source (payer)
    - 2000B Information Receiver (practice)
    - 2000C Subscriber (patient/subscriber)
    - 2100C Subscriber Name
    - 2000D Dependent (if patient != subscriber)
  - Support inquiry types:
    - Active coverage verification
    - Co-pay/deductible amounts
    - Service-type-specific benefits (office visit, specialist, imaging, etc.)
    - Remaining deductible
    - Out-of-pocket maximum status
- [ ] Implement X12 271 (Eligibility Response) parser:
  - Parse all benefit information segments (EB loops)
  - Extract:
    - Coverage active/inactive status
    - Plan name and group number
    - Co-pay amounts by service type
    - Deductible (individual/family, met/remaining)
    - Out-of-pocket maximum (met/remaining)
    - Prior authorization requirements
    - Referral requirements
    - In-network vs. out-of-network benefits
  - Handle AAA (error) segments gracefully
- [ ] Create eligibility verification API:
  ```
  POST /api/v1/eligibility/verify
  {
    "patient_id": "uuid",
    "coverage_id": "uuid",
    "service_type": "office_visit",
    "date_of_service": "2026-04-15"
  }

  Response:
  {
    "status": "active",
    "plan_name": "Blue Cross PPO Gold",
    "copay": { "office_visit": 25.00, "specialist": 50.00 },
    "deductible": { "individual": 1500.00, "remaining": 750.00 },
    "prior_auth_required": false,
    "referral_required": false,
    "raw_271": "..."
  }
  ```
- [ ] Implement batch eligibility verification:
  - Verify all patients with appointments for next business day
  - Run nightly at 8 PM (configurable)
  - Flag patients with inactive/changed coverage
  - Alert front desk for follow-up
- [ ] Store eligibility results as FHIR CoverageEligibilityResponse resources
- [ ] Implement eligibility result caching (24-hour TTL)

**Acceptance Criteria:**
- [ ] Real-time eligibility check returns in < 10 seconds
- [ ] Correctly parses 271 responses from top 5 payers (BCBS, Aetna, UHC, Cigna, Humana)
- [ ] Batch verification runs for next-day appointments
- [ ] Inactive coverage flagged and front desk alerted
- [ ] Results stored as FHIR resources with audit trail
- [ ] X12 transaction logs retained for 7 years

---

### T2: Prior Authorization Status Tracking Dashboard
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1
**References:** [[Prior-Authorization-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Build a dashboard for tracking prior authorization requests across payers. Prior auths are the biggest pain point in RCM -- delayed auths delay care and payment.

**Subtasks:**
- [ ] Design prior auth data model:
  ```sql
  CREATE TABLE prior_authorizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    patient_id UUID NOT NULL,
    encounter_id UUID,
    payer_id UUID NOT NULL,
    service_type VARCHAR(50) NOT NULL,
    cpt_codes TEXT[] NOT NULL,
    icd10_codes TEXT[] NOT NULL,
    status VARCHAR(30) NOT NULL,
    submitted_at TIMESTAMPTZ,
    decision_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    auth_number VARCHAR(100),
    denial_reason TEXT,
    payer_reference VARCHAR(255),
    clinical_docs JSONB,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );
  ```
- [ ] Build prior auth tracking dashboard (Next.js):
  - Kanban view: Draft | Submitted | Pending | Approved | Denied
  - List view with filters: by payer, by provider, by status, by date range
  - Color coding: green (approved), yellow (pending > 5 days), red (denied/expiring)
  - Click into detail view: patient info, service details, clinical docs, timeline
- [ ] Implement status update workflow:
  - Manual status updates from staff
  - Automated status checks via payer portals (stretch goal)
  - Expiration alerts (30, 14, 7 days before expiry)
  - Notification to provider when decision received
- [ ] Implement reporting:
  - Average time to decision by payer
  - Approval rate by payer and service type
  - Denial reasons breakdown
  - Expiring authorizations report
- [ ] Create FHIR ClaimResponse mapping for prior auth decisions

**Acceptance Criteria:**
- [ ] Dashboard shows all prior auths with current status
- [ ] Status filters and sorting work correctly
- [ ] Expiration alerts fire at configured intervals
- [ ] Reporting shows payer-level metrics
- [ ] Staff can update status and attach documents

---

### T3: Basic Prior Authorization Automation
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T2, [[EPIC-004-ai-clinical-documentation]] T3 (clinical NLU)
**References:** [[Prior-Authorization-Deep-Dive]], [[ADR-003-ai-agent-framework]]

**Description:**
Automate the clinical documentation gathering and submission for prior authorization requests. AI reviews clinical notes, gathers supporting documentation, and pre-fills the prior auth form.

**Subtasks:**
- [ ] Implement clinical documentation assembly:
  - From AI-generated SOAP notes (EPIC-004), extract:
    - Relevant diagnoses (ICD-10)
    - Requested procedure (CPT)
    - Medical necessity justification
    - Previous treatments tried and failed
    - Relevant lab results and imaging
  - Package into prior auth submission format
- [ ] Implement payer-specific requirements engine:
  ```json
  {
    "payer": "BCBS",
    "service": "MRI_lumbar",
    "requirements": {
      "diagnoses_required": true,
      "conservative_treatment_documented": true,
      "conservative_treatment_duration_days": 42,
      "imaging_required": false,
      "clinical_notes_required": true,
      "specific_forms": ["BCBS_PA_FORM_2026"]
    }
  }
  ```
- [ ] Implement AI-powered medical necessity letter generation:
  - Claude generates peer-to-peer review summary
  - Include clinical evidence supporting medical necessity
  - Reference applicable guidelines (InterQual, MCG)
  - Provider reviews and signs
- [ ] Create submission workflow:
  1. System identifies auth requirement (from eligibility check T1)
  2. AI assembles clinical documentation
  3. AI generates medical necessity letter
  4. Staff reviews assembled package
  5. Submit to payer (fax or electronic portal)
  6. Track in dashboard (T2)
- [ ] Implement fax integration (for payers without electronic submission):
  - Generate PDF from prior auth form
  - Send via fax API (SRFax, Phaxio, or similar)
  - Receive confirmation and store
- [ ] Track automation metrics:
  - Time saved per prior auth (vs. manual)
  - Auto-approval rate for AI-assembled submissions
  - Denial rate comparison (AI vs. manual submissions)

**Acceptance Criteria:**
- [ ] AI assembles prior auth package in < 5 minutes (vs. 30+ minutes manual)
- [ ] Medical necessity letter reads professionally and includes relevant clinical evidence
- [ ] Payer-specific requirements met for top 5 payers
- [ ] Staff can review and override AI-assembled documentation
- [ ] Submission tracking integrates with dashboard (T2)
- [ ] Fax submission confirmed and tracked

---

### T4: AI Coding Engine v1
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** [[EPIC-004-ai-clinical-documentation]] T6 (coding suggestions)
**References:** [[Revenue-Cycle-Deep-Dive]], [[ADR-003-ai-agent-framework]]

**Description:**
Extend the AI coding suggestions from EPIC-004 into a full coding engine that produces billing-ready code sets with compliance checks. This is the bridge between clinical documentation and claims.

**Subtasks:**
- [ ] Implement coding review workflow:
  ```
  AI suggests codes (EPIC-004 T6)
  -> Coder reviews suggestions
  -> Coder accepts/modifies/adds codes
  -> System validates code set
  -> Approved codes feed charge capture (T5)
  ```
- [ ] Implement code set validation:
  - ICD-10 code combination rules (excludes1, excludes2)
  - CPT code bundling rules (CCI edits)
  - Modifier requirements and restrictions
  - Gender-specific code validation
  - Age-specific code validation
  - Diagnosis-procedure linkage validation
- [ ] Implement HCC/RAF score calculation:
  - Calculate RAF score from approved diagnosis codes
  - Identify HCC opportunities (conditions documented but not coded)
  - Flag "HCC recapture" opportunities for annual wellness visits
- [ ] Implement coding queue:
  - Encounters awaiting coding review
  - Priority sorting: oldest first, high-value encounters
  - Coder assignment and workload balancing
  - Productivity metrics: codes reviewed per hour, accuracy rate
- [ ] Implement coding audit trail:
  - Original AI suggestions
  - Coder modifications (with reason)
  - Final approved code set
  - Timestamp and user for each action

**Acceptance Criteria:**
- [ ] Coding validation catches 99%+ of CCI edit violations
- [ ] HCC opportunities identified for documented conditions
- [ ] Coder can review and approve AI suggestions in < 1 minute per encounter
- [ ] Complete audit trail from AI suggestion to approved code
- [ ] Coding queue shows outstanding encounters with priority

---

### T5: Charge Capture Workflow
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T4, [[EPIC-003-fhir-data-layer]] T4 (FHIR CRUD)
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
Implement the charge capture workflow that takes approved clinical codes and creates billable charges. This ensures every rendered service is captured for billing.

**Subtasks:**
- [ ] Design charge data model:
  ```sql
  CREATE TABLE charges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    encounter_id UUID NOT NULL,
    patient_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    service_date DATE NOT NULL,
    cpt_code VARCHAR(10) NOT NULL,
    modifiers VARCHAR(10)[],
    icd10_codes VARCHAR(10)[] NOT NULL,
    units INTEGER NOT NULL DEFAULT 1,
    charge_amount DECIMAL(10,2) NOT NULL,
    allowed_amount DECIMAL(10,2),
    status VARCHAR(20) NOT NULL,
    fee_schedule_id UUID,
    claim_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );
  ```
- [ ] Implement fee schedule management:
  - Practice-specific fee schedules (UCR - usual/customary/reasonable)
  - Medicare fee schedule import (CMS MPFS)
  - Payer-specific contracted rates (by payer contract)
  - Fee schedule lookup by CPT + location + provider
- [ ] Implement auto-charge generation:
  - On encounter coding approval (T4), automatically create charges
  - Apply fee schedule to determine charge amounts
  - Link ICD-10 diagnoses to CPT procedures (pointer)
  - Flag missing charges (E/M without linked diagnoses)
- [ ] Build charge review interface:
  - Daily charge review for billing staff
  - Charges grouped by provider, date, status
  - Bulk actions: approve, hold, void
  - Missing charge alerts
- [ ] Implement charge hold rules:
  - Hold if missing prior authorization
  - Hold if eligibility not verified
  - Hold if diagnosis-procedure linkage invalid
  - Release hold when conditions met
- [ ] Map charges to FHIR Claim resource (for claims generation T6)

**Acceptance Criteria:**
- [ ] Charges auto-generated from coded encounters
- [ ] Fee schedule lookup returns correct charge amount
- [ ] Hold rules prevent premature billing
- [ ] Billing staff can review and approve charges efficiently
- [ ] Every charge traceable to encounter and coding decision
- [ ] FHIR Claim resource generated from approved charges

---

### T6: Claims Generation (X12 837P)
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T5, T1
**References:** [[X12-EDI-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Generate X12 837P (Professional) claims from approved charges. Claims must be compliant with HIPAA-mandated X12 5010 format.

**Subtasks:**
- [ ] Implement X12 837P generator:
  - ISA/GS/ST envelope generation
  - 2000A Billing Provider (practice NPI, tax ID)
  - 2000B Subscriber (insurance subscriber demographics)
  - 2010BA Subscriber Name
  - 2010BB Payer Name
  - 2300 Claim Information:
    - Place of service code
    - Diagnosis codes (up to 12 ICD-10)
    - Total charge amount
    - Prior authorization number (if applicable)
    - Referring provider (if applicable)
  - 2400 Service Line:
    - CPT/HCPCS code + modifiers
    - Service date
    - Charge amount
    - Units
    - Diagnosis pointers
    - Rendering provider NPI
  - SE/GE/IEA trailer segments
- [ ] Implement claim generation from FHIR:
  - Read FHIR Claim resource (from T5)
  - Read FHIR Patient, Coverage, Organization, Practitioner
  - Map FHIR data to X12 segments
  - Store generated 837P as FHIR ClaimResponse
- [ ] Implement claim ID generation and tracking:
  - Internal claim ID (UUID)
  - Payer claim number (assigned on submission)
  - Patient control number
  - Claim status tracking: generated, scrubbed, submitted, accepted, rejected, paid, denied
- [ ] Handle claim types:
  - Original claim
  - Corrected claim (frequency code 7)
  - Void/cancel (frequency code 8)
  - Replacement claim
- [ ] Implement secondary/tertiary claim generation:
  - After primary payer payment, generate secondary claim
  - Include primary payer payment information (COB)
  - Patient responsibility calculation
- [ ] Validate 837P against X12 5010 specification
- [ ] Store raw 837P files for audit (7-year retention)

**Acceptance Criteria:**
- [ ] Generated 837P passes X12 syntax validation
- [ ] Claims include all required segments for top 5 payers
- [ ] Secondary claims include correct COB information
- [ ] Claim IDs tracked across internal and payer systems
- [ ] Raw X12 files stored with audit trail
- [ ] Corrected and void claims generate correctly

---

### T7: Claims Scrubbing Rules Engine
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T6
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
Build a rules engine that scrubs claims before submission, catching errors that would cause denials. Claims that fail scrubbing are returned for correction.

**Subtasks:**
- [ ] Implement scrubbing rule categories:
  - **Patient demographics**: valid name, DOB, gender, subscriber ID
  - **Provider information**: valid NPI, taxonomy code, tax ID
  - **Payer information**: valid payer ID, plan type
  - **Coding rules**:
    - CCI edits (bundling conflicts)
    - MUE limits (maximum units per service)
    - Gender-specific code validation
    - Age-specific code validation
    - Diagnosis-procedure linkage
    - LCD/NCD coverage requirements
  - **Claim-level rules**:
    - Duplicate claim detection (same patient, date, CPT)
    - Timely filing check (within payer's filing limit)
    - Prior authorization verification
    - Referral requirement verification
    - Place of service consistency
- [ ] Implement rule engine architecture:
  ```
  Claim -> [Rule 1] -> [Rule 2] -> ... -> [Rule N] -> Clean / Errors

  Each rule returns:
  - PASS: rule satisfied
  - WARN: potential issue (submit with flag)
  - FAIL: claim must be corrected before submission
  ```
- [ ] Create rule management interface:
  - View all active rules with pass/fail rates
  - Enable/disable rules per payer
  - Custom rule creation (for practice-specific requirements)
  - Rule testing with sample claims
- [ ] Implement scrubbing reports:
  - Daily scrubbing summary: claims processed, clean rate, common errors
  - Error trend analysis (identify systematic issues)
  - Denial prevention estimate (dollars saved by catching errors)
- [ ] Integrate scrubbing into claims workflow:
  ```
  Charge approved -> Claim generated (T6)
  -> Auto-scrub -> Clean: route to submission
                -> Errors: route to correction queue
  ```

**Acceptance Criteria:**
- [ ] Scrubbing catches 95%+ of preventable denial causes
- [ ] Clean claim rate > 90% after scrubbing (vs. industry avg 80%)
- [ ] Scrubbing completes in < 2 seconds per claim
- [ ] Error messages are specific and actionable
- [ ] Rule management allows adding custom rules
- [ ] Scrubbing report shows daily trends

---

### T8: Clearinghouse Integration
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T6, T7
**References:** [[X12-EDI-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Integrate with an electronic clearinghouse (Change Healthcare/Optum or Availity) for claims submission and status tracking. The clearinghouse acts as the intermediary between MedOS and payers.

**Subtasks:**
- [ ] Evaluate and select clearinghouse:
  | Criteria | Change Healthcare | Availity | Trizetto |
  |----------|------------------|----------|----------|
  | Payer coverage | ? | ? | ? |
  | API quality | ? | ? | ? |
  | Pricing | ? | ? | ? |
  | BAA available | ? | ? | ? |
  | ERA/835 support | ? | ? | ? |
  | Real-time status | ? | ? | ? |
- [ ] Implement SFTP/API submission:
  - Configure SFTP connection to clearinghouse
  - Batch submission: upload 837P files on schedule (hourly)
  - Real-time submission: API call per claim (for urgent claims)
  - Implement acknowledgment (999/TA1) processing
  - Handle rejections at clearinghouse level (before payer)
- [ ] Implement claim status inquiry (X12 276/277):
  - `POST /api/v1/claims/{id}/status` - check claim status
  - Batch status check: all pending claims daily
  - Parse 277 response: accepted, rejected, pending, finalized
  - Update claim status in database
- [ ] Implement enrollment management:
  - Practice enrollment with each payer via clearinghouse
  - Provider enrollment verification
  - ERA (Electronic Remittance Advice) enrollment
  - EFT (Electronic Funds Transfer) enrollment
- [ ] Implement error handling and retry:
  - Clearinghouse rejection: parse error, route to correction queue
  - Network failure: retry with exponential backoff
  - Duplicate submission prevention (idempotency keys)
  - Dead letter queue for unresolvable submissions
- [ ] Security and compliance:
  - SFTP with key-based authentication
  - All X12 files encrypted at rest
  - BAA signed with clearinghouse
  - Transaction logging for HIPAA audit

**Acceptance Criteria:**
- [ ] Claims submitted successfully to clearinghouse
- [ ] Acknowledgments (999) processed and claim status updated
- [ ] Claim status inquiry returns current status from payer
- [ ] Clearinghouse rejections routed to correction queue with error details
- [ ] BAA signed and compliance documented
- [ ] Submission latency < 30 seconds for real-time, hourly for batch

---

### T9: Remittance Processing (X12 835)
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T8
**References:** [[X12-EDI-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Process electronic remittance advice (ERA/835) from payers to record payments, adjustments, and denials against claims.

**Subtasks:**
- [ ] Implement X12 835 parser:
  - Parse payment/remittance segments:
    - BPR (Financial Information) - payment amount, method, date
    - TRN (Trace Number) - check/EFT number
    - 2100 Claim Payment Information:
      - CLP (Claim Level) - claim status, charge, payment, patient responsibility
      - CAS (Adjustment) - adjustment reason codes, amounts
      - SVC (Service Level) - line-item payment details
      - AMT (Amounts) - allowed amount, deductible, copay, coinsurance
  - Handle multiple claims per remittance
  - Handle multiple remittances per file
- [ ] Implement payment posting:
  - Match 835 claim payments to internal claims (by patient control number)
  - Post payment amounts to charges
  - Record adjustments with reason codes:
    - Contractual adjustment (CO)
    - Patient responsibility (PR)
    - Other adjustment (OA)
    - Payer initiated reduction (PI)
  - Calculate patient balance
  - Trigger secondary claim generation if applicable
- [ ] Handle payment exceptions:
  - Unmatched payments (835 payment without matching claim)
  - Overpayments
  - Recoupments/takebacks
  - Zero-pay remittances
- [ ] Create payment posting dashboard:
  - Daily remittance summary
  - Unposted payments requiring manual matching
  - Payment variance alerts (expected vs. actual)
  - Collections summary by payer
- [ ] Store payment data as FHIR ClaimResponse / PaymentReconciliation resources
- [ ] Implement bank reconciliation support:
  - Match 835 totals to bank deposits
  - Flag discrepancies

**Acceptance Criteria:**
- [ ] 835 parser handles all standard adjustment reason codes
- [ ] Auto-matching rate > 95% (payment to claim)
- [ ] Patient responsibility calculated correctly
- [ ] Secondary claims triggered automatically when applicable
- [ ] Unmatched payments flagged for manual review
- [ ] Payment posting completes same day as remittance receipt

---

### T10: Denial Tracking and AI Denial Prediction
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T9, [[EPIC-004-ai-clinical-documentation]] T6 (coding engine)
**References:** [[Revenue-Cycle-Deep-Dive]], [[ADR-003-ai-agent-framework]]

**Description:**
Track claim denials, analyze patterns, and build an AI model that predicts denial risk before claims are submitted, enabling pre-emptive correction.

**Subtasks:**
- [ ] Implement denial tracking system:
  ```sql
  CREATE TABLE denials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    claim_id UUID NOT NULL,
    denial_date DATE NOT NULL,
    denial_code VARCHAR(20) NOT NULL,
    denial_reason TEXT NOT NULL,
    remark_code VARCHAR(20),
    denied_amount DECIMAL(10,2) NOT NULL,
    service_line_id UUID,
    category VARCHAR(30),
    status VARCHAR(20) NOT NULL,
    appeal_deadline DATE,
    assigned_to UUID,
    resolution_date DATE,
    resolution_amount DECIMAL(10,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );
  ```
- [ ] Implement denial categorization:
  - Map CARC/RARC codes to categories:
    - Clinical: medical necessity, experimental, not covered
    - Administrative: timely filing, missing info, duplicate
    - Coding: invalid code, unbundling, upcoding
    - Eligibility: inactive, not covered, out of network
    - Authorization: no prior auth, expired auth
  - Auto-categorize based on denial codes
- [ ] Build denial management dashboard:
  - Denial volume by category, payer, provider, time period
  - Aging report: days since denial, approaching appeal deadline
  - Denial rate trends (improving or worsening?)
  - Top denial reasons with actionable recommendations
  - Dollar value at risk (denials approaching write-off)
- [ ] Implement appeal workflow:
  - Auto-generate appeal letter template based on denial reason
  - Attach supporting clinical documentation
  - Track appeal submission and outcome
  - Calculate appeal success rate by category and payer
- [ ] Build AI denial prediction model v1:
  - Features:
    - Payer ID
    - CPT code(s)
    - ICD-10 code(s)
    - Charge amount
    - Prior auth status
    - Provider NPI
    - Historical denial rate for this code + payer combination
    - Documentation completeness score (from EPIC-004 T8)
  - Model: gradient boosted trees (XGBoost) or logistic regression
  - Training data: historical claims with outcomes
  - Output: denial probability (0-1) + predicted denial reason
  - Threshold: flag claims with > 30% denial probability
- [ ] Integrate prediction into claims workflow:
  ```
  Claim generated -> Scrubbed (T7) -> AI prediction
  -> Low risk: submit normally
  -> High risk: flag for review, show predicted denial reason + recommended fix
  ```

**Acceptance Criteria:**
- [ ] All denials categorized automatically from CARC/RARC codes
- [ ] Denial dashboard shows trends and actionable insights
- [ ] Appeal workflow tracks submission through resolution
- [ ] AI prediction model identifies 70%+ of eventual denials before submission
- [ ] Flagged claims show specific recommended corrections
- [ ] Denial rate demonstrably decreases over 30-day measurement period

---

### T11: Revenue Analytics Dashboard
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T5, T9, T10
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
Build a comprehensive revenue analytics dashboard for practice managers and billing staff to monitor financial health and identify optimization opportunities.

**Subtasks:**
- [ ] Implement key revenue metrics:
  | Metric | Description | Target |
  |---|---|---|
  | Days in AR | Average days from service to payment | < 35 |
  | Clean claim rate | % claims passing scrubbing first time | > 95% |
  | First pass resolution rate | % claims paid on first submission | > 90% |
  | Denial rate | % claims denied (partial or full) | < 5% |
  | Collection rate | Collected / Allowed amount | > 96% |
  | Cost to collect | Billing cost / Collections | < 4% |
  | Net collection rate | (Payments + adjustments) / (Charges - contractual) | > 98% |
- [ ] Build dashboard views:
  - **Executive summary**: headline metrics with trend indicators
  - **Revenue waterfall**: charges -> allowed -> payments -> adjustments -> denials -> AR
  - **Payer performance**: metrics broken down by insurance payer
  - **Provider performance**: charges and collections by provider
  - **Aging buckets**: AR aging (0-30, 31-60, 61-90, 91-120, 120+ days)
  - **Denial analysis**: denial rates by category with drill-down
  - **Projection**: 30/60/90 day revenue forecast based on pipeline
- [ ] Implement date range filtering:
  - Daily, weekly, monthly, quarterly, yearly views
  - Custom date range
  - Comparison periods (this month vs. last month, YoY)
- [ ] Implement drill-down capability:
  - Click metric -> see contributing claims
  - Click payer -> see payer-specific detail
  - Click provider -> see provider-specific detail
- [ ] Implement automated reporting:
  - Weekly email summary to practice manager
  - Monthly financial report (PDF export)
  - Alert on metric threshold breach (denial rate > 10%)
- [ ] Data visualization with charts:
  - Line charts for trends over time
  - Bar charts for comparisons
  - Pie charts for category breakdowns
  - Sankey diagram for revenue flow

**Acceptance Criteria:**
- [ ] All key metrics calculated correctly and in real-time
- [ ] Dashboard loads in < 3 seconds
- [ ] Drill-down from metric to individual claims works
- [ ] Date range filtering provides accurate comparative data
- [ ] Weekly automated reports delivered on schedule
- [ ] PDF export generates clean, printable report

---

### T12: Integration Testing
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1-T11
**References:** [[Revenue-Cycle-Deep-Dive]]

**Description:**
End-to-end integration testing of the complete revenue cycle from encounter documentation to payment posting.

**Subtasks:**
- [ ] Create test scenario: full revenue cycle walkthrough:
  1. Patient checks in -> eligibility verified (T1)
  2. Provider conducts encounter -> AI documentation (EPIC-004)
  3. AI suggests codes -> coder reviews and approves (T4)
  4. Charges captured automatically (T5)
  5. Claim generated and scrubbed (T6, T7)
  6. Claim submitted to clearinghouse (T8)
  7. Remittance received and posted (T9)
  8. Patient statement generated
  9. Analytics reflect completed cycle (T11)
- [ ] Test with synthetic data for each major payer
- [ ] Test denial scenarios:
  - Missing prior authorization
  - Inactive coverage
  - Coding error
  - Duplicate claim
  - Timely filing violation
- [ ] Test secondary billing flow
- [ ] Load test: 500 claims/day throughput
- [ ] Verify HIPAA audit trail covers entire cycle
- [ ] Verify all X12 transactions stored for compliance

**Acceptance Criteria:**
- [ ] Full cycle completes without manual intervention for clean claims
- [ ] Denial scenarios handled gracefully with clear user guidance
- [ ] 500 claims/day throughput achieved without performance degradation
- [ ] Audit trail complete from encounter to payment
- [ ] All X12 files retrievable for 7-year retention compliance

---

## Dependencies Map

```
T1 (Eligibility) ──> T2 (PA Tracking) ──> T3 (PA Automation)
                └──> T5 (Charge Capture) ──> T6 (Claims 837P) ──> T7 (Scrubbing)
T4 (Coding) ──────┘                                            └──> T8 (Clearinghouse)
                                                                      └──> T9 (Remittance)
                                                                            └──> T10 (Denials)
                                                              T5 + T9 + T10 ──> T11 (Analytics)
                                                              T1-T11 ──> T12 (Integration Test)
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| Revenue analytics (T11) | [[EPIC-006-pilot-readiness]] (success metrics) |
| Denial tracking (T10) | [[EPIC-006-pilot-readiness]] (pilot KPIs) |
| Full billing pipeline (T1-T9) | [[EPIC-006-pilot-readiness]] (pilot operational readiness) |
| Eligibility verification (T1) | [[EPIC-006-pilot-readiness]] (front desk workflow) |

| This Epic Requires | Provided By |
|---|---|
| AI coding suggestions | [[EPIC-004-ai-clinical-documentation]] T6 |
| FHIR resources from notes | [[EPIC-004-ai-clinical-documentation]] T7 |
| FHIR CRUD + search | [[EPIC-003-fhir-data-layer]] T4, T5 |
| Auth + API keys | [[EPIC-002-auth-identity-system]] T2, T6 |
| ECS + S3 + KMS | [[EPIC-001-aws-infrastructure-foundation]] T5, T7, T4 |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Clearinghouse onboarding takes 4-8 weeks | High | Critical | Start clearinghouse evaluation and contracting in Week 1; use clearinghouse sandbox for development |
| X12 format complexity causes claim rejections | High | High | Use established X12 library; test with clearinghouse validation tools; start with single payer |
| Payer enrollment delays for pilot practices | High | Medium | Begin enrollment process 60 days before pilot; prioritize top 5 payers by volume |
| AI denial prediction insufficient training data | Medium | Medium | Start with rule-based prediction using known denial patterns; add ML model as data accumulates |
| Fee schedule data management complexity | Medium | Medium | Start with Medicare fee schedule; add commercial payer schedules incrementally |
| Real-time eligibility API rate limits | Medium | Low | Implement caching; batch verification for scheduled appointments; respect rate limits |

---

## Definition of Done

- [ ] Eligibility verification returns real-time results for top 5 payers
- [ ] Prior auth tracking dashboard operational with status workflow
- [ ] AI-assisted prior auth reduces preparation time by 50%+
- [ ] Claims generated, scrubbed, and submitted electronically
- [ ] Clean claim rate > 90%
- [ ] Remittance auto-posts payments with > 95% match rate
- [ ] Denial tracking categorizes and routes denials for follow-up
- [ ] AI denial prediction flags high-risk claims before submission
- [ ] Revenue dashboard shows all key metrics with drill-down
- [ ] Full revenue cycle tested end-to-end with synthetic data
- [ ] All X12 transactions stored for 7-year HIPAA retention
