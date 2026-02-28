---
type: domain-knowledge
date: "2026-02-27"
tags:
  - domain
  - clinical
  - workflows
  - module-b
  - module-f
category: clinical
confidence: high
sources: []
---

# Clinical Workflows Overview

> A comprehensive mapping of clinical workflows in U.S. outpatient practice -- from patient scheduling through final billing. This document identifies every touchpoint, role responsibility, pain point, and AI automation opportunity for the [[HEALTHCARE_OS_MASTERPLAN]]. Written for a technical team building a Healthcare OS, not for clinicians. Every workflow described here is a candidate for Module B (Workflow Engine) or Module F (Patient Engagement) automation.

Related: [[HEALTHCARE_OS_MASTERPLAN]] | [[Ambient-AI-Documentation]] | [[Revenue-Cycle-Deep-Dive]] | [[FHIR-R4-Deep-Dive]] | [[Prior-Authorization-Deep-Dive]] | [[MOC-Domain-Knowledge]]

---

## 1. Patient Journey Map

The patient journey through an outpatient encounter consists of discrete phases, each involving different staff, systems, and data flows. Understanding this end-to-end is essential because MedOS must integrate across every phase -- not just the clinical encounter.

```
SCHEDULING -> PRE-VISIT -> CHECK-IN -> ROOMING -> PROVIDER VISIT -> CHECKOUT -> POST-VISIT -> BILLING
   (days before)  (24-48h)   (arrival)  (5-10 min) (15-40 min)    (5-10 min)  (same day)   (days-months)
```

**Touchpoints and responsible roles:**

| Phase | Primary Staff | Systems Used | Data Created |
|-------|--------------|-------------|--------------|
| Scheduling | Front desk / Patient (self-schedule) | EHR scheduler, phone, patient portal | Appointment, referral link |
| Pre-visit | Front desk / Automated system | Insurance verification, patient portal | Eligibility, intake forms |
| Check-in | Front desk | EHR, payment terminal | Demographics verification, copay |
| Rooming | Medical Assistant (MA) / Nurse | EHR, vitals devices | Vitals, chief complaint, medications |
| Provider Visit | Physician / APP | EHR, ambient AI, imaging viewer | Clinical note (SOAP), orders |
| Checkout | Front desk / MA | EHR, scheduling | Follow-up appt, referrals, patient instructions |
| Post-visit | Billing / Coding / Nursing | EHR, billing system, pharmacy, lab | Prescriptions, lab orders, referral letters |
| Billing | Billing staff | Practice management, clearinghouse | Claims (X12 837), ERA (X12 835) |

Each handoff between phases is a potential failure point. Information lost at check-in (wrong insurance ID) cascades into claim denials weeks later. A missed referral at checkout delays patient care. MedOS must ensure data integrity across every transition.

---

## 2. Pre-Visit Workflow

The pre-visit workflow begins when the appointment is scheduled and ends when the patient arrives. This phase is largely administrative but directly impacts clinical efficiency and revenue capture.

### 2.1 Scheduling

**Current state**: Scheduling is done via phone (60-70% of appointments), patient portal (20-30%), or in-person at checkout (10-15%). The front desk staff manages the schedule using the EHR's scheduling module, which typically provides a grid view of provider availability by day and time slot.

**Complexity factors:**
- **Appointment types**: New patient (longer slot, 30-60 min), established patient (shorter, 15-20 min), procedure (variable, 30-120 min), follow-up, urgent/same-day. Each type requires a different time slot length.
- **Provider preferences**: Some providers want 15-minute slots, others want 20. Some block Tuesdays for surgery. Some want a "catch-up" slot every 2 hours.
- **Resource constraints**: Procedures may require specific exam rooms (with procedure tables), equipment (ultrasound, casting materials), or additional staff (MA for assistance).
- **Insurance requirements**: Some payers require a referral before a specialist visit. The scheduler must verify that a referral exists or tell the patient to obtain one.

**MedOS opportunity**: ML-based scheduling optimization. Predict no-shows (typically 10-20% in outpatient) using historical data, patient demographics, day of week, weather, and distance from clinic. Overbook strategically to maximize provider utilization without creating excessive wait times. Self-scheduling via patient portal with smart availability that accounts for appointment type, provider preference, and resource availability.

### 2.2 Insurance Eligibility Verification

Before the patient arrives, the practice must verify that their insurance is active and determine the patient's financial responsibility (copay, deductible remaining, coinsurance). See [[Revenue-Cycle-Deep-Dive]] Section 2.2 for the X12 270/271 transaction details.

**Current state**: Staff manually checks eligibility via payer web portals (each payer has a different portal with different login credentials) or via a clearinghouse batch submission. This takes 3-5 minutes per patient. For a practice seeing 100 patients/day, that is 5-8 hours of staff time.

**MedOS opportunity**: Automated batch eligibility verification. Every evening, the system pulls the next day's schedule and runs X12 270 eligibility inquiries for all scheduled patients. Results are stored and surfaced to front desk staff in the morning. Exceptions (inactive coverage, high deductible) are flagged for proactive patient contact.

### 2.3 Intake Forms and Pre-Visit Questionnaires

Patients complete intake forms covering demographics, medical history, current medications, allergies, surgical history, family history, social history, and reason for visit. Traditionally done on paper clipboards in the waiting room. Increasingly done digitally via patient portal 24-48 hours before the visit.

**MedOS opportunity**: Digital intake via Module F (Patient Engagement). Pre-populate known fields from the patient's FHIR record. Send the questionnaire via SMS 48 hours before the visit. Responses flow directly into the clinical note as structured data, pre-populating the Subjective section for the [[Ambient-AI-Documentation]] system.

### 2.4 Prior Authorization (if required)

For certain services (advanced imaging, specialty medications, surgical procedures), the payer requires prior authorization before the service can be rendered. See [[Prior-Authorization-Deep-Dive]] for the complete workflow.

**Timing is critical**: If the prior auth is not obtained before the visit, the patient may arrive only to learn that their procedure cannot be performed that day. This wastes the patient's time, the provider's time, and the practice's revenue. Best practice is to identify PA requirements at scheduling time and initiate the PA process immediately.

---

## 3. Check-In Workflow

The check-in workflow occurs when the patient arrives at the practice. It is typically managed by front desk staff and takes 5-15 minutes.

### 3.1 Demographics Verification

The front desk confirms the patient's name, date of birth, address, phone number, email, emergency contact, and insurance information. In many practices, this is done by handing the patient a printed sheet of their demographics and asking them to mark any changes. In more modern practices, it is done on a tablet or kiosk.

**Why this matters**: A single character error in a subscriber ID, date of birth, or name spelling will cause a claim denial downstream. Roughly 30-40% of claim denials trace back to registration errors. See [[Revenue-Cycle-Deep-Dive]] for the financial impact.

**MedOS opportunity**: Patient check-in kiosk or mobile check-in (geofenced -- patient checks in when they arrive in the parking lot). Photo ID capture with OCR for name/DOB verification. Insurance card photo capture with OCR for payer ID, group number, and subscriber ID extraction. Real-time eligibility re-verification at check-in.

### 3.2 Copay Collection

The front desk collects the patient's copay (and sometimes outstanding balance) at check-in. Copay amounts are determined by the eligibility verification response (X12 271). Common amounts: $20-$50 for primary care, $30-$75 for specialists.

**Current pain point**: Many practices fail to collect copays at check-in (the patient "forgot their wallet," the system does not display the copay amount, the front desk is too busy). Uncollected copays are expensive to collect after the visit -- it costs $8-$12 in billing staff time and postage to collect a $30 copay by mail.

**MedOS opportunity**: Display copay amount prominently in the check-in workflow (from the eligibility verification). Support card-on-file (store payment method securely and charge after the visit if the patient prefers). Support Apple Pay/Google Pay at the kiosk. Automatically generate a patient statement for any outstanding balance and send via email/text.

### 3.3 Consent Forms

Patients sign consent forms for treatment, HIPAA privacy practices, financial responsibility, and (in MedOS practices) consent for ambient AI recording. Traditionally done on paper. Increasingly digital signatures on tablets.

**MedOS opportunity**: Electronic consent management. Consent forms are FHIR Consent resources. Once signed, the consent is active and does not need to be re-signed at every visit (unless the policy changes). The system tracks which consents are active, which are expiring, and which are missing. For ambient AI recording consent, the system will not activate the recording unless a valid consent is on file.

---

## 4. Clinical Encounter

The clinical encounter is the core of the patient journey. It typically consists of rooming, vitals, provider visit, documentation, and ordering.

### 4.1 Rooming

The Medical Assistant (MA) or nurse calls the patient from the waiting room, walks them to the exam room, and prepares them for the provider. This includes:

- **Vital signs**: Blood pressure, heart rate, respiratory rate, temperature, oxygen saturation, height, weight (BMI calculated). Typically entered into the EHR manually, though some practices use connected devices that transmit directly.
- **Chief complaint**: "What brings you in today?" The MA documents a brief CC in the EHR.
- **Medication reconciliation**: Review the patient's current medication list. "Are you still taking lisinopril 10mg daily?" Confirm, add, or discontinue medications. This is a regulatory requirement (The Joint Commission, CMS).
- **Allergy verification**: "Any new allergies since your last visit?"
- **Pre-visit questionnaires**: If the patient completed intake forms digitally, the MA reviews them. If not, the MA asks the questions.

**Time**: 5-10 minutes. The MA then notifies the provider that the patient is ready.

**MedOS opportunity**: Connected vitals devices that auto-populate the EHR (Bluetooth BP cuff, pulse oximeter, smart scale). Digital rooming workflow in the MedOS MA workspace. Pre-populated medication and allergy lists from the FHIR data layer. The MA confirms or updates rather than re-entering from scratch.

### 4.2 Provider Visit

The provider enters the exam room and conducts the clinical encounter. This is the 15-40 minute interaction where the actual medicine happens. The provider:

1. Reviews the patient's reason for visit and intake information
2. Conducts the history (HPI, ROS, PMH, etc.)
3. Performs a physical examination
4. Reviews available results (labs, imaging)
5. Formulates an assessment (diagnosis/differential)
6. Develops a plan (medications, tests, referrals, follow-up)
7. Provides patient education
8. Documents the encounter

Step 7 and 8 are where the majority of provider time is wasted. Documentation historically happens either during the visit (provider types while talking to the patient -- reduces eye contact and rapport) or after the visit (adds 5-15 minutes of "chart time" between patients, extending the day).

**MedOS opportunity**: This is where [[Ambient-AI-Documentation]] transforms the encounter. The provider focuses entirely on the patient. The AI handles documentation. Orders are placed via voice ("order a CBC and metabolic panel" is captured and converted to structured orders). Prescriptions are initiated from the conversation ("let's start metformin 500mg twice a day").

### 4.3 Orders

During or after the visit, the provider places orders:

- **Laboratory tests**: CBC, BMP, lipid panel, HbA1c, urinalysis, etc. Orders are transmitted to the lab (LabCorp, Quest, or in-house) via HL7v2 ORM messages or FHIR ServiceRequest.
- **Imaging**: X-ray, MRI, CT, ultrasound. Orders go to the imaging facility or radiology department.
- **Prescriptions**: Transmitted electronically to the pharmacy via NCPDP SCRIPT standard (e-prescribing). Controlled substances require EPCS (Electronic Prescribing for Controlled Substances) with two-factor authentication.
- **Referrals**: To other specialists. Referral letter generated, sent to the specialist's office (often still via fax), and may require prior authorization.
- **Procedures**: Scheduled for a future date (surgery, injection, biopsy).

**MedOS opportunity**: Voice-initiated ordering from the ambient AI transcript. The provider says "let's get an MRI of the left knee" during the conversation. The system creates a draft ServiceRequest, checks if prior auth is required ([[Prior-Authorization-Deep-Dive]]), and initiates the PA workflow automatically if needed.

---

## 5. Post-Visit Workflow

### 5.1 Checkout

After the visit, the patient returns to the front desk or a checkout station. The checkout workflow includes:

- **Follow-up scheduling**: The provider's plan specifies "follow up in 6 weeks." The front desk schedules the next appointment.
- **Referral coordination**: If the provider ordered a referral, the front desk (or referral coordinator) identifies an appropriate specialist, checks the specialist is in-network for the patient's insurance, and schedules or provides scheduling information.
- **Patient instructions**: After Visit Summary (AVS) printed or sent electronically. Includes visit summary, new medications, test orders, follow-up instructions.
- **Balance collection**: If the patient owes a balance beyond the copay (e.g., for a procedure or toward their deductible), the checkout staff attempts to collect.

### 5.2 Prescriptions

The provider's prescription orders are transmitted to the pharmacy via e-prescribing. The pharmacy dispenses the medication. Common failure points:

- **Prior authorization required**: The pharmacy runs the claim against the patient's drug benefit and discovers the medication requires PA. The pharmacy contacts the practice. The practice must submit a PA to the payer. This can delay medication access by days.
- **Formulary issues**: The prescribed medication is not on the patient's formulary (covered drug list). The pharmacy suggests an alternative. The provider must approve the change.
- **Patient cost**: The patient's copay for the medication is unexpectedly high. The patient calls the practice to ask for a cheaper alternative.

**MedOS opportunity**: Real-time formulary checking during the encounter. Before the provider finishes talking, the system has already checked the patient's formulary and flagged any issues. "Dr. Smith, brand-name Eliquis requires PA for this patient's plan. Generic apixaban is covered at Tier 2, $35 copay. Would you like to prescribe the generic?"

### 5.3 Referral Processing

Referrals are one of the most broken workflows in outpatient practice. The provider says "I'm referring you to an orthopedic surgeon." Then:

1. Someone in the practice (referral coordinator, MA, or front desk) must identify an appropriate specialist
2. Check that the specialist is in-network for the patient's insurance
3. Check if the payer requires a referral authorization
4. Send the referral (clinical notes, imaging, etc.) to the specialist -- often via fax
5. Schedule the appointment (or give the patient the specialist's number and hope they call)
6. Track whether the patient actually saw the specialist (referral loop closure)

30-50% of referrals are never completed -- the patient never sees the specialist. This represents both a care quality failure and a revenue leakage point (downstream procedures and follow-ups never happen).

**MedOS opportunity**: AI-assisted referral coordination. The system identifies in-network specialists, checks availability, sends clinical records electronically (FHIR-based referral exchange), and tracks referral completion. If the patient has not scheduled with the specialist within 7 days, an automated reminder is sent via Module F (Patient Engagement).

### 5.4 Billing and Coding

After the provider signs the note, the encounter enters the revenue cycle. See [[Revenue-Cycle-Deep-Dive]] for the complete flow. Key steps:

1. **Coding**: A coder (or the provider) assigns ICD-10 diagnosis codes and CPT procedure codes based on the clinical note. MedOS AI suggests codes from the ambient documentation pipeline.
2. **Charge capture**: Charges are entered into the practice management system.
3. **Claim generation**: An X12 837 Professional claim is generated and submitted to the clearinghouse.
4. **Adjudication**: The payer processes the claim (3-30 days).
5. **Payment posting**: ERA (X12 835) is received and posted.
6. **Patient billing**: Any patient responsibility (deductible, coinsurance) is billed to the patient.
7. **Denial management**: Denied claims are investigated and appealed.

---

## 6. Specialty Variations

### 6.1 Orthopedics

Orthopedic workflows differ from primary care in several important ways:

**Imaging-heavy**: Nearly every orthopedic visit involves imaging review. X-rays are often taken on-site before the provider sees the patient. The workflow adds a "pre-visit imaging" step between rooming and the provider visit. MRI and CT are ordered frequently, requiring prior authorization for most payers.

**Surgical scheduling**: A significant percentage of orthopedic encounters result in surgical scheduling. The surgical scheduling workflow involves: booking OR time at a surgery center or hospital, coordinating with anesthesia, obtaining surgical prior authorization (which is separate from and more complex than imaging PA), pre-operative testing (labs, EKG, clearance from PCP), patient pre-op education, and post-op follow-up planning.

**Physical therapy referrals**: Nearly universal in orthopedics. PT referrals may require prior authorization. The number of approved sessions varies by payer (6, 12, 20 sessions). Session extensions require additional PA submissions.

**Procedure-heavy office visits**: Orthopedic offices frequently perform in-office procedures: joint injections (corticosteroid, hyaluronic acid, PRP), fracture reduction, casting/splinting, DME fitting (braces, walking boots). Each procedure requires separate CPT coding and may require PA.

**Documentation patterns**: Orthopedic SOAP notes emphasize musculoskeletal exam findings (range of motion measurements in degrees, strength grading 0-5, special tests like McMurray's, Lachman's, drawer tests), imaging interpretation, and surgical planning.

### 6.2 Dermatology

**Photographic documentation**: Dermatology is uniquely visual. Providers photograph lesions at every visit to track changes over time. Photos must be stored in the medical record, linked to the encounter, and organized by body location. Photo management is a significant workflow step not present in most other specialties.

**Pathology workflow**: Biopsies are common in dermatology. The biopsy workflow involves: obtaining the specimen, labeling it, completing a pathology requisition form, sending it to the lab, receiving the pathology report (days later), reviewing the report, and contacting the patient with results. If malignancy is found, a treatment plan must be developed and may involve surgical excision, Mohs surgery, or referral to oncology.

**Procedure volume**: Dermatology is one of the most procedure-heavy specialties. A single visit may involve multiple biopsies, cryotherapy (liquid nitrogen), excisions, and cosmetic procedures. Each procedure requires separate documentation and coding.

**Cosmetic vs. medical**: Dermatology practices often offer both medical (insurance-billed) and cosmetic (patient-pay) services. The workflow must cleanly separate these, as cosmetic services have different consent forms, pricing, payment collection, and are not submitted to insurance. The EHR/PMS must handle both revenue streams.

### 6.3 Primary Care

**Breadth over depth**: Primary care encounters cover the widest range of conditions of any specialty. A single visit may address diabetes management, medication refills, a new skin rash, depression screening, preventive care (immunizations, cancer screening), and social determinants of health. The documentation must capture all of these.

**Preventive care protocols**: Primary care is the primary delivery point for evidence-based preventive care: immunization schedules (CDC recommendations), cancer screenings (mammography, colonoscopy, Pap smear, lung cancer CT), cardiovascular risk assessment, diabetes screening, depression screening (PHQ-9), fall risk assessment (for elderly). The practice must track which screenings are due for each patient and close care gaps.

**Chronic disease management**: Primary care manages the most chronic conditions per patient -- hypertension, diabetes, hyperlipidemia, obesity, COPD, asthma, depression, anxiety. Each condition has quality measures (HEDIS, MIPS) that the practice must report on. Documentation must support these measures.

**Referral hub**: Primary care generates more referrals than any other specialty. The PCP coordinates the patient's overall care, managing referrals to specialists, tracking specialist reports, and ensuring continuity.

---

## 7. Pain Points by Role

### 7.1 Physicians / Advanced Practice Providers

| Pain Point | Severity | Frequency | MedOS Module |
|------------|----------|-----------|-------------|
| Documentation burden (charting) | Critical | Every encounter | Module B - [[Ambient-AI-Documentation]] |
| Prior authorization phone calls | High | Daily (specialty-dependent) | Module D - [[Prior-Authorization-Deep-Dive]] |
| Inbox overload (results, messages, refills) | High | Daily | Module B - AI inbox triage |
| Coding complexity and audit anxiety | Medium | Every encounter | Module C - AI coding engine |
| Alert fatigue from CDS | Medium | Every encounter | Module B - Smart CDS |
| Staying current with guidelines | Medium | Ongoing | Module E - Evidence integration |
| Peer-to-peer reviews with payers | High | Weekly | Module D - AI PA agent |

**The documentation burden is the dominant pain point.** Studies consistently show that physicians rank documentation as their top frustration. The average physician spends 16 minutes per encounter on documentation. For a 20-patient day, that is over 5 hours of documentation -- often done after hours. This is the problem that [[Ambient-AI-Documentation]] solves directly.

### 7.2 Nurses / Medical Assistants

| Pain Point | Severity | Frequency | MedOS Module |
|------------|----------|-----------|-------------|
| Phone tag with patients | High | All day | Module F - Patient comms agent |
| Manual data entry (vitals, intake) | Medium | Every patient | Module B - Connected devices |
| Medication reconciliation | Medium | Every patient | Module B - Smart med rec |
| Fax-based communication | High | Daily | Module H - Integration layer |
| Prior auth paperwork | High | Daily | Module D - PA automation |
| Refill request management | Medium | Daily | Module B - AI refill processing |
| Patient education material prep | Low | Per encounter | Module F - Auto education |

**Phone tag is the silent killer of clinical staff productivity.** An MA may spend 30-60 minutes per day calling patients about test results, appointment reminders, medication questions, and referral follow-ups. Each call has a 40-60% chance of reaching the patient. Failed calls must be retried. Messages must be documented.

### 7.3 Front Desk Staff

| Pain Point | Severity | Frequency | MedOS Module |
|------------|----------|-----------|-------------|
| Insurance verification (multiple portals) | High | Every patient | Module C - Auto eligibility |
| No-show management | High | Daily | Module F - Smart reminders |
| Scheduling complexity | Medium | All day | Module B - ML scheduling |
| Patient complaints about wait times | Medium | Daily | Module F - Wait time comms |
| Copay collection awkwardness | Medium | Every patient | Module C - Digital payments |
| Phone volume | High | All day | Module F - AI phone agent |
| Referral scheduling for patients | Medium | Daily | Module B - Referral coordination |

**Front desk staff are the most overloaded role in a practice.** They simultaneously manage the phone (which rings constantly), check in arriving patients, check out departing patients, schedule appointments, verify insurance, collect payments, and handle patient complaints. They are interrupted every 2-3 minutes on average. MedOS Module F (Patient Engagement) can dramatically reduce phone volume through self-scheduling, automated reminders, and an AI chat/phone agent for common questions.

### 7.4 Billing Staff

| Pain Point | Severity | Frequency | MedOS Module |
|------------|----------|-----------|-------------|
| Coding errors from providers | High | Daily | Module C - AI coding |
| Claim denials (15-20% rate) | Critical | Daily | Module C - Denial management |
| Denial appeal process | High | Daily | Module C - AI appeal agent |
| Patient collections | High | Daily | Module C - Patient billing |
| Payer contract complexity | Medium | Ongoing | Module D - Contract modeling |
| ERA posting and reconciliation | Medium | Daily | Module C - Auto posting |
| Aging A/R follow-up | High | Weekly | Module C - A/R management |

**Denial management is the most costly pain point.** 65% of denied claims are never resubmitted or appealed. Of those that are appealed, 50-70% are overturned. This means practices are leaving significant revenue on the table simply because they do not have the staff bandwidth to appeal denials. MedOS Module C can automate denial identification, root cause analysis, and appeal letter generation. See [[Revenue-Cycle-Deep-Dive]] for detailed analysis.

### 7.5 Practice Managers

| Pain Point | Severity | Frequency | MedOS Module |
|------------|----------|-----------|-------------|
| Revenue visibility (delayed reporting) | High | Ongoing | Module E - Real-time analytics |
| Staff recruitment and retention | Critical | Ongoing | All (reduce burden) |
| Compliance management | High | Ongoing | Module G - Compliance engine |
| Provider productivity tracking | Medium | Monthly | Module E - Provider dashboards |
| Payer contract negotiations | Medium | Annually | Module D - Contract intelligence |
| Quality measure reporting (MIPS) | Medium | Quarterly/Annual | Module E - Quality engine |
| Multi-location coordination | Medium | Daily (if applicable) | Module B - Multi-site ops |

**Staffing is the existential threat to practices.** The medical support staff shortage is as acute as the physician shortage. Medical assistants, front desk staff, and billing specialists are leaving healthcare for less stressful jobs in other industries. MedOS addresses this indirectly by reducing the work burden on every role, making existing staff more efficient and less burned out.

---

## 8. Where AI Can Help - MedOS Module Mapping

Every pain point identified above maps to a MedOS module. Here is the consolidated AI automation opportunity map:

### Highest Impact (Deploy First)

1. **Ambient AI Documentation** (Module B): Eliminates documentation burden, the #1 pain point for providers. Immediately measurable ROI. See [[Ambient-AI-Documentation]].
2. **AI Coding Engine** (Module C): Generates ICD-10 and CPT codes from AI-generated notes. Reduces coding errors, accelerates billing, increases revenue capture. See [[Revenue-Cycle-Deep-Dive]].
3. **Prior Auth Automation** (Module D): Eliminates phone calls, faxes, and portal submissions for PA. Identifies PA requirements at scheduling time. See [[Prior-Authorization-Deep-Dive]].

### High Impact (Deploy Second)

4. **Patient Communication Agent** (Module F): AI-powered phone/SMS/chat agent that handles appointment reminders, scheduling, prescription refill requests, and billing questions. Reduces phone volume by 40-60%.
5. **Automated Eligibility Verification** (Module C): Batch eligibility checks every evening for next-day patients. Eliminates manual portal lookups. Flags exceptions proactively.
6. **Denial Management Agent** (Module C): Identifies denied claims, determines root cause, drafts appeal letters, resubmits automatically where possible.

### Medium Impact (Deploy Third)

7. **Smart Scheduling** (Module B): ML-predicted no-shows, automated overbooking, self-scheduling with smart availability.
8. **AI Inbox Triage** (Module B): Prioritizes provider inbox (results, messages, refills) by urgency. Auto-drafts responses for routine items (normal lab results, medication refills).
9. **Referral Coordination** (Module B): Automated specialist finding, in-network verification, record transmission, and referral loop closure tracking.
10. **Quality Reporting** (Module E): Automated MIPS/HEDIS measure calculation and gap identification.

---

## 9. Clinical Decision Support (CDS)

### What is CDS

Clinical Decision Support encompasses any system that provides clinicians with knowledge and patient-specific information at the point of care to enhance decision making. This ranges from simple drug-drug interaction alerts to complex predictive models for disease risk.

### CDS Hooks

CDS Hooks is an HL7-standardized specification for triggering clinical decision support at specific points in the clinical workflow. It works like webhooks -- the EHR fires a "hook" (event) at defined trigger points, and CDS services respond with cards (recommendations, alerts, suggestions).

**Standard hooks relevant to MedOS:**
- `patient-view`: Triggered when a provider opens a patient's chart. Good for chronic disease reminders, preventive care gaps.
- `order-select`: Triggered when a provider selects an order (lab, medication, imaging). Good for formulary checking, duplicate order detection.
- `order-sign`: Triggered when a provider signs an order. Good for drug-drug interactions, dosing alerts, PA requirement notification.
- `encounter-start`: Triggered at the beginning of an encounter. Good for pre-visit summaries, outstanding results review.
- `encounter-discharge`: Triggered at the end of an encounter. Good for care gap reminders, follow-up scheduling prompts.

### SMART on FHIR Apps

SMART (Substitutable Medical Applications, Reusable Technologies) on FHIR is a standard for launching third-party applications within the EHR context. SMART apps receive the patient context (who is the patient, who is the provider, what encounter is active) via OAuth2 and can read/write FHIR resources with the user's authorization.

For MedOS, SMART on FHIR is relevant in two directions:
1. **MedOS as a SMART app**: Launching within an existing EHR (Epic, Cerner) for practices that are not ready to fully migrate. The ambient documentation widget launches as a SMART app within Epic.
2. **Third-party SMART apps on MedOS**: The MedOS platform exposes a SMART on FHIR launch framework, allowing third-party clinical apps to run within the MedOS workspace.

### The Alert Fatigue Problem

CDS alerts are one of the most well-studied failures in health IT. Studies show that providers override 49-96% of CDS alerts. The reasons are well documented:

- **Volume**: Providers receive too many alerts per day (average 50-100 in a busy practice)
- **Low specificity**: Many alerts are not clinically relevant (e.g., alerting about a drug interaction that is intentional and monitored)
- **Interruption**: Alerts are modal dialogs that interrupt the workflow, requiring a click to dismiss
- **Cry-wolf effect**: After dismissing hundreds of irrelevant alerts, providers stop reading them entirely -- including the rare critical alert

**MedOS approach to CDS**: Ambient, contextual CDS rather than interruptive alerts. Instead of pop-up warnings, surface relevant information within the workflow naturally. Use Claude to filter and prioritize alerts -- only surface alerts that are clinically significant given the full patient context. Aim for fewer than 5 high-value alerts per provider per day rather than 50 low-value interruptions. Track alert acceptance rates and continuously tune the relevance model.

---

## 10. Telehealth Integration

### How Virtual Visits Change the Workflow

Telehealth (video visits) modifies several stages of the clinical workflow:

**Eliminated steps:**
- Physical check-in (patient joins from home)
- Rooming (no vitals unless patient has home devices)
- Physical examination (limited to visual inspection)

**Modified steps:**
- **Scheduling**: Must distinguish between in-person and virtual visit types. Some encounters require in-person (procedures, hands-on exam) and should not be offered virtually.
- **Pre-visit**: Technology check replaces demographics verification. Patient needs a device with camera/microphone, internet connection, and the ability to join a video call.
- **The encounter itself**: Provider and patient interact via video. The conversation is similar to in-person, but physical exam is limited. Provider may ask patient to self-examine ("press on your abdomen here -- does that hurt?") or show affected areas to the camera.
- **Documentation**: Ambient AI documentation works identically for telehealth -- the system captures the audio from the video call.
- **Post-visit**: Prescriptions, lab orders, and referrals work the same. Follow-up scheduling must determine whether the next visit should be virtual or in-person.

### Technical Requirements

**Video platform**: Telehealth requires a HIPAA-compliant video platform. Options:
- **Twilio Video**: Programmable video API. HIPAA BAA available. Most flexible for custom integration into the MedOS workspace. Cost: $0.004/participant/minute.
- **Doxy.me / Zoom for Healthcare**: Purpose-built telehealth platforms. Less customizable but faster to deploy.

**MedOS approach**: Integrate Twilio Video directly into the provider workspace. The video window and ambient AI recording run simultaneously. The provider sees the patient, talks naturally, and the AI generates the note -- identical workflow to in-person except the patient is on a screen.

**Audio capture for ambient AI**: For telehealth, audio capture is simpler than in-person because the conversation already flows through the application. The system captures the audio stream directly from the WebRTC connection -- no microphone placement concerns, no ambient noise from the clinic, and speaker diarization is trivial (provider audio and patient audio are on separate channels).

**Patient-reported vitals**: For telehealth encounters where vitals are needed, patients can use consumer health devices (Bluetooth blood pressure cuffs, pulse oximeters, smart scales) that sync data to Apple Health or Google Health Connect. MedOS Module F can ingest this data and pre-populate vitals in the encounter record. This is especially valuable for chronic disease management visits (hypertension, diabetes) where remote monitoring between visits provides better data than a single in-office reading.

**Interstate licensing**: Providers can only practice medicine in states where they hold a license. Telehealth encounters cross state lines when the patient is in a different state than the provider. The Interstate Medical Licensure Compact (IMLC) simplifies multi-state licensing for physicians (40 member states as of 2026). MedOS must verify that the provider is licensed in the patient's state at scheduling time.

**Prescribing limitations**: Some controlled substances have restrictions on telehealth prescribing. The Ryan Haight Act requires an in-person evaluation before prescribing Schedule II-V controlled substances via telemedicine, with certain exceptions that were expanded during COVID and partially extended post-pandemic. MedOS should flag when a provider attempts to prescribe a controlled substance in a telehealth encounter without a prior in-person visit on record.

---

## Summary

Clinical workflows in U.S. outpatient practice are complex, multi-step processes involving numerous staff roles, system handoffs, and regulatory requirements. Every stage -- from scheduling through final billing -- contains pain points that cause inefficiency, revenue leakage, staff burnout, and patient dissatisfaction.

MedOS addresses these pain points through eight integrated modules, with Module B (Provider Workflow Engine) and Module F (Patient Engagement) serving as the primary clinical workflow automation layers. The highest-impact interventions are [[Ambient-AI-Documentation]] (provider documentation burden), [[Prior-Authorization-Deep-Dive]] automation (administrative burden on all staff), and AI-powered coding and denial management (revenue cycle efficiency per [[Revenue-Cycle-Deep-Dive]]).

The key insight for MedOS is that clinical workflows are interconnected. Ambient documentation feeds coding, coding feeds billing, billing reveals denial patterns, denial patterns inform prior auth strategy. A platform that automates the entire workflow chain will always outperform point solutions that address individual pain points in isolation. This is the core thesis of the Healthcare OS architecture described in [[HEALTHCARE_OS_MASTERPLAN]].
