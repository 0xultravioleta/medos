---
type: compliance
date: "2026-02-28"
tags:
  - security
  - incident-response
  - hipaa
  - compliance
  - phase-1
status: draft
---

# Incident Response Playbook -- MedOS Healthcare OS

> This playbook defines the incident classification, response procedures, communication protocols, and post-incident review process for MedOS. As a HIPAA Business Associate, MedOS must comply with the Breach Notification Rule (45 CFR 164.400-414). Cross-reference with [[HIPAA-Deep-Dive]], [[HIPAA-Risk-Assessment]], and [[Auth-SMART-on-FHIR]].

---

## 1. Incident Classification

### 1.1 Severity Levels

| Priority | Name | Definition | Examples |
|----------|------|-----------|----------|
| P1 | Critical | Confirmed PHI data breach, system-wide outage, active security intrusion | Unauthorized PHI export, ransomware, database compromise, full platform down |
| P2 | High | Suspected data breach, single-service outage, anomalous authentication activity | Unusual bulk PHI access, API service down, failed auth spike (>100 in 5 min), suspected insider threat |
| P3 | Medium | Performance degradation, non-PHI data issue, failed backup, configuration error | Slow API response (>5s p95), backup job failure, misconfigured CORS, non-PHI logging error |
| P4 | Low | Minor UI bug, non-critical feature unavailable, cosmetic issue | Formatting error in reports, non-critical dashboard widget down, minor UX inconsistency |

### 1.2 Classification Decision Tree

```
Is PHI potentially exposed or accessed without authorization?
  ├─ YES → Is it confirmed (evidence of access/exfiltration)?
  │         ├─ YES → P1 (Critical)
  │         └─ NO (suspected only) → P2 (High)
  └─ NO → Is a core service (API, database, auth) unavailable?
           ├─ YES → Is it system-wide?
           │         ├─ YES → P1 (Critical)
           │         └─ NO (single service) → P2 (High)
           └─ NO → Is it affecting user workflows?
                    ├─ YES → P3 (Medium)
                    └─ NO → P4 (Low)
```

---

## 2. Response Procedures

### 2.1 P1 -- Critical Incident

**Target: Containment in 15 minutes, investigation initiated in 1 hour.**

| Step | Action | Responsible | Timeframe |
|------|--------|-------------|-----------|
| 1 | **Detect and Alert:** CloudWatch alarm or manual report triggers PagerDuty alert | Monitoring System | Automated |
| 2 | **Acknowledge:** On-call engineer acknowledges alert, joins incident channel | On-call Engineer | 5 min |
| 3 | **Contain:** Isolate affected system (revoke compromised credentials, block suspicious IPs, disable affected service) | On-call Engineer | 15 min |
| 4 | **Assess PHI Impact:** Query FHIR AuditEvent logs to determine scope of PHI exposure (which patients, which data elements, which tenant) | Security Lead | 1 hour |
| 5 | **Escalate:** Notify CTO, Privacy Officer, and Legal | On-call Engineer | 30 min |
| 6 | **Investigate:** Root cause analysis using logs, audit trail, and infrastructure telemetry | Engineering Team | 4 hours |
| 7 | **Remediate:** Deploy fix, restore service, verify containment | Engineering Team | Based on RCA |
| 8 | **HIPAA Breach Assessment:** Apply 4-factor risk assessment (see Section 5) | Privacy Officer + Legal | 24 hours |
| 9 | **Document:** Complete incident report with timeline, RCA, and remediation steps | Incident Commander | 48 hours |

### 2.2 P2 -- High Incident

**Target: Acknowledge in 30 minutes, investigation in 2 hours.**

| Step | Action | Responsible | Timeframe |
|------|--------|-------------|-----------|
| 1 | **Detect and Alert:** CloudWatch alarm or anomaly detection triggers notification | Monitoring System | Automated |
| 2 | **Acknowledge:** On-call engineer acknowledges and assesses severity | On-call Engineer | 30 min |
| 3 | **Investigate:** Review audit logs, check for PHI exposure indicators | On-call Engineer | 2 hours |
| 4 | **Escalate if needed:** If PHI exposure confirmed, escalate to P1 | On-call Engineer | As needed |
| 5 | **Remediate:** Fix root cause, restore service | Engineering Team | 4 hours |
| 6 | **Document:** Incident summary with findings and actions taken | On-call Engineer | 24 hours |

### 2.3 P3 -- Medium Incident

**Target: Acknowledge in 2 hours, schedule fix.**

| Step | Action | Responsible | Timeframe |
|------|--------|-------------|-----------|
| 1 | **Detect:** Monitoring alert or user report | Monitoring / Support | Automated |
| 2 | **Acknowledge:** Engineer acknowledges and logs ticket | On-call Engineer | 2 hours |
| 3 | **Assess:** Determine if degradation could escalate | On-call Engineer | 4 hours |
| 4 | **Fix:** Schedule remediation in current or next sprint | Engineering Team | 1-5 days |

### 2.4 P4 -- Low Incident

**Target: Log and prioritize.**

| Step | Action | Responsible | Timeframe |
|------|--------|-------------|-----------|
| 1 | **Log:** Create ticket in backlog with details | Support / Engineer | 24 hours |
| 2 | **Prioritize:** Include in next sprint planning | Engineering Lead | Next sprint |

---

## 3. Communication Templates

### 3.1 Internal Notification (P1/P2)

```
SUBJECT: [P1/P2] MedOS Security Incident - [Brief Description]

SEVERITY: [P1 Critical / P2 High]
DETECTED: [Timestamp UTC]
AFFECTED SYSTEM: [Service/component]
AFFECTED TENANTS: [Tenant IDs or "All"]
PHI EXPOSURE: [Confirmed / Suspected / None]

CURRENT STATUS: [Investigating / Contained / Remediated]

SUMMARY:
[2-3 sentence description of what happened]

ACTIONS TAKEN:
- [List of containment/investigation steps completed]

NEXT STEPS:
- [Planned actions with owners and ETAs]

INCIDENT COMMANDER: [Name]
INCIDENT CHANNEL: [Slack channel / bridge link]
```

### 3.2 External Notification -- Patient (if breach confirmed)

```
SUBJECT: Important Notice About Your Health Information

Dear [Patient Name],

We are writing to inform you of an incident that may have involved
some of your health information maintained by [Practice Name] through
the MedOS platform.

WHAT HAPPENED:
[Clear, non-technical description of the incident]

WHAT INFORMATION WAS INVOLVED:
[Specific types of information, e.g., name, date of birth, medical
record number -- list only what was actually exposed]

WHAT WE ARE DOING:
[Steps taken to contain and prevent recurrence]

WHAT YOU CAN DO:
- Monitor your health insurance statements for unfamiliar activity
- Review your credit reports at annualcreditreport.com
- Contact us at [phone/email] with questions or concerns

FOR MORE INFORMATION:
Contact our Privacy Officer at [email] or [phone].
You may also file a complaint with the HHS Office for Civil Rights
at https://www.hhs.gov/hipaa/filing-a-complaint/

Sincerely,
[Name, Title]
[Practice Name]
```

### 3.3 Regulatory Notification (HHS)

Submitted via the HHS Breach Portal (https://ocrportal.hhs.gov/ocr/breach/wizard_breach.jsf):

- Name of covered entity / business associate
- Contact information for entity and for individuals
- Date of breach and date of discovery
- Type of PHI involved
- Number of individuals affected
- Brief description of breach and investigation
- Actions taken in response

---

## 4. Contact List

| Role | Name | Phone | Email | Escalation |
|------|------|-------|-------|------------|
| CTO / Incident Commander | [PLACEHOLDER] | [PLACEHOLDER] | [PLACEHOLDER] | Primary for all P1 |
| Security Lead | [PLACEHOLDER] | [PLACEHOLDER] | [PLACEHOLDER] | Primary for P1/P2 security |
| Privacy Officer | [PLACEHOLDER] | [PLACEHOLDER] | [PLACEHOLDER] | PHI breach assessment |
| Legal Counsel | [PLACEHOLDER] | [PLACEHOLDER] | [PLACEHOLDER] | Breach notification, regulatory |
| On-call Engineer | PagerDuty rotation | Via PagerDuty | Via PagerDuty | First responder |
| AWS Support | N/A | N/A | AWS Support Console | Infrastructure issues (Business/Enterprise support) |

---

## 5. HIPAA Breach Notification Rules

### 5.1 Breach Definition

A breach is the acquisition, access, use, or disclosure of PHI in a manner not permitted under the HIPAA Privacy Rule that compromises the security or privacy of the PHI. A breach is **presumed** unless the entity demonstrates a **low probability** of compromise based on the four-factor risk assessment.

### 5.2 Four-Factor Risk Assessment

When a potential breach is identified, evaluate:

1. **Nature and extent of PHI involved:** What types of identifiers? Clinical data? Financial data? How likely is re-identification?
2. **Unauthorized person:** Who accessed or received the PHI? Was it another covered entity? A member of the workforce? An unknown external party?
3. **Whether PHI was actually acquired or viewed:** Was the data accessed, or just exposed? Is there evidence of actual viewing or download?
4. **Extent of mitigation:** What steps were taken to reduce risk? Was the data recovered? Was the recipient trustworthy?

If the assessment demonstrates low probability of compromise, document the analysis and retain for 6 years. No notification required.

### 5.3 Notification Requirements

| Scenario | Individual Notification | HHS Notification | Media Notification |
|----------|----------------------|------------------|-------------------|
| < 500 individuals affected | Without unreasonable delay, no later than 60 days from discovery | Annual log submitted to HHS (within 60 days of calendar year end) | Not required |
| >= 500 individuals affected | Without unreasonable delay, no later than 60 days from discovery | Notify HHS simultaneously with individual notification (without unreasonable delay) | Notify prominent media outlets in affected state(s)/jurisdiction(s) |

### 5.4 Exceptions (Not a Breach)

- Unintentional access by workforce member acting in good faith, within scope of authority, no further use/disclosure
- Inadvertent disclosure between authorized persons at same CE/BA
- Good faith belief that the unauthorized recipient could not retain the information

---

## 6. Post-Incident Review

### 6.1 Timeline

| Action | When |
|--------|------|
| Incident report draft | Within 48 hours of resolution |
| Root Cause Analysis (RCA) meeting | Within 5 business days |
| Final incident report | Within 10 business days |
| Preventive measures implemented | Per remediation SLA |
| Follow-up review | 30 days after preventive measures deployed |

### 6.2 Root Cause Analysis Template

```
INCIDENT ID: [INC-YYYY-NNN]
DATE OF INCIDENT: [Date]
DATE OF DISCOVERY: [Date]
DURATION: [Time from occurrence to resolution]
SEVERITY: [P1/P2/P3/P4]

TIMELINE:
[HH:MM UTC] - [Event description]
[HH:MM UTC] - [Event description]
...

WHAT HAPPENED:
[Factual description of the incident]

ROOT CAUSE:
[Technical root cause -- focus on systemic issues, not blame]

CONTRIBUTING FACTORS:
- [Factor 1]
- [Factor 2]

IMPACT:
- Patients affected: [Number]
- PHI types exposed: [List]
- Service downtime: [Duration]
- Tenants affected: [List]

ACTIONS TAKEN DURING INCIDENT:
- [Action 1]
- [Action 2]

PREVENTIVE MEASURES:
| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| [Measure] | [Name] | [Date] | [Pending/Done] |

LESSONS LEARNED:
- [Lesson 1]
- [Lesson 2]

PROCESS IMPROVEMENTS:
- [Improvement 1]
- [Improvement 2]
```

### 6.3 Documentation Retention

- All incident reports retained for **6 years** (HIPAA minimum)
- Breach assessment documentation retained for **6 years** regardless of outcome
- Post-incident review documentation retained for **6 years**

---

## 7. References

- [[HIPAA-Deep-Dive]] -- HIPAA regulatory framework, including Breach Notification Rule (Section 9)
- [[HIPAA-Risk-Assessment]] -- Risk register and PHI inventory
- [[Auth-SMART-on-FHIR]] -- Authentication, authorization, and audit trail architecture
- [[System-Architecture-Overview]] -- System architecture reference
- 45 CFR 164.400-414 -- HIPAA Breach Notification Rule
- 45 CFR 164.308(a)(6) -- Security Incident Procedures
- NIST SP 800-61 Rev. 2 -- Computer Security Incident Handling Guide
