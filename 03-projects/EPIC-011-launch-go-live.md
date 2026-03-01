---
type: epic
date: "2026-02-28"
status: in-progress
tags:
  - epic
  - launch
  - go-live
  - phase-1
  - sprint-6
---

# EPIC-011: Launch & Go-Live

## Goal

Deploy production environment, seed demo data, configure monitoring and communication channels, and prepare for first pilot practice onboarding. This EPIC covers all Sprint 6 tasks -- the final sprint of Phase 1.

**References:** [[PHASE-1-EXECUTION-PLAN]], [[EPIC-010-security-pilot-readiness]], [[monitoring-rotation-runbook]], [[pilot-communication-plan]]

---

## Sprint 6 Tasks

| ID | Task | Owner | Est. | Status |
|----|------|-------|------|--------|
| S6-T01 | Deploy production environment: Terraform apply, health checks, SSL | A | 4h | done |
| S6-T02 | Create demo environment with synthetic patient data (50 patients, 200 encounters, claims) | A | 3h | done |
| S6-T03 | Pilot practice onboarding (Practice #1): onboarding wizard, patient import, payer config | B | 4h | pending |
| S6-T04 | On-site training: providers, billing, front desk (2-hour sessions per role group) | B | 6h | pending |
| S6-T05 | Configure pilot success metrics tracking: time saved, coding accuracy, claim rates, adoption | B | 3h | done |
| S6-T06 | Set up daily monitoring rotation: error rates, AI quality, costs, support | A | 2h | done |
| S6-T07 | Configure automated backup verification: daily test restore, alert on failure | A | 2h | done |
| S6-T08 | Create pilot feedback channel: Slack, weekly cadence, issue tracker | BOTH | 1h | done |
| S6-T09 | Launch day go-live support: on-call for first full day, real-time resolution | BOTH | 8h | pending |

**Done:** 6/9 tasks (T01, T02, T05, T06, T07, T08)
**Pending:** 3/9 tasks (T03, T04, T09 -- require actual pilot practice)

---

## Go-Live Checklist

### Infrastructure
- [x] Production Terraform applied, all services healthy
- [x] SSL certificates valid, HTTPS enforced
- [x] Health check endpoints responding on all services
- [x] Auto-scaling configured and tested
- [x] Backup verification automated (daily test restore)
- [x] Monitoring dashboards configured (CloudWatch)
- [x] Alerting rules active (P1-P4 thresholds)

### Data
- [x] Demo environment seeded (50 patients, 200 encounters, sample claims)
- [x] Synthetic data verified as non-PHI
- [ ] Pilot practice patient demographics imported
- [ ] Payer configurations loaded for pilot practice

### Security
- [x] HIPAA Risk Assessment completed (Sprint 5)
- [x] Pen test passed, zero critical/high findings (Sprint 5)
- [x] Field-level encryption active (SSN, MRN, DOB)
- [x] Audit logging operational
- [x] Incident response playbook documented and tested

### Training
- [x] Training materials created (provider, billing, admin workflows)
- [ ] Provider training session conducted
- [ ] Billing staff training session conducted
- [ ] Front desk training session conducted

### Communication
- [x] Slack channels created (#medos-pilot-support, #medos-pilot-feedback)
- [x] Bug reporting process documented (GitHub Issues template)
- [x] Weekly check-in cadence established
- [x] Escalation matrix documented
- [ ] First weekly check-in scheduled on calendar

### Monitoring
- [x] Daily monitoring rotation documented
- [x] First week rotation schedule assigned
- [x] Alert response procedures documented
- [x] Daily report template created
- [x] Weekly metrics review template created

### Go-Live Day
- [ ] Both team members on-call for full day
- [ ] First real patient encounter processed successfully
- [ ] Zero blocking issues by end of Day 1
- [ ] Day 1 monitoring report filed

---

## Success Metrics

Defined in [[pilot-communication-plan]] and tracked via the pilot metrics dashboard:

| Metric | Target | Measurement Frequency |
|--------|--------|----------------------|
| Time saved per provider per day | > 30 min | Weekly |
| AI coding accuracy (ICD-10/CPT) | > 92% | Weekly (sample 20 notes) |
| Claim first-pass acceptance rate | > 95% | Weekly |
| Denial rate | < 5% | Weekly |
| Daily active users | > 80% of staff | Daily |
| AI-generated notes | > 70% of encounters | Daily |
| Provider NPS | > 50 | Monthly |
| System uptime | > 99.5% | Continuous |

---

## Dependencies

| Dependency | Source | Status |
|-----------|--------|--------|
| Security hardening complete | [[EPIC-010-security-pilot-readiness]] | done |
| Training materials created | S5-T08 | done |
| Monitoring infrastructure | S5-T10 | done |
| Tenant onboarding wizard | S5-T05 | done |
| Data migration tool | S5-T06 | done |
| Revenue cycle features | [[EPIC-009-revenue-cycle-completion]] | done |
| Actual pilot practice agreement | Business | pending |

---

## Deliverables

1. Production environment fully operational
2. Demo environment with synthetic data
3. Monitoring rotation with documented procedures ([[monitoring-rotation-runbook]])
4. Communication plan with all channels active ([[pilot-communication-plan]])
5. Pilot success metrics dashboard configured
6. Backup verification automated
7. First pilot practice onboarded, trained, and live (pending practice agreement)

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Pilot practice delays signing agreement | Medium | High | Start with demo environment; schedule onboarding ASAP after signing |
| First-day critical bug | Low | High | Both engineers on-call; rollback plan documented; incident response playbook ready |
| Provider resistance to AI documentation | Medium | Medium | Emphasize review workflow; confidence scoring makes AI transparent; training focuses on benefits |
| Higher-than-expected AWS costs | Low | Medium | Daily cost monitoring; auto-scaling limits set; reserved instances for baseline load |
