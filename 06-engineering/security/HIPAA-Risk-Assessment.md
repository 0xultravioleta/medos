---
type: compliance
date: "2026-02-28"
tags:
  - hipaa
  - security
  - compliance
  - risk-assessment
  - phase-1
status: draft
---

# HIPAA Security Risk Assessment -- MedOS Healthcare OS

> This document constitutes the formal HIPAA Security Risk Assessment for MedOS, required under 45 CFR 164.308(a)(1)(ii)(A). It maps all Protected Health Information (PHI) within the system, traces data flows, identifies risks, and documents mitigations. Cross-reference with [[HIPAA-Deep-Dive]], [[Auth-SMART-on-FHIR]], [[System-Architecture-Overview]], and [[Incident-Response-Playbook]].

---

## 1. Executive Summary

MedOS is an AI-native healthcare operating system targeting mid-size specialty practices (5-30 providers) in Florida. As a Business Associate under HIPAA, MedOS creates, receives, maintains, and transmits electronic Protected Health Information (ePHI) on behalf of Covered Entities (provider practices).

This risk assessment evaluates the security posture of MedOS prior to pilot launch. It covers the complete data lifecycle -- from patient registration through clinical documentation, AI-assisted coding, claims submission, and analytics -- identifying threats, vulnerabilities, and mitigations across all system layers.

**Assessment scope:** MedOS platform (FastAPI backend, Next.js frontend, PostgreSQL database, AI/ML pipeline, MCP agent layer, AWS infrastructure).

**Assessment date:** 2026-02-28

**Assessor:** MedOS Engineering Team

**Next review:** 90 days after pilot launch or upon significant architecture change.

---

## 2. PHI Inventory -- The 18 HIPAA Identifiers

The following table maps each of the 18 HIPAA identifiers to their presence, storage, encryption status, and access controls within MedOS.

| # | HIPAA Identifier | Present in MedOS | Storage Location | Encryption | Access Control |
|---|-----------------|-----------------|-----------------|------------|---------------|
| 1 | Name | Yes | FHIR Patient resource (JSONB), PostgreSQL | AES-256 at rest, TLS 1.3 in transit | RBAC + tenant isolation (schema-per-tenant) |
| 2 | Address (geographic subdivisions smaller than state) | Yes | FHIR Patient.address | AES-256 at rest, TLS 1.3 in transit | RBAC + tenant isolation |
| 3 | Dates (except year) related to individual | Yes | FHIR Patient.birthDate, Encounter dates | AES-256 at rest, TLS 1.3 in transit | RBAC + tenant isolation |
| 4 | Phone numbers | Yes | FHIR Patient.telecom | AES-256 at rest, TLS 1.3 in transit | RBAC + tenant isolation |
| 5 | Fax numbers | Optional | FHIR Patient.telecom | AES-256 at rest, TLS 1.3 in transit | RBAC + tenant isolation |
| 6 | Email addresses | Yes | FHIR Patient.telecom | AES-256 at rest, TLS 1.3 in transit | RBAC + tenant isolation |
| 7 | Social Security Number | Yes | Encrypted field, separate column with field-level encryption | AES-256 + field-level KMS per tenant | Restricted role only (billing admin) |
| 8 | Medical Record Number (MRN) | Yes | FHIR Patient.identifier | AES-256 at rest, TLS 1.3 in transit | RBAC + tenant isolation |
| 9 | Health plan beneficiary number | Yes | FHIR Coverage.identifier | AES-256 at rest, TLS 1.3 in transit | RBAC (billing roles) |
| 10 | Account numbers | Yes | FHIR Account resource | AES-256 at rest + field-level encryption | Restricted role (billing admin) |
| 11 | Certificate/license numbers | Rare | Provider credentials only (not patient) | AES-256 at rest | Admin role only |
| 12 | Vehicle identifiers and serial numbers | No | Not collected | N/A | N/A |
| 13 | Device identifiers and serial numbers | Future (RPM) | Will be in FHIR Device resource | AES-256 at rest | RBAC |
| 14 | Web URLs | No | Not collected as PHI | N/A | N/A |
| 15 | IP addresses | Yes (audit logs) | FHIR AuditEvent, CloudWatch | AES-256 at rest | Security admin only |
| 16 | Biometric identifiers (voice prints) | Yes (audio) | Whisper processing (ephemeral), not persisted as raw audio | In-transit TLS 1.3, ephemeral processing | AI pipeline only, auto-deleted after transcription |
| 17 | Full-face photographs | No | Not collected in MVP | N/A | N/A |
| 18 | Any other unique identifying number | Yes (internal UUIDs) | All FHIR resources | AES-256 at rest | RBAC + tenant isolation |

---

## 3. Data Flow Diagrams

### 3.1 Patient Registration Flow

```
User (Provider/Staff)
    │
    ▼
Next.js Frontend (Server Component -- PHI never in Client Component)
    │  HTTPS/TLS 1.3
    ▼
FastAPI Backend (/api/v1/patients)
    │  Auth0 JWT validation + RBAC check
    │  Tenant context extracted from token
    ▼
SQLAlchemy ORM (tenant schema selected via RLS)
    │
    ▼
PostgreSQL 17 (FHIR Patient resource as JSONB)
    │  AES-256 at rest (AWS KMS per-tenant key)
    │  Field-level encryption for SSN, MRN
    ▼
FHIR AuditEvent logged (who, what, when, from where)
```

### 3.2 Clinical Documentation Flow (AI Scribe)

```
Provider speaks during encounter
    │
    ▼
Audio captured (browser MediaRecorder API)
    │  HTTPS/TLS 1.3 (chunked upload)
    ▼
FastAPI Backend (/api/v1/agent-tasks)
    │  JWT validation + PHI access policy check
    ▼
MCP Gateway (credential injection, safety layer)
    │  Agent identity JWT, PHI policy enforcement
    ▼
Scribe MCP Server (start_session → submit_audio)
    │
    ▼
Whisper v3 (self-hosted GPU, VPC-internal)
    │  Audio transcribed → text (audio deleted after transcription)
    ▼
Claude API (via AWS Bedrock, HIPAA BAA)
    │  Transcript → structured SOAP note
    │  Confidence score attached to each section
    ▼
Confidence Router
    │  >= 0.95: auto-approve
    │  >= 0.85: pass to provider review
    │  < 0.85: mandatory human review
    ▼
FHIR DocumentReference (SOAP note as JSONB in PostgreSQL)
    │  AES-256 at rest, tenant-isolated schema
    ▼
FHIR AuditEvent logged (agent ID, action, resource, timestamp)
```

### 3.3 Claims Pipeline Flow

```
Encounter finalized (signed SOAP note)
    │
    ▼
AI Coding Agent (LangGraph)
    │  Claude suggests ICD-10 + CPT codes
    │  Confidence score per code
    ▼
Billing Staff Review (codes with confidence < 0.85 flagged)
    │
    ▼
X12 837P Generator
    │  Encounter → EDI 837P Professional Claim
    ▼
Clearinghouse (Availity)
    │  SFTP/AS2, encrypted in transit
    ▼
Payer adjudication
    │
    ▼
X12 835 ERA (Electronic Remittance Advice)
    │  Parsed by MedOS
    ▼
Payment Posting → FHIR ExplanationOfBenefit
    │
    ▼
Denial Management (if applicable)
    │  AI-assisted appeal generation
    ▼
FHIR AuditEvent logged at each step
```

### 3.4 MCP Agent Pipeline Flow

```
Incoming Request (API or internal trigger)
    │
    ▼
MCP Gateway (/mcp/messages)
    │  1. Agent identity JWT validation
    │  2. PHI access policy check (agent type → allowed resources)
    │  3. Credential injection (secrets from AWS Secrets Manager)
    │  4. Safety layer evaluation
    │     - Block: dangerous/prohibited content
    │     - Warn: suspicious patterns
    │     - Review: low confidence (< 0.85)
    │     - Sanitize: strip PHI for non-clinical agents
    ▼
Tool Registry (18 registered MCP tools)
    │  Route to appropriate MCP server
    ▼
MCP Server (FHIR or Scribe)
    │  Execute tool with tenant-scoped database connection
    ▼
Response + FHIR AuditEvent
    │  Every tool invocation logged with:
    │  - Agent ID, tool name, resource accessed
    │  - Timestamp, tenant ID, outcome
    ▼
Response returned through Gateway (PHI sanitized if needed)
```

---

## 4. Risk Register

| Risk ID | Category | Description | Likelihood (1-5) | Impact (1-5) | Score | Mitigation | Status |
|---------|----------|-------------|:-:|:-:|:-:|------------|--------|
| R-001 | Access Control | Unauthorized access to PHI via compromised provider credentials | 3 | 5 | 15 | MFA required for all users (Auth0), session timeout 15 min idle, SMART on FHIR scopes limit access to relevant patient context | Implemented |
| R-002 | Data Breach | PHI exposure through SQL injection in FastAPI endpoints | 2 | 5 | 10 | SQLAlchemy ORM with parameterized queries, Pydantic v2 input validation, WAF rules, pen test scope includes SQLi testing | Implemented |
| R-003 | AI/ML Risk | AI hallucination generates incorrect clinical content attributed to provider | 4 | 4 | 16 | Confidence scoring on all AI outputs, mandatory human review below 0.85 threshold, provider sign-off required before finalization | Implemented |
| R-004 | AI/ML Risk | PHI leakage through Claude API prompts/responses | 2 | 5 | 10 | Claude via AWS Bedrock with HIPAA BAA, zero-data-training policy, PHI never logged in prompts, MCP safety layer sanitizes for non-clinical agents | Implemented |
| R-005 | Infrastructure | Database compromise exposing multi-tenant PHI | 2 | 5 | 10 | Schema-per-tenant isolation with Row-Level Security (RLS), per-tenant KMS encryption keys, network isolation (private subnets, no public DB access) | Implemented |
| R-006 | Access Control | Privilege escalation -- staff accessing PHI beyond their role | 3 | 4 | 12 | RBAC with SMART on FHIR scopes, tenant isolation at schema level, attribute-based access control for sensitive fields (SSN), quarterly access reviews | Implemented |
| R-007 | Data Breach | PHI exposure in application logs or error messages | 3 | 4 | 12 | PHI scrubbing middleware in FastAPI, structured logging with PHI redaction, CloudWatch log encryption, code policy: never log any of 18 HIPAA identifiers | Implemented |
| R-008 | Insider Threat | Malicious or negligent employee accessing PHI without authorization | 2 | 5 | 10 | FHIR AuditEvent for every PHI access, break-the-glass logging with mandatory justification, anomaly detection on access patterns, background checks for staff with PHI access | Partial |
| R-009 | Vendor/Third-Party | Clearinghouse data breach exposing claims data | 2 | 4 | 8 | BAA with clearinghouse (Availity), encrypted transmission (SFTP/AS2), minimal PHI in claims (only what is required), vendor security assessment | Pending |
| R-010 | Infrastructure | AWS region outage causing system unavailability | 2 | 3 | 6 | Multi-AZ deployment, automated failover for RDS, S3 cross-region replication for backups, RTO 4h/RPO 1h targets | Planned |
| R-011 | Backup/DR | Data loss due to failed or corrupted backups | 2 | 5 | 10 | Automated daily PostgreSQL snapshots (AWS RDS), point-in-time recovery enabled (35-day retention), monthly backup restoration tests, encrypted backups with separate KMS key | Implemented |
| R-012 | Access Control | Stale user accounts retaining PHI access after role change or departure | 3 | 3 | 9 | Auth0 lifecycle management, automated deprovisioning webhooks, quarterly access reviews, 90-day inactive account suspension | Planned |
| R-013 | AI/ML Risk | MCP agent exceeding its authorized scope (accessing tools/resources beyond its policy) | 3 | 4 | 12 | Agent identity JWT with scoped permissions, PHI access policies per agent type in MCP Gateway, safety layer blocks unauthorized tool invocations, all tool calls audited | Implemented |
| R-014 | Data Breach | Cross-tenant data leakage through shared infrastructure | 2 | 5 | 10 | Schema-per-tenant with RLS enforced at database level, tenant context validated on every request, separate KMS keys per tenant, integration tests verify isolation | Implemented |
| R-015 | Physical | Unauthorized physical access to workstations displaying PHI | 3 | 3 | 9 | Screen lock policy (5 min inactivity), HIPAA training requirement, Next.js Server Components ensure PHI processed server-side, clean desk policy in BAA | Administrative |
| R-016 | Infrastructure | TLS misconfiguration exposing PHI in transit | 1 | 5 | 5 | TLS 1.3 enforced (no fallback to 1.2 or lower), AWS ALB handles termination with managed certificates, automated certificate rotation via ACM, security headers enforced | Implemented |
| R-017 | AI/ML Risk | Audio recording persisted beyond processing window creating PHI retention risk | 2 | 4 | 8 | Audio auto-deleted after Whisper transcription completes, no raw audio stored in database, ephemeral processing in memory, retention policy enforced in Scribe MCP server | Implemented |
| R-018 | Vendor/Third-Party | Claude API (Bedrock) service disruption affecting clinical workflows | 3 | 3 | 9 | Graceful degradation (providers can document manually), queue-based retry for non-urgent AI tasks, SLA monitoring via CloudWatch, fallback to direct documentation entry | Planned |

**Risk Scoring:** Likelihood x Impact. Critical >= 15, High 10-14, Medium 5-9, Low 1-4.

---

## 5. Technical Controls Summary

### 5.1 Encryption

| Layer | Standard | Implementation |
|-------|----------|---------------|
| At rest | AES-256 | AWS RDS encryption (KMS-managed keys, per-tenant keys) |
| In transit | TLS 1.3 | AWS ALB termination, internal service mesh TLS |
| Field-level | AES-256-GCM | SSN, MRN encrypted with tenant-specific KMS key via SQLAlchemy TypeDecorator |
| Backups | AES-256 | RDS snapshots encrypted with separate KMS key |
| Logs | AES-256 | CloudWatch log group encryption |

### 5.2 Access Control

- **Authentication:** Auth0 with MFA required, SMART on FHIR launch context
- **Authorization:** RBAC (provider, nurse, billing, admin, security) + ABAC for sensitive fields
- **Tenant isolation:** Schema-per-tenant with PostgreSQL RLS, tenant extracted from JWT on every request
- **Session management:** 15-minute idle timeout, 8-hour absolute timeout, secure cookie flags
- **Break-the-glass:** Emergency access with mandatory justification, enhanced logging, auto-alert to security admin

### 5.3 Audit Logging

- **Standard:** FHIR AuditEvent (aligned with IHE Basic Audit Log Patterns)
- **Scope:** Every PHI access, modification, and deletion logged
- **Fields captured:** who (user/agent ID), what (resource type and ID), when (timestamp), where (IP, user agent), why (purpose of use)
- **Retention:** 7 years (HIPAA requirement: 6 years minimum)
- **Integrity:** Append-only log, CloudWatch Logs with tamper detection

### 5.4 MCP Security Pipeline

- **Agent identity:** JWT-based authentication for each AI agent
- **PHI access policies:** Per-agent-type resource access rules enforced at MCP Gateway
- **Credential injection:** Secrets injected at runtime from AWS Secrets Manager (never in code or config)
- **Safety layer:** Four-tier evaluation (Block, Warn, Review, Sanitize) applied to every agent request
- **Tool-level audit:** Every MCP tool invocation creates a FHIR AuditEvent

---

## 6. Administrative Controls

### 6.1 HIPAA Training

- All personnel with PHI access must complete HIPAA training before access is granted
- Annual refresher training required
- Training completion tracked and auditable
- Training covers: PHI identification, minimum necessary principle, breach reporting, device security

### 6.2 Access Review Schedule

| Review | Frequency | Owner | Scope |
|--------|-----------|-------|-------|
| User access audit | Quarterly | Security Admin | All active accounts, role assignments |
| Privileged access review | Monthly | CTO | Admin, security, and break-the-glass accounts |
| Vendor access review | Semi-annually | Security Admin | Third-party integrations and BAAs |
| Agent permissions review | Quarterly | Engineering Lead | MCP agent policies and tool access |

### 6.3 Policies and Procedures

- Acceptable Use Policy
- Password and MFA Policy
- Incident Response Policy (see [[Incident-Response-Playbook]])
- Data Retention and Disposal Policy
- Bring Your Own Device (BYOD) Policy
- Vendor Management Policy

---

## 7. References

- [[HIPAA-Deep-Dive]] -- Full HIPAA regulatory reference
- [[Auth-SMART-on-FHIR]] -- Authentication and authorization architecture
- [[System-Architecture-Overview]] -- Complete system architecture
- [[Incident-Response-Playbook]] -- Incident classification and response procedures
- [[Penetration-Test-Scope]] -- Pre-pilot security testing plan
- [[SOC2-HITRUST-Roadmap]] -- Compliance certification timeline
- 45 CFR 164.308(a)(1)(ii)(A) -- HIPAA Security Rule Risk Analysis requirement
- NIST SP 800-66 Rev. 2 -- Implementing the HIPAA Security Rule
