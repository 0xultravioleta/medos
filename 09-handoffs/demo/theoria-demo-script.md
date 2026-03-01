---
type: demo-script
date: "2026-03-01"
status: active
tags:
  - demo
  - theoria
  - pitch
  - di-rezze
---

# Theoria Medical Demo Script: MedOS Live Platform Walkthrough

> Stage directions for the Dr. Di Rezze pitch meeting.
> Follow this script act by act. Each section includes what to click, what to say, and where to navigate.
> Total runtime: ~14 minutes (adjustable).

**Platform URL:** https://medos-platform.vercel.app
**Login:** justin@medos.ai / demo123
**Related:** [[theoria-medical-pitch-strategy]] | [[EPIC-016-theoria-medical-pilot]] | [[HEALTHCARE_OS_MASTERPLAN]]

---

## Pre-Demo Checklist

Complete every item before starting the demo. Do not proceed if any item fails.

- [ ] Vercel deployment is live and working at https://medos-platform.vercel.app
- [ ] All 53 routes load without errors
- [ ] Login credentials work (justin@medos.ai / demo123)
- [ ] Theoria sidebar appears after login
- [ ] All 13 Theoria pages render correctly:
  - [ ] `/theoria/discharge` — Discharge Reconciliation
  - [ ] `/theoria/guardian` — Post-Acute Guardian
  - [ ] `/theoria/readmission` — Readmission Risk
  - [ ] `/theoria/ccm` — CCM Time Tracker
  - [ ] `/theoria/rpm` — RPM Revenue Dashboard
  - [ ] `/theoria/care-gaps` — Care Gap Scanner
  - [ ] `/theoria/aco-reach` — ACO REACH Performance
  - [ ] `/theoria/facility` — Facility Console
  - [ ] `/theoria/shift-handoff` — Shift Handoff
  - [ ] `/theoria/executive` — PE Executive Dashboard
  - [ ] `/theoria/credentialing` — Credentialing
  - [ ] `/theoria/staffing` — Dynamic Staffing
  - [ ] `/theoria/care-plan` — Care Plan Optimizer
- [ ] Mock data shows Theoria-specific details (facility names, Michigan/Florida)
- [ ] ChartEasy/ChatEasy references visible in discharge reconciliation page
- [ ] Executive dashboard shows Amulet Capital metrics
- [ ] ACO REACH shows Empassion Health benchmarks
- [ ] Browser zoom at 100%
- [ ] Chrome in fullscreen mode (F11)
- [ ] Resolution set to 1920x1080
- [ ] Screen recording software ready (OBS or Playwright E2E)
- [ ] Pitch strategy doc accessible offline ([[theoria-medical-pitch-strategy]])
- [ ] Backup: local dev server running (`cd Z:/medos-platform/frontend && npm run dev`)

---

## Technical Setup Notes

- **Browser:** Chrome, fullscreen mode (F11)
- **Resolution:** 1920x1080 for recording and screen share
- **Mouse movement:** Slow and deliberate — hover on key numbers for 2-3 seconds before moving
- **Page pacing:** Each page gets 20-30 seconds of exploration before moving on
- **Tab management:** Only one tab open. No tab switching during demo.
- **Reference:** Have [[theoria-medical-pitch-strategy]] open on a second monitor or printed
- **Fallback:** If Vercel is down, switch to `localhost:3000` immediately (local dev server must be running)

---

## ACT 1: "The False Sense of Security" (3 minutes)

> **Goal:** Hook Dr. Di Rezze with HIS founding story. Show that MedOS solves the exact problem that drove him to create Theoria.

### Step 1: Open MedOS Platform

1. Open Chrome. Navigate to **https://medos-platform.vercel.app**
2. The login screen appears. Pause for 1 second — let the medical-grade UI register.
3. Enter credentials:
   - Email: `justin@medos.ai`
   - Password: `demo123`
4. Click **Sign In**.

> **SAY:** "Dr. Di Rezze, let me show you something. This is MedOS — the operating system we built for healthcare."

### Step 2: Navigate to Discharge Reconciliation

5. In the left sidebar, find the **Theoria** section.
6. Click **Discharge Reconciliation** (`/theoria/discharge`).
7. Wait for the page to fully load. The Hospital-to-SNF transition queue appears.

> **SAY:** "You founded Theoria because you saw something no one else was acting on — that 'false sense of security' when a patient gets discharged from the hospital to a SNF. The handoff looked clean on paper, but critical information was falling through the cracks."

### Step 3: Show the Transition Queue

8. Point to the queue. Highlight the hospital names: **Ascension Genesys**, **Beaumont**.
9. Slowly scroll through the list. Each row shows: patient, origin hospital, destination SNF, discrepancy count, status.

> **SAY:** "Here's your transition queue. Every patient moving from hospital to SNF. These are your facilities — Ascension Genesys, Beaumont. MedOS is watching every discharge in real time."

### Step 4: Open a Discrepancy Report

10. Click on the first patient row with a red discrepancy indicator.
11. The discrepancy detail panel opens. Pause for 3 seconds.
12. Point to the **New Medications** section — highlight items flagged in red.

> **SAY:** "This patient was discharged with three new medications that aren't in their SNF chart. One is a blood thinner. Without this system, that discrepancy might not be caught until rounds the next morning — or later."

### Step 5: Show ChartEasy Comparison

13. Click the **Med Changes** tab.
14. The split-view comparison appears: Hospital discharge meds vs ChartEasy record.
15. Hover over each flagged discrepancy. Let the color coding speak.

> **SAY:** "This is the ChartEasy comparison. Left side: what the hospital prescribed. Right side: what's in your ChartEasy record. Red means missing. Yellow means dosage changed. MedOS catches every discrepancy automatically — no human has to comb through PDFs."

### Bridge to ACT 2

> **SAY:** "This is the problem you founded Theoria to solve. MedOS catches every discrepancy before the patient even arrives. But what about after they're back in your SNF? What happens when something changes at 2 AM?"

---

## ACT 2: "The Guardian" (3 minutes)

> **Goal:** Show predictive monitoring. Prove that MedOS turns reactive care into proactive intervention.

### Step 1: Navigate to Post-Acute Guardian

1. Click **Post-Acute Guardian** in the sidebar (`/theoria/guardian`).
2. The live monitoring dashboard loads. Wearable data streams are visible.

> **SAY:** "This is the Guardian. It monitors your patients around the clock using data from wearables — Oura Rings, Apple Watches, Dexcom sensors."

### Step 2: Show Live Dashboard

3. Point to the device status panel. Show the connected device counts.
4. Point to the vital signs grid — heart rate, SpO2, blood glucose, weight trends.
5. Hover over a patient card showing trending vitals.

> **SAY:** "Real-time data from real devices. Not hypothetical — we built the FHIR ingestion pipeline for these exact sensors."

### Step 3: Open the Alert Queue

6. Click the **Alert Queue** tab or section.
7. Find the CHF patient alert: **+3.2 lbs weight gain overnight**.
8. Click into the alert detail. Pause for 3 seconds.

> **SAY:** "Here's your CHF patient at one of the Michigan SNFs. Gained 3.2 pounds overnight. That's a textbook early sign of fluid retention — potential CHF exacerbation."

### Step 4: Show AI Confidence and Recommendation

9. Point to the **AI Confidence Score** (displayed prominently).
10. Point to the **Recommended Action** section.

> **SAY:** "Our AI cross-references the weight gain against their medication list — they're on Furosemide — their recent labs, and their historical trend. Confidence score: 87%. Recommended action: increase Furosemide dose and schedule telemedicine check-in within 4 hours. Your doctor sees this before rounds."

### Step 5: Navigate to Readmission Risk

11. Click **Readmission Risk** in the sidebar (`/theoria/readmission`).
12. Show the risk scoring model — LACE+ scores across the patient population.

> **SAY:** "Your ChartEasy stores the data. MedOS makes it predictive. Every patient has a real-time readmission risk score. Red means intervene now."

### Bridge to ACT 3

> **SAY:** "Every prevented readmission is $15,000 to $25,000 saved for your ACO. But MedOS doesn't just save money — it captures revenue you're currently leaving on the table."

---

## ACT 3: "The Revenue Engine" (3 minutes)

> **Goal:** Show the money. CCM, RPM, and care gap revenue that Theoria is currently missing.

### Step 1: Navigate to CCM Time Tracker

1. Click **CCM Time Tracker** in the sidebar (`/theoria/ccm`).
2. The time tracking dashboard loads.

> **SAY:** "Chronic Care Management — CPT 99490. Every eligible patient is worth $60 to $150 per month. But you have to document at least 20 minutes of non-face-to-face time. Most groups miss this."

### Step 2: Show the 20-Minute Threshold

3. Point to the **time threshold visualization**. Show patients approaching and exceeding the 20-minute mark.
4. Hover over a patient close to threshold — show the time breakdown.

> **SAY:** "MedOS automatically tracks every qualifying interaction — chart reviews, messages on ChatEasy, care coordination calls. When your doctor hits 20 minutes, the claim is ready."

### Step 3: Switch to Billing Tab

5. Click the **Billing** tab.
6. Show CPT 99490 revenue capture numbers — monthly totals, eligible vs captured.

> **SAY:** "Here's the revenue. Eligible patients on the left, captured billing on the right. That gap is money you're owed but not collecting."

### Step 4: Navigate to RPM Revenue

7. Click **RPM Revenue** in the sidebar (`/theoria/rpm`).
8. Show the device billing matrix: CPT 99453, 99454, 99457, 99458.

> **SAY:** "Remote Patient Monitoring is another revenue stream. Device setup, monthly monitoring, clinical time — each has its own CPT code. MedOS tracks all of them."

### Step 5: Navigate to Care Gap Scanner

9. Click **Care Gaps** in the sidebar (`/theoria/care-gaps`).
10. Show the care gap scanner — open gaps by measure, by facility.

> **SAY:** "$60 to $150 per patient per month in CCM alone. For your 2,400 attributed lives in ACO REACH, that's $1.7 to $4.3 million per year in recovered revenue. And every closed care gap improves your quality score."

### Bridge to ACT 4

> **SAY:** "And all of this feeds directly into your ACO REACH performance. Let me show you."

---

## ACT 4: "The Platform" (3 minutes)

> **Goal:** Show enterprise scale. Prove MedOS handles 21 states, 87 facilities, 142 providers.

### Step 1: Navigate to ACO REACH Performance

1. Click **ACO REACH** in the sidebar (`/theoria/aco-reach`).
2. The performance dashboard loads with quality measures.

> **SAY:** "This is your ACO REACH command center. Every quality measure, every benchmark, tracked in real time."

### Step 2: Show Quality Measures

3. Point to quality measures vs targets. Show green (met), yellow (at risk), red (below).
4. Point to the **Empassion Health benchmarks** section.

> **SAY:** "You're in ACO REACH through Empassion Health — the largest participant in the country. These are your quality measures against their benchmarks. Green means you're exceeding. Red means there's shared savings at risk."

### Step 3: Show Shared Savings Multiplier

5. Point to the **shared savings calculation** section.
6. Hover over the multiplier — show how quality improvement directly increases revenue.

> **SAY:** "Here's the math that matters: every percentage point improvement in quality multiplies your shared savings. MedOS makes this visible and actionable."

### Step 4: Navigate to Facility Console

7. Click **Facility Console** in the sidebar (`/theoria/facility`).
8. The multi-site overview loads. Show 4+ SNFs across Michigan and Florida.

> **SAY:** "One platform for 21 states, 87 facilities, 142 providers. This is your network at a glance."

### Step 5: Show Staffing Tab

9. Click the **Staffing** tab.
10. Show provider allocation across facilities — who is covering where, when.

> **SAY:** "Provider allocation in real time. Who's covering which facility, what shift, what's their current patient load."

### Step 6: Navigate to Shift Handoff

11. Click **Shift Handoff** in the sidebar (`/theoria/shift-handoff`).
12. Show the handoff briefing — priority-ranked patient list.

> **SAY:** "When your night doctor logs off and the morning doctor logs on, they get this: a priority-ranked briefing. 'Patient Jones at Sunrise SNF — potassium 5.8, trending up, recheck at 6 AM.' No critical results missed. Ever."

### Bridge to ACT 5

> **SAY:** "One platform for everything. But the real question isn't whether this works — it's what it means for your valuation."

---

## ACT 5: "The Exit Math" (2 minutes)

> **Goal:** Hit the financial argument. Speak in PE language. Close with the ask.

### Step 1: Navigate to PE Executive Dashboard

1. Click **Executive Dashboard** in the sidebar (`/theoria/executive`).
2. The board-level KPI view loads: Revenue, EBITDA, Facilities, Providers.

> **SAY:** "This is what Amulet sees. Board-level KPIs in real time. Revenue, EBITDA, facility count, provider count — all from a single dashboard."

### Step 2: Show Financial Tab

3. Click the **Financial** tab.
4. Point to the **valuation bridge** visualization.
5. Hover over the two scenarios: without MedOS vs with MedOS.

> **SAY:** "Here's the math. Right now, Theoria is valued as a physician services company — 8 to 12x EBITDA. That's the market reality."

### Step 3: Deliver the Kill Line

6. Point to the "with MedOS" scenario number.
7. Pause. Make eye contact (if in person) or pause for 3 seconds (if virtual).

> **SAY (THE KILL LINE):** "At a physician services multiple: $81 million. With MedOS as your platform: $178 million. That's $97 million in created value. The difference between being a staffing company and being a technology platform."

### Step 4: Navigate to Credentialing

8. Click **Credentialing** in the sidebar (`/theoria/credentialing`).
9. Show the 21-state licensing management view.

> **SAY:** "And the operational complexity? 21-state credentialing, license tracking, renewal management — all automated. Every new state you enter, MedOS handles the licensing maze."

### Step 5: The Close

10. Return to the Executive Dashboard. Let the numbers sit on screen.

> **SAY (THE CLOSE):** "This was built in 15 days by 2 people. 53 pages, 522 tests, 44 AI tools, 5 agents. Imagine what 90 days looks like with your team and your data. Give us one Michigan facility. Read-only. Zero disruption. If we don't find at least $500K in uncaptured annual revenue across your network, we walk away."

11. Pause. Let the silence work. Do not speak first after the close.

---

## Post-Demo Actions

| Timing | Action |
|--------|--------|
| During demo | Note every question Dr. Di Rezze asks — these reveal priorities |
| End of demo | Ask: "Which of these resonated most?" — his answer determines pilot scope |
| Same day | Thank-you email + 1-page platform summary PDF |
| Day 2 | Send personalized video walkthrough of the feature he was most interested in |
| Day 5 | Share VBC Opportunity Model with conservative revenue estimates |
| Day 10 | Propose 90-day pilot scope and timeline |
| Day 14 | Follow up for decision on pilot |

---

## Screen Recording Instructions

If a pre-recorded backup of the demo is needed (for connectivity issues or leave-behind):

### Option 1: Automated E2E Recording (Recommended)

```bash
cd Z:/medos-platform/frontend
npm run demo:headless
```

- This runs the full Playwright E2E test suite across all 53 pages including all 13 Theoria pages
- Output: MP4 video via Playwright + ffmpeg
- Duration: ~8-12 minutes for the full walkthrough
- Resolution: 1920x1080, 30fps

### Option 2: Manual Recording with OBS Studio

1. Open OBS Studio
2. Set output resolution: 1920x1080
3. Set framerate: 30fps
4. Set output format: MP4 (H.264)
5. Set audio: Desktop audio + microphone (for narration)
6. Start recording before opening the browser
7. Follow the ACT 1-5 script above at a measured pace
8. Stop recording after the close
9. Export and trim dead air from beginning/end

### Option 3: Playwright Custom Script

```bash
cd Z:/medos-platform/frontend
npx playwright test --project=chromium --headed
```

- Use `--headed` to see the browser during recording
- Combine with OBS for narration overlay

### Recording Best Practices

- Record the demo at least twice — keep the better take
- Target 8-12 minutes for the full demo
- Add 2 seconds of pause between each page navigation (helps with editing)
- Ensure mouse cursor is visible at all times
- Test playback before sending — verify audio sync and resolution

---

## Route Reference (All 13 Theoria Pages)

| ACT | Route | Page Name | Time Allocation |
|-----|-------|-----------|-----------------|
| 1 | `/theoria/discharge` | Discharge Reconciliation | 90 sec |
| 2 | `/theoria/guardian` | Post-Acute Guardian | 60 sec |
| 2 | `/theoria/readmission` | Readmission Risk | 30 sec |
| 3 | `/theoria/ccm` | CCM Time Tracker | 45 sec |
| 3 | `/theoria/rpm` | RPM Revenue Dashboard | 30 sec |
| 3 | `/theoria/care-gaps` | Care Gap Scanner | 30 sec |
| 4 | `/theoria/aco-reach` | ACO REACH Performance | 45 sec |
| 4 | `/theoria/facility` | Facility Console | 45 sec |
| 4 | `/theoria/shift-handoff` | Shift Handoff | 30 sec |
| 5 | `/theoria/executive` | PE Executive Dashboard | 45 sec |
| 5 | `/theoria/credentialing` | Credentialing | 20 sec |
| — | `/theoria/staffing` | Dynamic Staffing | (referenced in ACT 4) |
| — | `/theoria/care-plan` | Care Plan Optimizer | (referenced if asked) |

**Total estimated demo time:** ~14 minutes (can compress to 10 by reducing exploration time per page)

---

*This demo script was prepared for the Dr. Justin Di Rezze pitch meeting. Follow the ACTs sequentially. Do not skip ACT 1 — the founding story hook is the most important emotional anchor in the entire presentation.*
