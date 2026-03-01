---
type: training
date: "2026-02-28"
tags:
  - training
  - provider
  - clinical
  - phase-1
---

# Provider Workflow Guide -- MedOS

> Step-by-step guide for providers using MedOS during patient encounters. This covers login, encounter management, AI scribe usage, SOAP note review, coding approval, and sign-off.

---

## Prerequisites

- MedOS account with Provider role assigned
- Multi-factor authentication (MFA) configured on your Auth0 account
- Modern browser (Chrome, Edge, or Firefox -- latest version)
- Microphone access enabled in browser settings (for AI scribe)

---

## Workflow Steps

### Step 1: Log In

1. Navigate to the MedOS portal URL provided by your practice administrator.
2. Enter your email and password.
3. Complete MFA verification (authenticator app or SMS code).
4. You will land on the **Provider Dashboard** showing today's schedule.

[Screenshot: Provider Dashboard with today's appointments listed]

**Expected result:** Dashboard displays your name, practice, and scheduled patients.

### Step 2: Start an Encounter

1. From the dashboard, click on the patient's name in today's schedule.
2. Review the **Patient Summary** panel: demographics, active conditions, recent encounters, medications, and allergies.
3. Click **Start Encounter** to begin a new clinical session.

[Screenshot: Patient Summary panel with Start Encounter button]

**Expected result:** Encounter timer starts. A new encounter record is created with status "in-progress."

### Step 3: Activate the AI Scribe

1. In the encounter view, click the **AI Scribe** toggle (microphone icon) in the toolbar.
2. Grant microphone permission if prompted by your browser.
3. Speak naturally with the patient. The scribe listens in real-time and displays a live transcript in the side panel.
4. You can pause and resume the scribe at any time using the toggle.

[Screenshot: Encounter view with AI Scribe active, transcript panel visible]

**Expected result:** Live transcript appears as you speak. The scribe indicator shows "Listening..." status.

### Step 4: Review the SOAP Note

1. When the conversation is complete, click **Generate SOAP Note**.
2. The AI processes the transcript and produces a structured SOAP note (Subjective, Objective, Assessment, Plan).
3. Each section displays a **confidence score** (percentage). Sections below 85% are highlighted for your attention.
4. Review each section for accuracy. Click any section to edit directly.

[Screenshot: Generated SOAP note with confidence scores per section]

**Expected result:** SOAP note generated within 30 seconds. All four sections populated with relevant clinical content.

### Step 5: Review Suggested Codes

1. Below the SOAP note, review the **AI-Suggested Codes** panel showing ICD-10 diagnosis codes and CPT procedure codes.
2. Each code shows a confidence score. Codes below 85% are flagged for review.
3. Accept, reject, or modify codes as clinically appropriate.
4. Add additional codes manually if needed using the search field.

[Screenshot: Suggested codes panel with confidence indicators]

**Expected result:** Relevant ICD-10 and CPT codes listed with confidence scores. Most codes should be above 85% for common encounters.

### Step 6: Sign Off

1. Once you are satisfied with the SOAP note and coding, click **Sign and Finalize**.
2. Confirm your electronic signature in the dialog.
3. The encounter status changes to "completed" and the documentation is locked for billing.

[Screenshot: Sign and Finalize confirmation dialog]

**Expected result:** Encounter finalized. Documentation locked. Billing staff can now process the claim.

---

## Common Issues and FAQ

**Q: The AI scribe is not picking up my voice.**
A: Check that your browser has microphone permission. Try a different microphone input in browser settings. Ensure no other application is using the microphone.

**Q: The SOAP note has low confidence scores across all sections.**
A: This usually indicates poor audio quality. Ensure you are in a quiet room and speaking clearly. Background noise significantly reduces transcription accuracy.

**Q: I need to edit the note after signing.**
A: Contact your administrator. Signed notes can be amended (not deleted) with a documented reason. The original and amendment are both retained in the audit trail.

**Q: The suggested codes seem wrong for this encounter type.**
A: The AI coding model improves with practice-specific data over time. Always apply your clinical judgment. Reject incorrect codes and add the correct ones -- this feedback trains the system.

**Q: I need to access a patient's records in an emergency but they are not on my schedule.**
A: Use the **Break-the-Glass** feature in the patient search. You must provide a justification reason. This access is logged and reviewed by the security team.

---

## References

- [[Clinical-Workflows-Overview]] -- Full patient journey map
- [[Ambient-AI-Documentation]] -- AI scribe technical pipeline
- [[Auth-SMART-on-FHIR]] -- Authentication and access control
