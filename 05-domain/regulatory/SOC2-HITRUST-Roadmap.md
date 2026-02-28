---
type: domain-knowledge
date: "2026-02-27"
tags:
  - domain
  - regulatory
  - soc2
  - hitrust
  - compliance
  - module-g
category: regulatory
confidence: high
sources: []
---

# SOC 2 and HITRUST Roadmap

This document lays out the complete compliance certification roadmap for a healthcare SaaS platform. SOC 2 and HITRUST are not optional "nice-to-haves" -- they are table-stakes requirements for selling to hospitals, health systems, and insurance companies. See [[HEALTHCARE_OS_MASTERPLAN]] for overall product context, [[HIPAA-Deep-Dive]] for the regulatory foundation, and [[MOC-Architecture]] for the technical architecture that supports these compliance programs.

---

## 1. SOC 2 Type I vs Type II

### SOC 2 Type I

A **point-in-time** assessment. The auditor evaluates whether your controls are **designed** appropriately as of a specific date. Think of it as a snapshot.

- **What it proves**: "As of March 15, 2027, Company X had controls in place that were suitably designed to meet the Security trust service criteria."
- **Duration**: 4-8 weeks of auditor engagement
- **Cost**: $20,000-50,000
- **Value**: Gets you past procurement gates quickly. Many enterprise buyers accept Type I while you work toward Type II.

### SOC 2 Type II

An **observation period** assessment. The auditor evaluates whether your controls **operated effectively** over a period of time (typically 6-12 months).

- **What it proves**: "From March 15, 2027 through March 14, 2028, Company X's controls operated effectively to meet the Security trust service criteria."
- **Duration**: 6-12 month observation period, then 4-8 weeks of auditor fieldwork
- **Cost**: $30,000-80,000
- **Value**: The gold standard. Required by most enterprise healthcare buyers. Demonstrates sustained operational discipline, not just good intentions.

### Key Difference

Type I says "we have locks on the doors." Type II says "we kept the locks on the doors and used them correctly for the past year." Healthcare buyers overwhelmingly prefer Type II, but will accept Type I as interim evidence while you build your observation period.

---

## 2. SOC 2 Trust Service Criteria

SOC 2 is organized around five Trust Service Criteria (TSC). You choose which ones to include in your audit scope.

### Security (Required -- always in scope)

The foundation. Covers protection against unauthorized access. Includes:
- Logical and physical access controls
- System operations and monitoring
- Change management
- Risk mitigation

### Availability

System uptime and operational commitments. Covers:
- SLA performance and monitoring
- Disaster recovery and business continuity
- Incident response and communication
- Capacity planning

### Processing Integrity

Data is processed completely, accurately, and in a timely manner. Covers:
- Input validation and error handling
- Data processing monitoring
- Quality assurance procedures
- Output reconciliation

### Confidentiality

Protection of data designated as confidential (distinct from privacy). Covers:
- Data classification policies
- Encryption at rest and in transit
- Access restrictions based on data classification
- Secure data disposal

### Privacy

Collection, use, retention, disclosure, and disposal of personal information. Covers:
- Privacy notice and consent
- Data subject rights (access, correction, deletion)
- Data retention policies
- Third-party data sharing agreements

---

## 3. SOC 2 for Healthcare SaaS

### Which Criteria Matter Most

For a healthcare SaaS platform, include **all five criteria** in your SOC 2 scope. Here is why:

| Criteria | Healthcare Relevance |
|----------|---------------------|
| Security | Non-negotiable. Every healthcare buyer requires this. |
| Availability | Clinicians depend on uptime. Downtime can affect patient care. |
| Processing Integrity | Clinical data must be accurate. Errors have life-safety implications. |
| Confidentiality | PHI is the definition of confidential data. |
| Privacy | Patients have rights under HIPAA. Buyers will ask about these controls. |

### What Auditors Look For

1. **Access reviews** -- quarterly reviews of who has access to what. Documented, with evidence of revocations.
2. **Change management** -- every code change has a ticket, a review, and an approval trail. No "cowboy commits" to production.
3. **Incident response** -- documented plan, tested at least annually, with post-incident reviews.
4. **Vendor management** -- BAAs with every subprocessor that touches PHI. Annual vendor risk assessments.
5. **Encryption everywhere** -- at rest (KMS), in transit (TLS 1.2+), and key management procedures.
6. **Monitoring and alerting** -- evidence that anomalies are detected and investigated, not just logged.
7. **Employee training** -- annual security awareness training with completion records.
8. **Background checks** -- for all employees with access to production systems or PHI.
9. **Risk assessment** -- formal, documented risk assessment performed at least annually. Not a checkbox exercise -- auditors want to see real analysis.
10. **Board/management oversight** -- evidence that leadership reviews security posture regularly.

---

## 4. Our SOC 2 Timeline

### Month 1-5: Foundation Building

- Implement all technical controls (see [[AWS-HIPAA-Infrastructure]])
- Write policies and procedures (Information Security Policy, Acceptable Use, Incident Response, Change Management, Access Control, Data Classification, Business Continuity)
- Select compliance automation platform (Vanta, Drata, or Secureframe)
- Connect AWS accounts to compliance platform for automated evidence collection
- Begin collecting evidence: access reviews, change management logs, training records
- Conduct initial risk assessment

### Month 6: SOC 2 Type I Readiness

- Engage SOC 2 audit firm (select 2-3 months in advance)
- Conduct internal readiness assessment against TSC
- Remediate any gaps identified
- Auditor performs Type I fieldwork (2-4 weeks)

### Month 7: Type I Report Issued

- Receive Type I report
- Address any observations or recommendations
- Begin using Type I report for customer procurement processes

### Month 7-14: Type II Observation Period (6 months minimum)

- Continue operating controls consistently
- Monthly internal compliance reviews
- Quarterly access reviews (documented)
- Maintain change management discipline
- Collect and preserve all evidence in compliance platform
- Conduct tabletop incident response exercise (at least once)

### Month 14-15: Type II Fieldwork and Report

- Auditor performs Type II fieldwork (4-6 weeks)
- Auditor samples evidence from the observation period
- Receive Type II report
- Address any exceptions or qualifications

### Ongoing: Annual Type II Renewal

- New observation period begins immediately after the previous one ends
- Annual audit fieldwork
- Continuous evidence collection and monitoring

---

## 5. HITRUST CSF

### What It Is

The HITRUST Common Security Framework (CSF) is a certifiable compliance framework that maps to and incorporates requirements from multiple regulatory standards:

- HIPAA Security Rule
- HIPAA Privacy Rule
- NIST 800-53
- ISO 27001/27002
- PCI DSS
- COBIT
- State privacy laws (CCPA, etc.)
- FTC Red Flags Rule

### Why Healthcare Demands It

HIPAA is a law, not a certification. There is no "HIPAA certified" stamp. This creates a problem: how does a hospital know if a vendor is actually HIPAA compliant? Self-attestation is meaningless.

HITRUST solves this by providing a **certifiable** framework that covers HIPAA requirements. When you achieve HITRUST certification, healthcare buyers can point to a third-party validated assessment rather than conducting their own vendor security audit.

**Practical impact**: Without HITRUST, every hospital prospect sends you a 300-question security questionnaire. With HITRUST, you hand them your certification letter and skip the questionnaire. This directly reduces sales cycle length.

### How It Maps to HIPAA

HITRUST CSF v11 contains 156 control references organized into 19 domains. Each control maps back to specific HIPAA Security Rule requirements (plus other frameworks). When you satisfy a HITRUST control, you simultaneously satisfy the corresponding HIPAA requirement.

Key domains:
1. Information Protection Program
2. Endpoint Protection
3. Portable Media Security
4. Mobile Device Security
5. Wireless Security
6. Configuration Management
7. Vulnerability Management
8. Network Protection
9. Transmission Protection
10. Password Management
11. Access Control
12. Audit Logging and Monitoring
13. Education, Training, and Awareness
14. Third-Party Assurance
15. Incident Management
16. Business Continuity and Disaster Recovery
17. Risk Management
18. Physical and Environmental Security
19. Data Protection and Privacy

---

## 6. HITRUST Assessment Types

### e1 Assessment (Essential, 1-year)

- **Scope**: 44 control requirements (fundamental cybersecurity hygiene)
- **Effort**: 2-4 months
- **Cost**: $15,000-30,000 (assessor fees)
- **Best for**: Early-stage companies demonstrating baseline security
- **Validity**: 1 year
- **Limitation**: Not sufficient for large health system procurement

### i1 Assessment (Implemented, 1-year)

- **Scope**: 182 control requirements (leading security practices)
- **Effort**: 4-6 months
- **Cost**: $30,000-60,000 (assessor fees)
- **Best for**: Companies that need more than e1 but are not ready for r2
- **Validity**: 1 year
- **Limitation**: Some large health systems require r2

### r2 Assessment (Risk-based, 2-year)

- **Scope**: Variable based on risk factors (typically 300-500+ control requirements)
- **Effort**: 6-12 months
- **Cost**: $50,000-120,000 (assessor fees)
- **Best for**: Companies selling to large health systems and payers
- **Validity**: 2 years (with annual interim assessment)
- **Advantage**: The gold standard. Accepted by virtually all healthcare organizations.

### Our Strategy

- **Phase 1 (Month 12)**: Begin with i1 assessment. It provides meaningful certification that satisfies most mid-market healthcare buyers while we build toward r2.
- **Phase 2 (Month 18-24)**: Pursue r2 validated assessment once the control environment is mature enough to sustain the heavier requirements.

---

## 7. HITRUST Timeline

### Month 1-11: Foundation and Control Implementation

- Implement technical controls aligned with HITRUST CSF domains
- Document all policies and procedures (many overlap with SOC 2 -- reuse)
- Deploy automated monitoring for HITRUST-specific controls
- Train staff on HITRUST requirements
- Select a HITRUST Authorized External Assessor

### Month 12-14: i1 Assessment

- **Readiness assessment** with assessor (Month 12)
- Gap remediation (2-4 weeks)
- **Validated assessment** fieldwork (Month 13)
- Assessor submits to HITRUST for quality review
- **Certification issued** (Month 14-15, depending on HITRUST review queue)

### Month 16-20: r2 Preparation

- Expand control environment to cover r2 scope
- Additional policy development for risk-based controls
- Implement any missing technical controls
- Internal mock assessment

### Month 20-26: r2 Validated Assessment

- **Readiness assessment** with assessor (Month 20)
- Gap remediation (4-8 weeks)
- **Validated assessment** fieldwork (Month 22-24)
- Assessor submits to HITRUST for quality review
- **Certification issued** (Month 24-26)

### Ongoing: Annual Interim Assessment

- r2 certification is valid for 2 years
- Annual interim assessment required to maintain certification
- Continuous monitoring and evidence collection

---

## 8. Compliance Automation Platform Comparison

### Vanta

| Aspect | Details |
|--------|---------|
| Healthcare focus | Strong. Native HIPAA and HITRUST modules. |
| AWS integration | Excellent. Automated evidence collection from CloudTrail, Config, IAM. |
| SOC 2 support | Full lifecycle: readiness, monitoring, auditor portal. |
| HITRUST support | Yes, including CSF mapping and evidence organization. |
| Vendor management | Built-in vendor risk assessment and tracking. |
| Pricing | $10,000-25,000/year (depends on scope and company size). |
| Strengths | Most mature healthcare compliance automation. Large auditor network. |
| Weaknesses | Can be expensive for very early-stage startups. |

### Drata

| Aspect | Details |
|--------|---------|
| Healthcare focus | Good. HIPAA module available. HITRUST in progress. |
| AWS integration | Excellent. Similar coverage to Vanta. |
| SOC 2 support | Full lifecycle with continuous monitoring. |
| HITRUST support | Growing. Not as mature as Vanta for HITRUST specifically. |
| Vendor management | Available. |
| Pricing | $8,000-20,000/year. |
| Strengths | Modern UI, good developer experience, competitive pricing. |
| Weaknesses | HITRUST module less mature than Vanta. |

### Secureframe

| Aspect | Details |
|--------|---------|
| Healthcare focus | Good. HIPAA and HITRUST modules available. |
| AWS integration | Strong. Automated evidence collection. |
| SOC 2 support | Full lifecycle. |
| HITRUST support | Available, improving rapidly. |
| Vendor management | Built-in. |
| Pricing | $8,000-18,000/year. |
| Strengths | Fastest setup time. Good for lean teams. AI-powered policy generation. |
| Weaknesses | Smaller auditor network. Fewer healthcare-specific features. |

### Recommendation

**Vanta** for healthcare SaaS. The HITRUST integration is the most mature, the auditor network is the largest, and healthcare buyer procurement teams are familiar with Vanta reports. The price premium is justified by reduced friction in the sales process.

---

## 9. Evidence Collection (Day 1 Requirements)

### What to Collect from the Start

Every piece of evidence you collect from Day 1 makes your first audit dramatically easier and less expensive. Retrofitting evidence is painful and sometimes impossible.

### Automated Evidence (from AWS)

| Evidence Type | AWS Source | Collection Method |
|---------------|-----------|------------------|
| API activity logs | CloudTrail | Continuous, S3 + CloudWatch |
| Configuration compliance | AWS Config | Continuous, conformance packs |
| Threat detection | GuardDuty | Continuous, findings to Security Hub |
| Encryption status | KMS, RDS, S3 | AWS Config rules |
| Network flow logs | VPC Flow Logs | Continuous, CloudWatch Logs |
| Access management | IAM | Compliance platform integration |
| Vulnerability scanning | ECR, Inspector | On push + scheduled |
| Backup verification | RDS snapshots | Automated daily, verified monthly |

### Manual Evidence (process-driven)

| Evidence Type | Frequency | Owner | Tool |
|---------------|-----------|-------|------|
| Access reviews | Quarterly | Engineering lead | Compliance platform |
| Vendor risk assessments | Annually per vendor | Security lead | Compliance platform |
| Risk assessment | Annually | CTO/CISO | Documented process |
| Incident response test | Annually (minimum) | Engineering team | Tabletop exercise, documented |
| Security training | Annually per employee | HR/Engineering | LMS with completion tracking |
| Background checks | On hire | HR | Background check service |
| Policy reviews | Annually | CTO/CISO | Version-controlled documents |
| Change management logs | Continuous | Engineering | GitHub PRs + deployment logs |
| Penetration testing | Annually | External firm | Formal report |
| Business continuity test | Annually | Engineering | DR drill, documented |

### Change Management Evidence

This is the single most-sampled control in SOC 2 audits. Every change to production must have:

1. **A ticket** (GitHub Issue or Jira)
2. **A code review** (GitHub PR with at least one approval)
3. **Automated tests passing** (CI pipeline)
4. **Deployment record** (GitHub Actions run log)
5. **Rollback capability** (documented process)

```
GitHub Issue → Branch → PR (with review) → CI passes → Merge → Deploy → Verify
     ↑                    ↑                    ↑           ↑         ↑
   Evidence            Evidence             Evidence   Evidence  Evidence
```

### Incident Response Evidence

Maintain an incident log from Day 1, even if you have zero incidents. Auditors want to see:

- Incident response policy (written and approved)
- Incident classification criteria (severity levels)
- Communication templates (internal and external)
- Post-incident review template
- At least one tabletop exercise documented per year
- Any actual incidents documented with timeline, root cause, and remediation

---

## 10. Cost Breakdown

### SOC 2 Costs

| Item | Cost | Frequency |
|------|------|-----------|
| Compliance platform (Vanta) | $15,000-20,000 | Annual |
| Type I audit | $25,000-40,000 | One-time |
| Type II audit | $35,000-60,000 | Annual |
| Penetration testing | $10,000-25,000 | Annual |
| Security training platform | $2,000-5,000 | Annual |
| Background checks | $50-100/person | Per hire |
| Policy writing (if outsourced) | $5,000-15,000 | One-time |
| **Year 1 Total (Type I)** | **$57,000-105,000** | |
| **Year 2+ Total (Type II)** | **$62,000-110,000** | |

### HITRUST Costs

| Item | Cost | Frequency |
|------|------|-----------|
| i1 assessment (assessor fees) | $30,000-50,000 | Annual |
| r2 assessment (assessor fees) | $60,000-100,000 | Every 2 years |
| HITRUST subscription fee | $10,000-15,000 | Annual |
| Interim assessment | $15,000-25,000 | Annual (r2 years) |
| Gap remediation consulting | $10,000-30,000 | As needed |
| **i1 Year 1 Total** | **$50,000-95,000** | |
| **r2 Certification Total** | **$85,000-140,000** | |

### Combined Annual Compliance Budget

| Phase | SOC 2 | HITRUST | Tools | Total |
|-------|-------|---------|-------|-------|
| Year 1 (Type I + i1 prep) | $57K-105K | $50K-95K | $17K-25K | $124K-225K |
| Year 2 (Type II + r2 prep) | $62K-110K | $60K-100K | $17K-25K | $139K-235K |
| Year 3+ (Type II + r2 interim) | $62K-110K | $25K-40K | $17K-25K | $104K-175K |

### ROI Justification

- Without SOC 2 + HITRUST, each enterprise deal requires 40-80 hours of custom security questionnaire responses
- With certifications, security reviews take 2-4 hours (hand over the reports)
- At even 10 enterprise deals per year, certifications save 360-760 hours of sales engineering time
- Healthcare enterprise contracts are $50K-500K+ ARR; one closed deal pays for the entire compliance program

---

## 11. Practical Day 1 Checklist

Everything on this list should be implemented before writing a single line of application code. These items cost almost nothing to set up but become extremely expensive to retrofit.

### Identity and Access Management

- [ ] Enable MFA for all AWS IAM users (enforce via IAM policy)
- [ ] Enable MFA for all GitHub accounts
- [ ] Set up SSO (Google Workspace or Okta) for all SaaS tools
- [ ] Create an access onboarding/offboarding checklist
- [ ] Document who has access to what (maintain in a spreadsheet until you have a compliance platform)
- [ ] Implement least-privilege IAM policies (no `*` resources in production)

### Logging and Monitoring

- [ ] Enable CloudTrail in all regions (see [[AWS-HIPAA-Infrastructure]])
- [ ] Enable VPC Flow Logs
- [ ] Enable AWS Config with HIPAA conformance pack
- [ ] Enable GuardDuty
- [ ] Set up CloudWatch alarms for critical metrics
- [ ] Configure log retention: minimum 1 year for compliance, 7 years for healthcare best practice

### Encryption

- [ ] Enable encryption at rest for all data stores (RDS, S3, EBS, CloudWatch Logs)
- [ ] Enforce TLS 1.2+ for all connections
- [ ] Create per-tenant KMS keys
- [ ] Enable KMS key rotation
- [ ] Document encryption key management procedures

### Change Management

- [ ] Require PR reviews for all code changes (GitHub branch protection rules)
- [ ] Require CI passing before merge
- [ ] Tag all deployments with timestamps and commit SHAs
- [ ] Maintain a deployment log (automated via GitHub Actions)
- [ ] Document rollback procedures

### Policies (Write These Now)

- [ ] Information Security Policy
- [ ] Acceptable Use Policy
- [ ] Access Control Policy
- [ ] Change Management Policy
- [ ] Incident Response Plan
- [ ] Data Classification Policy
- [ ] Data Retention and Disposal Policy
- [ ] Business Continuity and Disaster Recovery Plan
- [ ] Vendor Management Policy
- [ ] Employee Security Training Policy

> **Tip**: Use your compliance platform's policy templates as a starting point. Vanta, Drata, and Secureframe all provide customizable templates that are pre-mapped to SOC 2 and HITRUST controls. Do not write these from scratch.

### Vendor Management

- [ ] Inventory all third-party services that touch data
- [ ] Obtain BAAs from any vendor that will process PHI
- [ ] Document vendor security assessment process
- [ ] Review vendor SOC 2 reports annually

### Incident Response

- [ ] Write incident response plan (classification, escalation, communication)
- [ ] Set up a dedicated incident communication channel (Slack channel or equivalent)
- [ ] Create incident report template
- [ ] Schedule first tabletop exercise within 3 months of launch
- [ ] Set up PagerDuty or Opsgenie for on-call alerting

### Training

- [ ] Select a security awareness training platform (KnowBe4, Curricula, or similar)
- [ ] Conduct initial security training for all team members
- [ ] Set up annual training cadence with completion tracking
- [ ] Include HIPAA-specific training modules
- [ ] Document training completion records

### Evidence Repository

- [ ] Select compliance automation platform (Vanta recommended)
- [ ] Connect AWS accounts for automated evidence collection
- [ ] Connect GitHub for change management evidence
- [ ] Connect HR systems for employee lifecycle evidence
- [ ] Set up quarterly evidence review calendar

---

## Framework Mapping

The good news: significant overlap exists between SOC 2 and HITRUST controls. Implementing one makes the other substantially easier.

| Control Area | SOC 2 TSC | HITRUST Domain | Overlap |
|-------------|-----------|----------------|---------|
| Access control | CC6.1-6.8 | 01 Access Control | ~85% |
| Change management | CC8.1 | 06 Configuration Mgmt | ~80% |
| Incident response | CC7.3-7.5 | 15 Incident Mgmt | ~75% |
| Risk assessment | CC3.1-3.4 | 17 Risk Mgmt | ~70% |
| Encryption | CC6.1, CC6.7 | 09 Transmission Protection | ~90% |
| Monitoring | CC7.1-7.2 | 12 Audit Logging | ~85% |
| Training | CC1.4 | 13 Education/Training | ~80% |
| Vendor mgmt | CC9.2 | 14 Third-Party Assurance | ~75% |

**Practical implication**: If you build for HITRUST r2, you automatically satisfy most SOC 2 requirements. Run them in parallel, not sequentially.

---

## Summary Timeline

```
Month 1-5:   Build infrastructure + Write policies + Deploy compliance platform
Month 6-7:   SOC 2 Type I audit and report
Month 7-14:  SOC 2 Type II observation period
Month 12-15: HITRUST i1 assessment and certification
Month 14-15: SOC 2 Type II audit and report
Month 16-20: Expand controls for HITRUST r2
Month 20-26: HITRUST r2 validated assessment and certification
Month 26+:   Annual SOC 2 Type II + HITRUST interim assessments
```

This timeline is aggressive but achievable for a focused team. The key insight: **start collecting evidence and following processes from Day 1**. The certifications are just third-party validation of what you are already doing. If you build compliance into your engineering culture from the beginning, audits are a formality rather than a fire drill.
