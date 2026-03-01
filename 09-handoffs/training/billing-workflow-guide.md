---
type: training
date: "2026-02-28"
tags:
  - training
  - billing
  - revenue-cycle
  - phase-1
---

# Billing Staff Workflow Guide -- MedOS

> Step-by-step guide for billing staff managing the revenue cycle in MedOS: eligibility verification, code review, claims submission, tracking, denial management, and analytics.

---

## Prerequisites

- MedOS account with Billing Staff role assigned
- Multi-factor authentication (MFA) configured
- Familiarity with ICD-10, CPT, and X12 EDI basics (see [[Revenue-Cycle-Deep-Dive]])
- Payer contract information configured by administrator

---

## Workflow Steps

### Step 1: Verify Patient Eligibility

1. Navigate to **Revenue Cycle > Eligibility** from the main menu.
2. Select the patient from the upcoming schedule or search by name/MRN.
3. Click **Verify Eligibility**. MedOS sends a real-time X12 270 inquiry to the patient's payer via the clearinghouse.
4. Review the X12 271 response: coverage status, copay, deductible remaining, authorization requirements.
5. Flag any issues (inactive coverage, high deductible, prior auth required) before the encounter.

[Screenshot: Eligibility verification panel showing coverage details]

**Expected result:** Eligibility response returned within 15 seconds. Coverage details displayed with clear pass/fail indicators.

### Step 2: Review AI-Suggested Codes

1. After a provider finalizes an encounter, it appears in the **Billing Queue** (Revenue Cycle > Billing Queue).
2. Open the encounter to view the signed SOAP note and AI-suggested ICD-10 and CPT codes.
3. Codes with confidence below 85% are highlighted for manual review.
4. Verify code accuracy against the clinical documentation. Modify, add, or remove codes as needed.
5. Click **Approve Codes** to move the encounter to claim generation.

[Screenshot: Billing Queue with encounter list and code review panel]

**Expected result:** Most routine encounters have codes above 85% confidence and require minimal adjustment.

### Step 3: Submit Claims

1. Navigate to **Revenue Cycle > Claims**.
2. Approved encounters are listed as "Ready to Submit."
3. Review the claim summary (patient, provider, codes, charges, payer).
4. Click **Submit Claim**. MedOS generates an X12 837P Professional Claim and transmits it to the clearinghouse (Availity).
5. Track submission status: Submitted, Accepted, Rejected, Paid, Denied.

[Screenshot: Claims list with status indicators and Submit Claim button]

**Expected result:** Claim submitted and accepted by clearinghouse within minutes. Status updates automatically.

### Step 4: Track Claim Status

1. The **Claims Dashboard** shows all claims with real-time status.
2. Filter by status (Pending, Accepted, Rejected, Paid, Denied), date range, payer, or provider.
3. Click any claim to view details including the X12 835 remittance advice when available.
4. Payment postings are automated from X12 835 ERA data.

[Screenshot: Claims Dashboard with filters and status breakdown]

### Step 5: Manage Denials

1. Denied claims appear in **Revenue Cycle > Denials**.
2. Each denial shows the denial reason code, payer explanation, and the original claim.
3. Click **Review Denial** to see the AI-generated analysis: likely cause, suggested corrective action, and appeal recommendation.
4. Choose to **Correct and Resubmit** (for coding/billing errors) or **Appeal** (for clinical disagreements).
5. For appeals, the AI drafts an appeal letter referencing clinical documentation. Review, edit, and submit.

[Screenshot: Denial management view with AI appeal suggestion]

**Expected result:** AI identifies denial root cause with recommended action. Appeal letters reference specific clinical evidence.

### Step 6: Run Analytics

1. Navigate to **Revenue Cycle > Analytics** for financial performance reports.
2. Available reports: Days in A/R, Clean Claim Rate, Denial Rate by Payer, Collection Rate, Revenue by Provider/CPT.
3. Filter by date range, provider, payer, or encounter type.
4. Export reports as CSV or PDF for practice meetings.

[Screenshot: Analytics dashboard with key revenue cycle metrics]

---

## Prior Authorization Workflow

1. When eligibility check indicates prior auth is required, a task appears in **Revenue Cycle > Prior Auth**.
2. Review the clinical documentation and the AI-drafted authorization request.
3. Submit the prior auth via the payer's portal or fax (tracked in MedOS).
4. Track status: Submitted, Approved, Denied, Peer-to-Peer Required.
5. For denials, initiate the peer-to-peer review process with the provider.

See [[Prior-Authorization-Deep-Dive]] for detailed prior auth workflows.

---

## Appeal Workflow

1. From the denial view, click **Initiate Appeal**.
2. Select appeal level: First-level (internal), Second-level (external), or State Insurance Commissioner.
3. AI drafts the appeal with clinical evidence, coding rationale, and payer policy references.
4. Review and edit the appeal letter. Attach supporting documentation.
5. Submit and track through resolution.

---

## Common Issues and FAQ

**Q: Eligibility check returns "no match found."**
A: Verify the patient's insurance ID, date of birth, and subscriber information match exactly what the payer has on file.

**Q: A claim was rejected by the clearinghouse (not denied by payer).**
A: This is usually a formatting error. Check the rejection reason code. Common causes: invalid NPI, missing taxonomy code, incorrect payer ID.

**Q: The AI-suggested codes look unusual for this encounter type.**
A: The AI model learns from practice patterns. For the first few weeks, expect more manual corrections. Your corrections improve future suggestions.

---

## References

- [[Revenue-Cycle-Deep-Dive]] -- Complete RCM lifecycle
- [[Prior-Authorization-Deep-Dive]] -- Prior auth automation details
- [[X12-EDI-Deep-Dive]] -- X12 270/271, 837P, 835 format reference
