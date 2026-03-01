---
type: compliance
date: "2026-02-28"
tags:
  - security
  - pentest
  - compliance
  - phase-1
status: draft
---

# Penetration Test Scope -- MedOS Healthcare OS

> This document defines the scope, methodology, rules of engagement, and expected deliverables for the pre-pilot penetration test of MedOS. Cross-reference with [[HIPAA-Risk-Assessment]], [[HIPAA-Deep-Dive]], and [[Auth-SMART-on-FHIR]].

---

## 1. Objective

Identify exploitable vulnerabilities in the MedOS platform before pilot launch with real patient data. The penetration test will validate that technical controls documented in the [[HIPAA-Risk-Assessment]] function as designed and that no critical or high-severity vulnerabilities exist in the production-bound codebase.

---

## 2. Scope

### 2.1 In Scope

| System | Description | Access Level |
|--------|-------------|-------------|
| FastAPI Backend | All REST API endpoints (`/api/v1/*`, `/mcp/*`, `/.well-known/*`) | Authenticated + unauthenticated |
| Next.js Frontend | All routes, forms, and client-side logic | Browser-based testing |
| PostgreSQL Database | Access control validation, tenant isolation testing | Via API only (no direct DB access) |
| Authentication System | Auth0 integration, JWT handling, session management, MFA bypass attempts | Authenticated + unauthenticated |
| MCP Protocol Layer | MCP Gateway, tool registry, agent authentication, PHI policy enforcement | Authenticated (agent credentials) |
| LangGraph Agent Pipeline | Agent prompt injection, tool abuse, confidence score manipulation | Via API |
| API Rate Limiting | Brute force protection, DoS resilience at application layer | Unauthenticated |
| FHIR API | FHIR resource access, search parameter injection, cross-tenant queries | Authenticated |

### 2.2 Out of Scope

| System | Reason |
|--------|--------|
| AWS Infrastructure (VPC, EC2, RDS, KMS) | Covered by AWS Shared Responsibility Model; AWS maintains SOC 2, HIPAA compliance |
| Third-party clearinghouse APIs (Availity) | Separate vendor; covered by their own security assessments and BAA |
| Auth0 Identity Platform | Separate vendor with SOC 2 Type II certification |
| Mobile applications | Not yet built in MVP |
| Physical security | Assessed separately per HIPAA administrative safeguards |
| Social engineering | Not included in technical pen test scope |

---

## 3. Test Types

### 3.1 OWASP Top 10 (2021)

| # | Category | MedOS-Specific Focus |
|---|----------|---------------------|
| A01 | Broken Access Control | Cross-tenant PHI access, RBAC bypass, IDOR on FHIR resources |
| A02 | Cryptographic Failures | TLS configuration, field-level encryption validation, KMS key separation |
| A03 | Injection | SQL injection via FHIR search parameters, prompt injection on Claude/LangGraph agents |
| A04 | Insecure Design | Business logic flaws in claims pipeline, confidence score bypass |
| A05 | Security Misconfiguration | CORS policy, security headers, debug endpoints, verbose error messages |
| A06 | Vulnerable Components | Dependency audit (Python, npm), known CVEs in FastAPI/Next.js/SQLAlchemy |
| A07 | Authentication Failures | JWT manipulation, session fixation, MFA bypass, token replay |
| A08 | Software/Data Integrity | FHIR resource tampering, audit log integrity, deployment pipeline security |
| A09 | Logging/Monitoring Failures | PHI in logs, audit trail completeness, alert triggering |
| A10 | SSRF | Internal service access via FHIR reference URLs, MCP tool URL handling |

### 3.2 Healthcare-Specific Tests

| Test | Description |
|------|-------------|
| PHI Exposure | Attempt to extract PHI through error messages, logs, API responses, and cached data |
| Tenant Isolation | Attempt cross-tenant data access via parameter manipulation, JWT tampering, and direct schema queries |
| AI Prompt Injection | Attempt to manipulate Claude via crafted clinical text to extract PHI or bypass safety layer |
| Agent Scope Escalation | Attempt to invoke MCP tools beyond an agent's authorized policy |
| FHIR Search Abuse | Test for excessive data return via `_include`, `_revinclude`, and chained search parameters |
| Break-the-Glass Bypass | Attempt to access emergency access without proper justification logging |
| Audit Trail Evasion | Attempt to access PHI without generating corresponding AuditEvent |

---

## 4. Rules of Engagement

### 4.1 General Rules

- Testing MUST be conducted against the **staging/demo environment only** -- never production
- **No destructive testing:** no data deletion, no table drops, no service disruption
- **No denial-of-service attacks** (application-layer rate limit testing is permitted with controlled volume)
- **Notify MedOS engineering team** at least 24 hours before testing begins
- **Use designated test accounts only** -- do not create unauthorized accounts
- **All test data must be synthetic** -- never use real patient information
- **Stop immediately** if real PHI is discovered in the test environment and notify MedOS team

### 4.2 Communication

- Daily status updates during active testing period
- Immediate notification for any Critical (CVSS >= 9.0) findings
- All findings communicated via encrypted channel (GPG-signed email or secure portal)

### 4.3 Authorized Testing Window

- **Duration:** 5 business days
- **Hours:** 08:00-20:00 EST (to enable real-time monitoring by MedOS team)
- **Emergency stop:** MedOS team can halt testing at any time via designated Slack channel

---

## 5. Test Environment

| Component | Detail |
|-----------|--------|
| Staging URL | `https://staging.medos-platform.com` (placeholder -- will be provided) |
| API Base | `https://staging-api.medos-platform.com/api/v1/` |
| MCP Endpoint | `https://staging-api.medos-platform.com/mcp/` |
| Test Tenant | `test-clinic-pentest` (isolated schema with synthetic data) |
| Test Accounts | Provider, Nurse, Billing Staff, Admin, Security Admin (credentials provided securely at kickoff) |
| Test Agent Credentials | Clinical Scribe agent JWT, Billing agent JWT (provided at kickoff) |
| Synthetic Data | 100 synthetic patients, 500 encounters, 200 claims pre-loaded |

---

## 6. Deliverables

### 6.1 Reports

| Deliverable | Timeline | Format |
|-------------|----------|--------|
| Executive Summary | Within 3 business days of test completion | PDF, 2-3 pages |
| Detailed Technical Report | Within 5 business days of test completion | PDF + machine-readable (JSON/CSV) |
| Remediation Recommendations | Included in technical report | Prioritized by CVSS score |
| Retest Verification Report | Within 3 business days of remediation completion | PDF |

### 6.2 Report Contents

Each finding must include:

- **Title:** Clear description of the vulnerability
- **CVSS v3.1 Score:** With vector string
- **Severity:** Critical / High / Medium / Low / Informational
- **Affected Component:** Specific endpoint, page, or system
- **Description:** Technical explanation of the vulnerability
- **Steps to Reproduce:** Detailed, reproducible instructions
- **Evidence:** Screenshots, HTTP request/response pairs, tool output
- **Business Impact:** PHI exposure potential, compliance implications
- **Remediation Recommendation:** Specific fix with code-level guidance where applicable

---

## 7. Remediation SLA

| Severity | CVSS Score | Response Time | Remediation Deadline | Retest |
|----------|-----------|---------------|---------------------|--------|
| Critical | 9.0 - 10.0 | Immediate (< 4h) | 24 hours | Required before pilot |
| High | 7.0 - 8.9 | Same day | 72 hours | Required before pilot |
| Medium | 4.0 - 6.9 | Within 48h | 2 weeks | Next sprint |
| Low | 0.1 - 3.9 | Acknowledged | Next sprint | Optional |
| Informational | 0.0 | Logged | Backlog | Not required |

**Pilot launch gate:** Zero unresolved Critical or High findings.

---

## 8. Recommended Vendors

The following healthcare-focused penetration testing firms are recommended for evaluation. Final selection should consider healthcare experience, HIPAA familiarity, and availability.

| Vendor | Specialty | Notes |
|--------|-----------|-------|
| Coalfire | Healthcare compliance + pen testing | HITRUST assessor, extensive healthcare portfolio |
| CynergisTek (Clearwater) | Healthcare-exclusive cybersecurity | Deep HIPAA expertise, risk assessment + pen test |
| Praetorian | Advanced application security | Strong API and AI security testing capabilities |
| NetSPI | Continuous pen testing platform | Automated + manual hybrid approach, healthcare clients |

---

## 9. References

- [[HIPAA-Risk-Assessment]] -- Risk register and PHI inventory
- [[HIPAA-Deep-Dive]] -- HIPAA regulatory requirements
- [[Auth-SMART-on-FHIR]] -- Authentication and authorization architecture
- [[System-Architecture-Overview]] -- System architecture reference
- OWASP Testing Guide v4.2
- NIST SP 800-115 -- Technical Guide to Information Security Testing
