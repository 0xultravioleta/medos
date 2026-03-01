---
type: operations
date: "2026-02-28"
tags:
  - operations
  - pilot
  - communication
  - launch
  - phase-1
---

# Pilot Communication Plan

## Purpose

Structured communication framework for MedOS pilot practices. Clear channels, defined cadences, and explicit escalation paths ensure no issue goes unaddressed and no stakeholder is left in the dark.

**References:** [[monitoring-rotation-runbook]], [[Incident-Response-Playbook]]

---

## Communication Channels

| Channel | Purpose | Audience | Response SLA |
|---------|---------|----------|-------------|
| **Slack: #medos-pilot-support** | Primary support channel. Bug reports, questions, real-time troubleshooting. | MedOS team + pilot practice contacts | P1: 15 min, P2: 1 hr, P3: 4 hr |
| **Slack: #medos-pilot-feedback** | Feature requests, workflow suggestions, general feedback. Non-urgent. | MedOS team + pilot practice contacts | 1 business day acknowledgment |
| **Email: support@medos.health** | Formal communications, documentation, non-Slack users. | All stakeholders | 4 business hours |
| **Phone: Emergency Hotline** | P1 issues only. System down, PHI exposure, patient safety concern. | Pilot practice admin -> MedOS on-call | Immediate pickup or callback within 15 min |
| **GitHub Issues** | Technical bug tracking with full reproduction details. | MedOS engineering team | Triage within 1 business day |

**Channel rules:**
- PHI must NEVER appear in Slack or GitHub Issues. Use patient IDs or encounter IDs only.
- All P1 issues start with phone call, then documented in Slack and GitHub.
- Feature requests go to `#medos-pilot-feedback`, not `#medos-pilot-support`.

---

## Weekly Check-in Agenda Template

**Cadence:** Every Friday, 2:00 PM ET (30 minutes)
**Attendees:** MedOS team lead + pilot practice champion (office manager or lead provider)

```markdown
# MedOS Pilot Weekly Check-in - YYYY-MM-DD

## 1. System Health Summary (5 min)
- Uptime this week: XX.X%
- Error rate trend: X% -> X%
- Key incidents (if any):

## 2. Usage Metrics Review (5 min)
- SOAP notes generated: X
- Claims submitted: X
- Active users: X / Y total
- Most-used features:
- Least-used features:

## 3. Open Issues Triage (10 min)
| # | Issue | Severity | Status | ETA |
|---|-------|----------|--------|-----|
| | | | | |

## 4. Feature Requests Prioritization (5 min)
| Request | Votes | Effort | Priority |
|---------|-------|--------|----------|
| | | | |

## 5. Next Week Focus (5 min)
- Planned releases:
- Scheduled maintenance:
- Training sessions:

## Action Items
- [ ] [Owner] Action item
```

---

## Bug Reporting Process

### GitHub Issues Template

When reporting bugs, use this required fields template:

```markdown
**Title:** [Brief description of the bug]

**Severity:** P1 (Critical) / P2 (High) / P3 (Medium) / P4 (Low)

**Reporter:** [Name, role, practice]

**Date/Time:** YYYY-MM-DD HH:MM ET

**Environment:** Production / Demo

**Steps to Reproduce:**
1. Step 1
2. Step 2
3. Step 3

**Expected Behavior:**
[What should happen]

**Actual Behavior:**
[What actually happened]

**Screenshots/Recordings:**
[Attach if available -- ensure NO PHI visible]

**Browser/Device:**
[e.g., Chrome 125, Windows 11]

**Impact:**
[How many users affected? Workaround available?]

**IMPORTANT: Do NOT include any PHI (patient names, DOB, SSN, MRN) in bug reports.
Use encounter IDs or anonymized identifiers only.**
```

### Bug Lifecycle

```
Reported -> Triaged (1 business day) -> Assigned -> In Progress -> Fix Deployed -> Verified -> Closed
```

| Severity | Triage SLA | Fix SLA | Verification |
|----------|-----------|---------|-------------|
| P1 | Immediate | 4 hours | Same day |
| P2 | 4 hours | 2 business days | Next deploy |
| P3 | 1 business day | Next sprint | Next deploy |
| P4 | 1 week | Backlog | Next deploy |

---

## Feature Request Process

```
Discussion -> Triage -> Roadmap -> Sprint Planning -> Implementation -> Release
```

1. **Discussion:** User submits in `#medos-pilot-feedback` or weekly check-in
2. **Triage:** Team reviews weekly, assesses effort (S/M/L/XL) and impact (1-5)
3. **Roadmap:** High-impact/low-effort items added to roadmap. Communicate timeline to requester.
4. **Sprint Planning:** Roadmap items pulled into sprint based on priority and capacity
5. **Implementation:** Standard development process with tests
6. **Release:** Deployed, requester notified, feedback collected

**Prioritization matrix:**

| | Low Effort | Medium Effort | High Effort |
|---|-----------|--------------|-------------|
| **High Impact** | Do now | Next sprint | Plan carefully |
| **Medium Impact** | Next sprint | Backlog | Evaluate ROI |
| **Low Impact** | If time permits | Backlog | Decline or defer |

---

## Escalation Matrix

| Severity | Description | Response Time | First Contact | Escalation | Resolution SLA |
|----------|-----------|---------------|-------------|------------|----------------|
| **P1 - Critical** | System down, PHI exposure, patient safety risk | 15 minutes | On-call engineer (phone) | Both engineers + Dr. Di Reze | 4 hours |
| **P2 - High** | Feature broken, data incorrect, workflow blocked | 1 hour | `#medos-pilot-support` | Primary on-call engineer | 2 business days |
| **P3 - Medium** | Minor bug, UI issue, non-blocking problem | 4 hours | `#medos-pilot-support` | Triage in daily check | Next sprint |
| **P4 - Low** | Enhancement, cosmetic, nice-to-have | 1 business day | `#medos-pilot-feedback` | Weekly review | Backlog |

**Escalation rules:**
- If P2 not resolved in 4 hours, escalate to P1 procedures
- If practice reports same issue 3+ times, auto-escalate one severity level
- All PHI-related issues are automatically P1 regardless of functional impact

---

## Success Criteria Reviews

### Review Cadence

| Period | Cadence | Format | Attendees |
|--------|---------|--------|-----------|
| Month 1 (Weeks 1-4) | Weekly | 30-min video call | MedOS team + practice champion |
| Months 2-3 (Weeks 5-12) | Biweekly | 30-min video call | MedOS team + practice champion |
| Month 4+ | Monthly | 30-min video call + written report | MedOS team + practice champion + Dr. Di Reze |

### Success Metrics Tracked

| Category | Metric | Target | Measurement |
|----------|--------|--------|-------------|
| **Efficiency** | Time saved per provider per day | > 30 min | Provider self-report + system timestamps |
| **Quality** | AI coding accuracy (ICD-10/CPT) | > 92% | Sample audit 20 notes/week |
| **Revenue** | Claim first-pass acceptance rate | > 95% | Claims dashboard |
| **Revenue** | Denial rate | < 5% | Claims dashboard |
| **Adoption** | Daily active users | > 80% of staff | Login analytics |
| **Adoption** | Notes generated via AI | > 70% of encounters | System metrics |
| **Satisfaction** | Provider NPS | > 50 | Monthly survey |
| **Reliability** | System uptime | > 99.5% | Monitoring dashboard |

### Review Report Template

```markdown
# Pilot Success Review - Week X / Month X

## Executive Summary
[2-3 sentences on overall pilot health]

## Key Metrics
| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Time saved/provider/day | > 30 min | | |
| AI coding accuracy | > 92% | | |
| First-pass acceptance | > 95% | | |
| Daily active users | > 80% | | |
| System uptime | > 99.5% | | |

## Wins
- [What's working well]

## Challenges
- [What needs attention]

## Action Items
- [ ] [Owner] Action
```

---

## Stakeholder Updates

### Dr. Di Reze -- Bi-Weekly Email

**Cadence:** Every other Monday, 9:00 AM ET
**Format:** Email with metrics summary + link to dashboard

```markdown
Subject: MedOS Pilot Update - Week X-Y

Dr. Di Reze,

**Pilot Status:** ON TRACK / NEEDS ATTENTION / AT RISK

**Key Numbers:**
- Active providers: X / Y
- SOAP notes generated this period: X
- AI coding accuracy: X%
- Claims submitted: X (first-pass rate: X%)
- System uptime: XX.X%
- Provider satisfaction (last survey): X/10

**Highlights:**
- [1-2 positive developments]

**Concerns:**
- [0-2 items needing attention, with mitigation plans]

**Next Steps:**
- [1-2 upcoming milestones]

Full dashboard: [link]

Best regards,
MedOS Team
```

---

## Communication Calendar (First Month)

| Week | Monday | Wednesday | Friday |
|------|--------|-----------|--------|
| Week 1 (Launch) | Launch day all-hands | Mid-week check-in (ad hoc) | Weekly check-in #1 |
| Week 2 | Dr. Di Reze update #1 | -- | Weekly check-in #2 |
| Week 3 | -- | -- | Weekly check-in #3 |
| Week 4 | Dr. Di Reze update #2 | -- | Weekly check-in #4 + Month 1 review |
