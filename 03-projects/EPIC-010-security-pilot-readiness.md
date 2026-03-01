---
type: epic
date: "2026-02-28"
status: in-progress
priority: 1
tags:
  - epic
  - sprint-5
  - security
  - pilot
  - phase-1
owner: ""
target-date: "2026-05-23"
---

# EPIC-010: Security Hardening & Pilot Readiness

> **Timeline:** Weeks 11-12 (Sprint S5)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-009-revenue-cycle-completion]] (done)
> **Blocks:** [[EPIC-006-pilot-readiness]] (Sprint 6 go-live)

## Objective

Harden MedOS for production deployment and prepare for pilot practice onboarding. This epic covers the full security audit cycle (HIPAA risk assessment, penetration testing, OWASP mitigation), field-level encryption for sensitive identifiers, tenant onboarding automation, EHR integration bridge, staff training materials, practice configuration, monitoring/alerting, incident response planning, and load testing. Completing this epic means MedOS is ready to onboard its first pilot practice with confidence that PHI is protected, the system handles production load, and staff can operate independently.

Sprint 4 deliverables: Prior Auth agent, Denial Management agent, underpayment detection, claims analytics, X12 pipeline. See [[EPIC-009-revenue-cycle-completion]].

---

## Timeline (Gantt)

```
Week 11 (Sprint 5, Part 1)
|----- T1: HIPAA Security Risk Assessment ---------|
|---- T2: Penetration Test (scope + engage) -------|
|---- T3: Security Hardening Checklist -------------|
|---- T4: Field-Level Encryption -------------------|

Week 12 (Sprint 5, Part 2)
|---- T5: Tenant Onboarding Wizard -----------------|
|---- T6: Data Migration Tool ----------------------|
|---- T7: EHR Integration Bridge -------------------|
|---- T8: User Training Materials ------------------|
|---- T9: Practice Configuration Panel -------------|
|---- T10: Monitoring & Alerting -------------------|
|---- T11: Incident Response Playbook --------------|
|---- T12: Load Testing ----------------------------|
```

---

## Tasks

### T1: HIPAA Security Risk Assessment
**Task ID:** S5-T01
**Complexity:** L
**Estimate:** 6h
**Owner:** B
**Dependencies:** All prior sprints
**References:** [[HIPAA-Deep-Dive]], [[SOC2-HITRUST-Roadmap]]

**Description:**
Document all PHI data flows through MedOS (ingestion, processing, storage, transmission, display). Identify risks at each flow point and document mitigations. Produce a formal risk register with risk severity, likelihood, and residual risk after controls. This is required for HIPAA compliance and will be presented to pilot practices during onboarding.

**Acceptance Criteria:**
- [ ] All PHI flows documented (audio -> transcript -> SOAP note -> claim -> remittance)
- [ ] Risk register with severity/likelihood/mitigation for each identified risk
- [ ] All high-severity risks have documented mitigations
- [ ] BAA requirements verified (AWS, Anthropic/Bedrock, Auth0)
- [ ] Document ready for pilot practice due diligence review

**Status:** pending

---

### T2: Penetration Test
**Task ID:** S5-T02
**Complexity:** L
**Estimate:** 8h
**Owner:** A
**Dependencies:** All prior sprints
**References:** [[HIPAA-Deep-Dive]], [[SOC2-HITRUST-Roadmap]]

**Description:**
Scope and engage a third-party penetration testing vendor to test MedOS API endpoints, authentication flows, tenant isolation, and FHIR resource access controls. Remediate all critical and high findings before pilot launch. Pen test report will be shared with pilot practices as evidence of security posture.

**Acceptance Criteria:**
- [ ] Pen test scope defined (API, auth, multi-tenancy, FHIR, MCP gateway)
- [ ] Vendor engaged and test scheduled
- [ ] Test completed with formal report
- [ ] Zero critical findings remaining
- [ ] All high findings remediated or mitigated with documented timeline
- [ ] Report available for pilot practice due diligence

**Status:** pending

---

### T3: Security Hardening Checklist
**Task ID:** S5-T03
**Complexity:** M
**Estimate:** 4h
**Owner:** A
**Dependencies:** S0-T15
**References:** [[HIPAA-Deep-Dive]], [[Auth-SMART-on-FHIR]], [[NextJS-Healthcare-Frontend]]

**Description:**
Comprehensive security hardening pass across the entire stack: WAF rules tuned for healthcare-specific attacks, rate limiting on all API endpoints (especially auth and FHIR search), input validation on all user-facing endpoints, error message sanitization to ensure no PHI leaks in error responses, CORS configuration locked to known origins, and CSP headers for the frontend.

**Acceptance Criteria:**
- [ ] WAF rules tuned (SQL injection, XSS, path traversal, FHIR-specific patterns)
- [ ] Rate limiting configured on all API endpoints (auth: 10/min, FHIR search: 100/min, write: 50/min)
- [ ] Input validation comprehensive on all endpoints (Pydantic models, max lengths, allowed characters)
- [ ] Error responses sanitized -- no PHI in any error message or stack trace
- [ ] CORS locked to known origins (frontend domain only)
- [ ] CSP headers configured for Next.js frontend
- [ ] OWASP Top 10 checklist verified

**Status:** in-progress

---

### T4: Field-Level Encryption
**Task ID:** S5-T04
**Complexity:** M
**Estimate:** 4h
**Owner:** A
**Dependencies:** S0-T08, S0-T33
**References:** [[ADR-002-multi-tenancy-isolation]], [[HIPAA-Deep-Dive]]

**Description:**
Implement field-level encryption for SSN and other sensitive identifiers (MRN, insurance subscriber ID) using per-tenant KMS keys. When data is at rest in PostgreSQL, these fields contain ciphertext. Decryption happens only in the application layer with the tenant's KMS key, ensuring that a database breach does not expose raw identifiers.

**Acceptance Criteria:**
- [ ] SSN encrypted at field level using tenant-specific KMS key
- [ ] MRN and subscriber ID encrypted at field level
- [ ] Database inspection shows ciphertext, not plaintext
- [ ] Decryption only with correct tenant KMS key
- [ ] Key rotation supported without re-encrypting all data (envelope encryption)
- [ ] Performance impact < 50ms per field encryption/decryption operation

**Status:** pending

---

### T5: Tenant Onboarding Wizard
**Task ID:** S5-T05
**Complexity:** L
**Estimate:** 6h
**Owner:** B
**Dependencies:** S0-T33
**References:** [[ADR-002-multi-tenancy-isolation]]

**Description:**
Build a multi-step onboarding wizard that allows a new practice to self-onboard: enter organization info (name, NPI, Tax ID, address), create admin user account, configure practice settings (specialties, locations, providers, payer contracts). The wizard creates the tenant schema, provisions KMS key, seeds initial configuration, and activates the tenant.

**Acceptance Criteria:**
- [ ] Multi-step wizard UI (org info -> admin user -> practice config -> confirmation)
- [ ] Tenant schema created with RLS policies
- [ ] Per-tenant KMS key provisioned
- [ ] Admin user account created with appropriate RBAC roles
- [ ] Practice configuration seeded (specialties, locations, providers)
- [ ] Payer contracts configurable during onboarding
- [ ] Wizard validates all inputs before proceeding to next step
- [ ] Onboarding completion triggers welcome email

**Status:** in-progress

---

### T6: Data Migration Tool
**Task ID:** S5-T06
**Complexity:** L
**Estimate:** 6h
**Owner:** A
**Dependencies:** S1-T06
**References:** [[FHIR-R4-Deep-Dive]], [[Clinical-Workflows-Overview]]

**Description:**
Build a data migration tool that imports patient demographics from CSV and HL7v2 sources, validates against FHIR Patient resource schema, creates FHIR Patient resources in the tenant's schema, and runs the patient matching algorithm to detect and merge duplicates. Designed for pilot onboarding where practices need to migrate existing patient panels.

**Acceptance Criteria:**
- [ ] CSV import with configurable column mapping
- [ ] HL7v2 ADT message parsing for demographics
- [ ] FHIR Patient resource validation before creation
- [ ] Patient matching algorithm runs on imported records
- [ ] Duplicate detection with configurable match threshold
- [ ] Manual review queue for uncertain matches (< 5% of total)
- [ ] Import report: total records, created, matched, duplicates, errors
- [ ] Handles 1000+ patients in a single import batch

**Status:** pending

---

### T7: EHR Integration Bridge
**Task ID:** S5-T07
**Complexity:** L
**Estimate:** 6h
**Owner:** A
**Dependencies:** S1-T01
**References:** [[FHIR-R4-Deep-Dive]], [[Auth-SMART-on-FHIR]]

**Description:**
Build a FHIR R4 client that connects to Epic and Cerner sandbox environments, reads patient demographics, and creates corresponding MedOS Patient resources. This bridge enables bidirectional sync of demographics during pilot, with MedOS as the clinical documentation and billing layer and the existing EHR as the system of record for demographics.

**Acceptance Criteria:**
- [ ] FHIR R4 client with SMART on FHIR authentication
- [ ] Read Patient resources from Epic sandbox
- [ ] Read Patient resources from Cerner sandbox
- [ ] Create corresponding MedOS Patient resources with provenance tracking
- [ ] Bidirectional demographics sync (name, DOB, address, phone, insurance)
- [ ] Conflict resolution strategy for concurrent edits
- [ ] Connection health monitoring and retry logic

**Status:** pending

---

### T8: User Training Materials
**Task ID:** S5-T08
**Complexity:** M
**Estimate:** 6h
**Owner:** B
**Dependencies:** S2-T09, S3-T10
**References:** [[Clinical-Workflows-Overview]], [[Ambient-AI-Documentation]]

**Description:**
Create video walkthroughs (Loom recordings) for three core user personas: providers (AI documentation workflow), billing staff (claims and denial management), and practice administrators (configuration and onboarding). Each video < 10 minutes, covering the complete workflow with real MedOS screens.

**Acceptance Criteria:**
- [ ] Provider workflow video: start encounter -> AI documentation -> review SOAP note -> approve -> sign
- [ ] Billing workflow video: check eligibility -> review claim -> submit -> track remittance -> manage denials
- [ ] Admin workflow video: onboard practice -> configure settings -> manage users -> view analytics
- [ ] Each video < 10 minutes
- [ ] Videos accessible from within MedOS help section
- [ ] Written quick-start guides accompanying each video

**Status:** pending

---

### T9: Practice Configuration Panel
**Task ID:** S5-T09
**Complexity:** M
**Estimate:** 4h
**Owner:** B
**Dependencies:** S5-T05
**References:** [[ADR-002-multi-tenancy-isolation]]

**Description:**
Build an admin configuration panel where practice administrators can manage all practice settings post-onboarding: add/remove providers, manage locations, update fee schedules, configure payer contracts, set specialty-specific settings (e.g., orthopedic-specific CPT code favorites), and manage user roles.

**Acceptance Criteria:**
- [ ] Provider management: add, edit, deactivate providers (NPI, specialty, schedule)
- [ ] Location management: add, edit locations (address, phone, place of service code)
- [ ] Fee schedule management: upload/edit fee schedules per payer
- [ ] Payer contract management: add/edit payer contracts with effective dates
- [ ] Specialty settings: CPT favorites, diagnosis favorites, default modifiers
- [ ] User role management: assign/revoke roles (provider, billing, admin, front desk)
- [ ] All changes audit-logged

**Status:** in-progress

---

### T10: Monitoring & Alerting
**Task ID:** S5-T10
**Complexity:** M
**Estimate:** 3h
**Owner:** A
**Dependencies:** S0-T17
**References:** [[AWS-HIPAA-Infrastructure]]

**Description:**
Implement CloudWatch dashboards showing API latency, error rates, throughput, and resource utilization. Configure PagerDuty integration for critical alerts. Dashboard should provide at-a-glance view of system health for the on-call engineer.

**Acceptance Criteria:**
- [ ] CloudWatch dashboard: API latency (P50, P95, P99), error rate, throughput, CPU/memory
- [ ] Alerts configured: error rate > 5%, P99 > 2s, CPU > 80%, memory > 85%
- [ ] PagerDuty integration for critical alerts (on-call rotation)
- [ ] LLM-specific metrics: token usage, Langfuse traces, agent success rate
- [ ] Cost monitoring dashboard: daily/weekly AWS spend with budget alerts
- [ ] Dashboard accessible to on-call engineer without AWS console access

**Status:** pending

---

### T11: Incident Response Playbook
**Task ID:** S5-T11
**Complexity:** S
**Estimate:** 3h
**Owner:** B
**Dependencies:** None
**References:** [[HIPAA-Deep-Dive]], [[SOC2-HITRUST-Roadmap]]

**Description:**
Create incident response playbook covering: data breach notification procedures (HIPAA 60-day rule), system outage response, security incident classification and escalation, on-call rotation schedule, and communication templates for affected parties. Test with a tabletop exercise before pilot launch.

**Acceptance Criteria:**
- [ ] Data breach notification procedure documented (detection -> investigation -> notification -> remediation)
- [ ] System outage response procedure (detection -> triage -> communication -> resolution -> postmortem)
- [ ] Security incident classification matrix (critical/high/medium/low with response times)
- [ ] On-call rotation schedule defined
- [ ] Contact list current (team, legal, HIPAA officer, payer contacts)
- [ ] Communication templates for breach notification
- [ ] Tabletop exercise conducted with at least one scenario

**Status:** pending

---

### T12: Load Testing
**Task ID:** S5-T12
**Complexity:** M
**Estimate:** 4h
**Owner:** A
**Dependencies:** All prior sprints
**References:** [[AWS-HIPAA-Infrastructure]]

**Description:**
Simulate pilot practice load: 50 concurrent users, 100 encounters per hour, with FHIR CRUD operations, AI documentation generation, claims processing, and analytics queries running simultaneously. Verify the system handles this load within latency targets and without data loss.

**Acceptance Criteria:**
- [ ] Load test script simulating 50 concurrent users across all workflows
- [ ] 100 encounters/hour sustained for 1 hour
- [ ] API P99 latency < 1 second under load
- [ ] Zero errors under sustained load
- [ ] Zero data loss verified post-test
- [ ] AI pipeline (Whisper + Claude) handles concurrent requests without queue overflow
- [ ] Database connection pool adequate (no connection exhaustion)
- [ ] Load test report with bottleneck analysis

**Status:** pending

---

## Dependencies Map

```
T1 (Risk Assessment) ─────────────┐
T2 (Pen Test) ────────────────────┤
T3 (Hardening) ───────────────────┤
T4 (Field Encryption) ────────────┤
                                   ├──> Sprint 6 (Pilot Go-Live)
T5 (Onboarding Wizard) ──> T9 (Config Panel)
T6 (Data Migration) ──────────────┤
T7 (EHR Bridge) ──────────────────┤
T8 (Training) ────────────────────┤
T10 (Monitoring) ─────────────────┤
T11 (Incident Response) ──────────┤
T12 (Load Test) ──────────────────┘
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| HIPAA Risk Assessment (T1) | [[EPIC-006-pilot-readiness]] -- pilot practices require evidence of risk assessment |
| Pen Test Report (T2) | [[EPIC-006-pilot-readiness]] -- pilot practices require pen test results |
| Tenant Onboarding Wizard (T5) | [[EPIC-006-pilot-readiness]] -- S6-T03 uses wizard to onboard first practice |
| Data Migration Tool (T6) | [[EPIC-006-pilot-readiness]] -- S6-T03 imports patient demographics |
| Training Materials (T8) | [[EPIC-006-pilot-readiness]] -- S6-T04 conducts training using these materials |
| Monitoring & Alerting (T10) | [[EPIC-006-pilot-readiness]] -- S6-T06 daily monitoring uses these dashboards |
| Practice Config Panel (T9) | [[EPIC-006-pilot-readiness]] -- admin self-service post-onboarding |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Pen test finds critical vulnerability in multi-tenancy isolation | Medium | Critical | Proactive tenant isolation testing in prior sprints; schema-per-tenant + RLS already implemented |
| EHR sandbox credentials delayed by Epic/Cerner | High | Medium | Start with mock FHIR server; real sandbox connection can follow pilot launch |
| Load test reveals performance bottleneck in AI pipeline | Medium | High | GPU autoscaling configured; queue-based processing prevents cascade failure |
| Pilot practice requires integrations not yet built | Medium | Medium | Scope pilot to workflows already supported; document integration roadmap for extras |
| HIPAA risk assessment reveals gaps requiring significant rework | Low | High | Continuous compliance throughout Phase 1 (audit trail, encryption, RLS from Sprint 0) |

---

## Definition of Done

- [ ] HIPAA Security Risk Assessment complete with risk register
- [ ] Penetration test complete with zero critical findings
- [ ] Security hardening checklist verified (OWASP Top 10)
- [ ] Field-level encryption for SSN and sensitive identifiers
- [ ] Tenant onboarding wizard functional end-to-end
- [ ] Data migration tool handling 1000+ patient imports
- [ ] EHR integration bridge connecting to Epic/Cerner sandbox
- [ ] Training materials for 3 user personas
- [ ] Practice configuration panel live
- [ ] Monitoring dashboards and alerting configured
- [ ] Incident response playbook documented and tested
- [ ] Load test passing at pilot scale (50 users, 100 encounters/hour)

---

## References

- [[EPIC-009-revenue-cycle-completion]] -- Sprint 4 scope (X12 pipeline, PA, denial mgmt)
- [[EPIC-006-pilot-readiness]] -- Sprint 6 go-live (depends on this epic)
- [[HIPAA-Deep-Dive]] -- HIPAA compliance requirements
- [[SOC2-HITRUST-Roadmap]] -- Compliance timeline and costs
- [[ADR-002-multi-tenancy-isolation]] -- Tenant isolation architecture
- [[Auth-SMART-on-FHIR]] -- Authentication and EHR integration
- [[AWS-HIPAA-Infrastructure]] -- Infrastructure hardening
- [[agent-architecture]] -- Agent security pipeline
- [[mcp-integration-plan]] -- MCP gateway security
