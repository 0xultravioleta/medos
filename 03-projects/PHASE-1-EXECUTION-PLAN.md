---
type: execution-plan
date: "2026-02-28"
status: active
tags:
  - project
  - execution-plan
  - phase-1
  - roadmap
priority: 1
---

# Phase 1 Execution Plan: Foundation to Pilot-Ready in 90 Days

> **Duration:** 90 days (2026-03-01 to 2026-05-29)
> **Team:** 2 people + AI tools (Claude Code + Cursor = 5-10x multiplier)
> **Budget:** ~$15,000-25,000 (AWS + tools + services)
> **Goal:** Functional platform that 3-5 pilot practices can use for AI documentation, coding assistance, and basic revenue cycle

---

## Overview

This document is the operational bible for Phase 1 of Healthcare OS. It breaks 90 days into seven sprints, each with day-level task granularity, explicit ownership, testable acceptance criteria, and hard dependencies. Every task is sized for completion in 1-8 hours by an engineer with AI tooling. No task is vague. No day is unplanned.

Phase 1 delivers six capabilities in sequence:
1. HIPAA-compliant AWS infrastructure (Sprint 0)
2. Authentication and multi-tenant identity (Sprint 0)
3. FHIR-native data layer with patient matching (Sprint 1)
4. AI-powered clinical documentation from audio (Sprint 2)
5. Revenue cycle MVP -- eligibility, coding, claims (Sprints 3-4)
6. Pilot-ready security, onboarding, and training (Sprints 5-6)

### Team Allocation

| Person | Code | Primary Focus | Secondary Focus |
|--------|------|---------------|-----------------|
| Person A | A | Infrastructure, backend, DevOps, AI pipeline | Security, compliance automation |
| Person B | B | Clinical domain, FHIR modeling, revenue cycle, business logic | Frontend, integrations, pilot relationships |

Both are technical. Both write code. The split is about domain ownership, not skill division. On any given day either person can pick up the other's tasks if blocked.

### Key Milestones

| ID | Milestone | Date | Sprint | Hard Criteria |
|----|-----------|------|--------|---------------|
| M1 | Infrastructure operational | 2026-03-14 | S0 | Terraform deploys full stack, CI/CD pushes to ECS, RDS accessible |
| M2 | First FHIR resource stored and retrieved | 2026-03-28 | S1 | Patient CRUD via API with tenant isolation verified |
| M3 | First AI-generated clinical note | 2026-04-11 | S2 | Audio in, structured SOAP note out, provider review UI functional |
| M4 | First claim generated | 2026-04-25 | S3 | Eligibility check + AI coding + CMS-1500 claim from encounter |
| M5 | Prior auth submitted electronically | 2026-05-09 | S4 | X12 278 request built and transmitted to test payer |
| M6 | Security audit passed | 2026-05-23 | S5 | Pen test complete, HIPAA checklist 100%, SOC 2 prep started |
| M7 | First pilot practice onboarded | 2026-05-29 | S6 | Practice live on demo environment with real workflows |

### Master Dependencies Chart

```
EPIC-001 (AWS Infra) ─────────────────────────┐
  VPC, RDS, ECS, KMS, CI/CD                    │
                                                ▼
EPIC-002 (Auth & Identity) ──────> EPIC-003 (FHIR Data Layer)
  OAuth2, RBAC, Sessions, Audit      Patient, Encounter, FHIR CRUD
                                                │
                              ┌─────────────────┼─────────────────┐
                              ▼                 ▼                 ▼
                    EPIC-004 (AI Docs)  EPIC-005 (Revenue)  EPIC-006 (Pilot)
                    Whisper, Claude,    Eligibility, Claims, Security, Onboard,
                    Note Generation     Prior Auth, Denials  Training, Demo
```

---

## Sprint 0: Foundation

**Sprint Goal:** Stand up HIPAA-compliant AWS infrastructure, CI/CD pipeline, FastAPI skeleton, auth system, and database with working end-to-end deployment.

**Sprint Dates:** 2026-03-01 (Sun) to 2026-03-14 (Sat) -- 10 working days

**References:** [[EPIC-001-aws-infrastructure-foundation]], [[EPIC-002-auth-identity-system]], [[ADR-004-fastapi-backend-architecture]], [[HIPAA-Deep-Dive]]

### Day-by-Day Breakdown

#### Day 1 (Mon, Mar 2) -- AWS Account & Repo Setup

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T01 | Create AWS Organization + management account, enable SCPs for HIPAA guardrails | A | 3h | -- | Organization created, SCPs deny non-approved regions and public S3 | pending |
| S0-T02 | Create workload accounts: medos-dev, medos-staging, medos-prod via Organizations | A | 2h | S0-T01 | Three accounts accessible, SSO configured for both team members | pending |
| S0-T03 | Create GitHub monorepo with branch protection, CODEOWNERS, PR templates | B | 2h | -- | Repo live, main branch protected, require 1 review, status checks | pending |
| S0-T04 | Initialize Terraform backend (S3 state bucket + DynamoDB lock table) in mgmt account | A | 2h | S0-T01 | `terraform init` succeeds, state stored in S3 with encryption | pending |
| S0-T05 | Create `.env.example` with all anticipated variables (no values), document in README | B | 1h | S0-T03 | File lists every variable needed for local dev and deployment | pending |

#### Day 2 (Tue, Mar 3) -- Terraform Module Structure & KMS

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T06 | Create Terraform module directory structure (networking, database, compute, secrets, monitoring, iam) with environment var files | A | 3h | S0-T04 | `terraform plan` for dev runs with zero resources, provider pinned | pending |
| S0-T07 | Implement Terraform tagging strategy module (enforce Environment, Project, CostCenter, HIPAA tags on all resources) | A | 2h | S0-T06 | Every resource gets default tags via provider block | pending |
| S0-T08 | Create KMS module: infrastructure keys (RDS, S3, logs, secrets) + per-tenant key template with auto-rotation | A | 3h | S0-T06 | All infra keys created, rotation enabled, key policies least-privilege | pending |
| S0-T09 | Auth provider evaluation: build comparison matrix for Auth0, Clerk, Keycloak against SMART on FHIR, BAA, multi-tenant, MFA, cost | B | 4h | -- | ADR drafted with clear recommendation and cost projection | pending |

**Decision Point:** Choose auth provider (Auth0 vs Clerk vs Keycloak). Recommendation: Auth0 Enterprise (BAA available, SMART on FHIR support, fastest to integrate). Decision must be made by EOD Day 2.

#### Day 3 (Wed, Mar 4) -- VPC & S3

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T10 | Create Terraform VPC module: 3-AZ public/private/database subnets, single NAT (dev), VPC Flow Logs to S3 | A | 4h | S0-T06 | Database subnets have no internet route, flow logs capturing | pending |
| S0-T11 | Configure VPC endpoints: S3 (gateway), ECR, Secrets Manager, CloudWatch Logs, KMS, STS (interface) | A | 3h | S0-T10 | ECR pulls work without NAT, verified by test pull | pending |
| S0-T12 | Create S3 module + buckets: logs, backups, attachments, audit, terraform-state with KMS encryption, versioning, lifecycle policies | A | 2h | S0-T08 | All buckets created, public access blocked, lifecycle policies applied | pending |
| S0-T13 | Sign BAA with selected auth provider, create tenant in provider, configure SMART on FHIR app registration | B | 3h | S0-T09 | BAA signed and stored, test OAuth2 flow returns tokens | pending |

#### Day 4 (Thu, Mar 5) -- RDS, ECS, Security Services

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T14 | Create Terraform RDS module: PostgreSQL 17, pgvector, gp3 storage, KMS encryption, parameter group (pgaudit, SSL), database subnet placement | A | 4h | S0-T10, S0-T08 | PostgreSQL 17 running, pgvector enabled, connection via SSM verified | pending |
| S0-T15 | Create Terraform ECS module: Fargate cluster, ALB with HTTPS, ACM cert, WAF basic rules, Cloud Map service discovery, ECR repos | A | 4h | S0-T10 | ALB serving HTTPS, ECR accepts pushes, sample task runs | pending |
| S0-T16 | Design RBAC + ABAC permission model: role hierarchy, permission matrix per FHIR resource, ABAC attributes (tenant, provider, department, location) | B | 4h | S0-T13 | Permission model documented, covers all MVP FHIR resources, edge cases listed | pending |
| S0-T17 | Enable CloudTrail (org-wide, multi-region), GuardDuty, AWS Config with HIPAA conformance pack, Security Hub | A | 3h | S0-T01, S0-T12 | Security Hub HIPAA score > 80%, CloudTrail logging verified | pending |

#### Day 5 (Fri, Mar 6) -- FastAPI Skeleton & CI/CD

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T18 | Create FastAPI project scaffold per [[ADR-004-fastapi-backend-architecture]]: src layout, routers, services, repositories, middleware, config | A | 3h | S0-T03 | Project runs locally, `/health` returns 200 with version info | pending |
| S0-T19 | Create Dockerfile (multi-stage, non-root user) + docker-compose.yml for local dev (API + PostgreSQL + Redis) | A | 2h | S0-T18 | `docker-compose up` brings full stack locally, health check passes | pending |
| S0-T20 | Set up Alembic for database migrations with schema-per-tenant support, create initial migration (shared tables + sample tenant schema) | A | 3h | S0-T18 | `alembic upgrade head` creates tables, `alembic downgrade` rolls back cleanly | pending |
| S0-T21 | Create GitHub Actions CI pipeline: lint (ruff), test (pytest), security scan (trivy, semgrep), build Docker image | B | 3h | S0-T18, S0-T03 | PR triggers all checks, merge to main builds image successfully | pending |
| S0-T22 | Create GitHub Actions deploy pipeline: build, push to ECR, update ECS task def, force new deployment, health check | B | 3h | S0-T15, S0-T21 | Merge to main deploys to dev ECS within 5 minutes, health check passes | pending |
| S0-T23 | Create Terraform CI/CD pipeline: plan on PR, apply on merge, OIDC auth (no long-lived AWS keys) | B | 2h | S0-T06, S0-T03 | Terraform plan comments on PR, apply runs on merge | pending |

#### Day 6 (Mon, Mar 9) -- Auth Implementation

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T24 | Implement SMART on FHIR OAuth2 flow: authorization endpoint, token exchange, JWKS validation, token refresh | A | 4h | S0-T13, S0-T18 | Full OAuth2 flow works, tokens contain required claims | pending |
| S0-T25 | Implement JWT validation middleware: signature verification (RS256), tenant extraction, PostgreSQL search_path routing per [[ADR-002-multi-tenancy-schema-per-tenant]] | A | 3h | S0-T24, S0-T20 | Requests to wrong tenant return 403, search_path set correctly per request | pending |
| S0-T26 | Create roles in auth provider matching RBAC hierarchy, define SMART on FHIR scopes, map roles to default scope sets | B | 3h | S0-T16, S0-T13 | All 7 roles created, scopes enforced, scope validation rejects insufficient access | pending |
| S0-T27 | Implement MFA enforcement policies: required for clinical roles, TOTP + WebAuthn, challenge on new device/IP | B | 2h | S0-T13 | Clinical users cannot authenticate without MFA, 2+ methods available | pending |

#### Day 7 (Tue, Mar 10) -- Session Management & Models

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T28 | Deploy ElastiCache Redis (encryption at rest + in transit, private subnet) via Terraform module | A | 2h | S0-T10, S0-T08 | Redis accessible from private subnets, TLS enforced | pending |
| S0-T29 | Implement server-side session management: create/validate/revoke sessions in Redis, 30-min idle timeout, 3 concurrent session limit | A | 3h | S0-T28, S0-T24 | Sessions expire on idle, concurrent limit enforced, admin can revoke | pending |
| S0-T30 | Create User model + Tenant model in SQLAlchemy: user_id, tenant_id, roles, email, MFA status, created/updated timestamps | B | 2h | S0-T20 | Models migrate cleanly, CRUD operations pass unit tests | pending |
| S0-T31 | Implement RBAC middleware: permission check decorator, tenant isolation enforcement, ABAC attribute evaluation | B | 3h | S0-T25, S0-T16 | Decorator rejects unauthorized access, tenant isolation verified | pending |

#### Day 8 (Wed, Mar 11) -- FHIR Base Tables & Audit

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T32 | Create FHIR resource base tables per [[ADR-001-fhir-native-data-model]]: fhir_resources table with JSONB, resource_type, tenant_id, version, GIN indexes | A | 4h | S0-T20, S0-T14 | Tables created in tenant schema, GIN index on JSONB, query by resource_type works | pending |
| S0-T33 | Implement schema-per-tenant provisioning per [[ADR-002-multi-tenancy-schema-per-tenant]]: create schema on tenant onboarding, run migrations per schema | A | 3h | S0-T32 | New tenant gets isolated schema, tables created, cross-schema query impossible from app | pending |
| S0-T34 | Implement auth audit trail: FHIR AuditEvent table, async event emitter for login/logout/permission denied/session events | B | 4h | S0-T30, S0-T14 | Every auth event generates AuditEvent, audit table is append-only | pending |
| S0-T35 | Implement API key management: create (returns plaintext once), list (prefix only), revoke, rate limit per key (1000 req/min) | B | 3h | S0-T31 | API key auth works for integration endpoints, revoked keys rejected | pending |

#### Day 9 (Thu, Mar 12) -- FHIR Patient CRUD & Search

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T36 | Implement FHIR Patient CRUD endpoints: POST (create), GET by ID (read), PUT (update with version), DELETE (soft delete) | A | 4h | S0-T32, S0-T25 | Patient CRUD works, version incrementing on update, FHIR-compliant responses | pending |
| S0-T37 | Implement basic FHIR search parameters for Patient: name, birthdate, identifier, gender, phone, email (GET /Patient?name=...) | A | 3h | S0-T36 | Search returns matching patients, supports AND chaining, pagination via Bundle | pending |
| S0-T38 | Implement break-the-glass emergency access flow: justification form, temporary elevated access (2h window), auto-revoke, admin notification | B | 4h | S0-T31, S0-T29 | Provider accesses unassigned patient with justification, all actions flagged in audit, access auto-revokes | pending |
| S0-T39 | Set up AWS Budgets: per-environment monthly limits, per-service budgets, anomaly detection, alert thresholds at 50/80/100/120% | B | 2h | S0-T01 | Budget alerts configured, test notification received | pending |

#### Day 10 (Fri, Mar 13) -- Integration Testing & Sprint Review

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S0-T40 | Sprint 0 integration testing: deploy full stack to dev ECS, run auth flow end-to-end, verify tenant isolation (2 test tenants), FHIR Patient CRUD via API | A | 4h | ALL | Full flow works: login -> get token -> create patient -> read patient -> verify isolation | pending |
| S0-T41 | Cross-tenant security tests: provider in tenant-alpha cannot read tenant-beta patients, expired tokens rejected, revoked sessions rejected, JWT tampering rejected | A | 3h | S0-T40 | All 12+ security test scenarios pass (see EPIC-002 T10 matrix) | pending |
| S0-T42 | Create environment separation Terraform configs: dev/staging/prod variable files with appropriate sizing (see EPIC-001 T10 table) | B | 2h | S0-T06 | Each environment has independent state, no cross-environment network routes | pending |
| S0-T43 | Sprint 0 review: demo deployed system, document what shipped vs planned, retrospective on velocity, Sprint 1 planning | BOTH | 2h | S0-T40 | Review notes documented, Sprint 1 backlog confirmed | pending |

### Sprint 0 Deliverables
- Fully deployed HIPAA-compliant AWS infrastructure via Terraform (zero ClickOps)
- Working CI/CD pipeline: PR checks, auto-deploy to dev on merge
- FastAPI running on ECS Fargate with health endpoint
- Auth system: OAuth2 flow, JWT with tenant context, RBAC middleware, MFA
- PostgreSQL 17 with pgvector, schema-per-tenant, FHIR resource tables
- FHIR Patient CRUD + basic search
- Audit trail for all auth events
- Security services: CloudTrail, GuardDuty, Config, Security Hub

### Sprint 0 Risks
| Risk | Mitigation |
|------|------------|
| AWS Organizations account provisioning delay (24-48h) | Start T01 first thing Day 1; parallel-track auth eval (T09) which needs no AWS |
| Auth provider BAA takes >48h to process | Email BAA request on Day 1 even before evaluation complete; have Keycloak self-hosted fallback |
| pgvector unavailable on RDS 17 in us-east-1 | Verify extension list before selecting region; Aurora PostgreSQL as fallback |
| 10 days too aggressive for all tasks | Tasks prioritized: infra (Days 1-5) then auth (Days 6-7) then FHIR (Days 8-9) -- each layer independently valuable |

### Sprint 0 Definition of Done
- `terraform plan` shows no drift across all environments
- CI/CD deploys automatically on merge to main
- Auth flow: login, MFA, get token, access FHIR endpoint, tenant isolation verified
- Security Hub HIPAA compliance score > 80%
- All secrets in Secrets Manager, none in code

---

## Sprint 1: Core Data Layer

**Sprint Goal:** Build the FHIR-native data layer with Encounter, Observation, and Condition resources, patient probabilistic matching, and the event bus for downstream AI and billing pipelines.

**Sprint Dates:** 2026-03-15 to 2026-03-28 (10 working days)

**References:** [[EPIC-003-fhir-data-layer]], [[ADR-001-fhir-native-data-model]], [[FHIR-R4-Deep-Dive]]

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S1-T01 | Implement FHIR Encounter resource: CRUD, search by patient/date/status/type, link to Patient | A | 4h | S0-T36 | Encounter CRUD works, search returns correct results, linked to Patient | pending |
| S1-T02 | Implement FHIR Observation resource: CRUD, search by patient/code/date, LOINC code validation | A | 4h | S1-T01 | Observation stored with LOINC codes, searchable by patient and code | pending |
| S1-T03 | Implement FHIR Condition resource: CRUD, search by patient/code/status, ICD-10 code validation | A | 4h | S1-T01 | Condition stored with ICD-10 codes, active/resolved status tracking | pending |
| S1-T04 | Implement FHIR Practitioner + PractitionerRole resources: CRUD, link to Organization/tenant | B | 4h | S0-T36 | Practitioner linked to tenant, searchable by specialty/name | pending |
| S1-T05 | Implement FHIR Organization resource: CRUD, tenant-to-org mapping, NPI validation | B | 3h | S0-T36 | Organization stores NPI, address, taxonomy codes | pending |
| S1-T06 | Build patient probabilistic matching engine: name (Jaro-Winkler), DOB (exact), SSN-last4 (exact), phone (normalized), configurable threshold (0.0-1.0) | A | 6h | S0-T36 | Match score > 0.85 links patients, < 0.6 creates new, 0.6-0.85 flags for review | pending |
| S1-T07 | Implement FHIR Bundle support: transaction bundles (atomic multi-resource create/update), batch bundles | A | 4h | S1-T01 | Transaction bundle creates Patient + Encounter + Observations atomically or rolls back | pending |
| S1-T08 | Deploy EventBridge event bus with Terraform: create bus, define event schemas for FHIR resource CRUD events | B | 3h | S0-T06 | Events published on resource create/update/delete, schema validated | pending |
| S1-T09 | Implement FHIR resource event publisher: emit events to EventBridge on every CRUD operation with resource type, action, tenant_id | A | 3h | S1-T08, S1-T01 | Every FHIR CRUD emits event, events visible in EventBridge console | pending |
| S1-T10 | Implement FHIR Consent resource: patient consent records, machine-executable policy, consent check middleware | B | 4h | S0-T36, S0-T31 | Consent stored per patient, API checks consent before returning PHI | pending |
| S1-T11 | Implement FHIR resource versioning: version history, vread (read specific version), _history endpoint | A | 3h | S0-T32 | Every update increments version, previous versions retrievable via _history | pending |
| S1-T12 | Create FHIR validation layer: validate all incoming resources against FHIR R4 profiles, reject malformed resources with OperationOutcome | B | 4h | S1-T01 | Invalid FHIR resources rejected with descriptive OperationOutcome errors | pending |
| S1-T13 | Implement pgvector embeddings for clinical text: generate embeddings on Observation.text and Condition.text, store in vector column | A | 4h | S0-T14, S1-T02 | Semantic search "chest pain" returns relevant Conditions and Observations | pending |
| S1-T14 | Build FHIR Capability Statement endpoint: /metadata returns server capabilities, supported resources, search params, auth info | B | 2h | S1-T01 | GET /metadata returns valid CapabilityStatement, passes FHIR validator | pending |
| S1-T15 | Sprint 1 integration test: create full patient encounter workflow (Patient -> Encounter -> Observations -> Conditions), verify event bus, verify search | BOTH | 4h | ALL | Full workflow works end-to-end, events flowing, search returns correct data | pending |

### Sprint 1 Deliverables
- 7 FHIR resource types with full CRUD and search (Patient, Encounter, Observation, Condition, Practitioner, PractitionerRole, Organization)
- Patient probabilistic matching engine
- EventBridge event bus emitting FHIR lifecycle events
- FHIR validation, versioning, and Bundle support
- pgvector semantic search on clinical text
- FHIR Capability Statement endpoint

### Sprint 1 Risks
| Risk | Mitigation |
|------|------------|
| Patient matching false positives in production | Configurable threshold, manual review queue for 0.6-0.85 range |
| FHIR validation library performance | Cache compiled profiles, validate async for batch imports |
| EventBridge event ordering | Use event IDs with timestamps, design consumers to be idempotent |

### Sprint 1 Definition of Done
- Full patient encounter workflow works via API
- Cross-tenant isolation verified for all 7 resource types
- Event bus emitting events on all FHIR operations
- Patient matching returns correct scores for test dataset

---

## Sprint 2: AI Clinical Documentation MVP

**Sprint Goal:** Ship the ambient AI documentation pipeline: audio capture to structured clinical note, with provider review UI.

**Sprint Dates:** 2026-03-29 to 2026-04-11 (10 working days)

**Status:** done

**References:** [[EPIC-004-ai-clinical-documentation]], [[EPIC-007-mcp-sdk-refactoring]], [[ADR-003-ai-agent-framework]], [[ADR-004-fastapi-backend-architecture]], [[ADR-005-mcp-sdk-integration]]

**Sprint 2 Actual Delivery:**
- 174 tests passing, ruff clean
- 32 MCP tools registered (12 FHIR + 6 Scribe + 8 Billing + 6 Scheduling)
- 3 LangGraph agents operational (Clinical Scribe + Prior Auth + Denial Management)
- HIPAAFastMCP subclass with @hipaa_tool decorators
- Approval workflow API with queue management
- Frontend `/docs` page with MCP tool catalog
- ADR-005 created, EPIC-007 completed

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S2-T01 | Deploy Whisper v3 on ECS GPU task (g5.xlarge): Dockerfile, model download, health check, auto-scaling to zero when idle | A | 6h | S0-T15 | Whisper endpoint accepts audio file, returns transcript in < 2x real-time | done |
| S2-T02 | Build audio upload API: accept WAV/MP3/M4A (up to 30 min), store in S3 per-tenant prefix, return job ID | A | 3h | S0-T12, S0-T25 | Audio uploaded, stored encrypted in S3, job ID returned for polling | done |
| S2-T03 | Build transcription pipeline: EventBridge trigger on audio upload -> Whisper task -> store transcript -> emit transcript-ready event | A | 4h | S2-T01, S2-T02, S1-T08 | Audio upload triggers transcription, transcript stored, event emitted | done |
| S2-T04 | Build Claude clinical NLP pipeline: transcript -> structured extraction (chief complaint, HPI, ROS, exam findings, assessment, plan) using Claude API with HIPAA BAA | A | 6h | S2-T03 | Claude extracts structured clinical data from transcript with > 90% accuracy on test cases | done |
| S2-T05 | Build SOAP note generator: structured extraction -> formatted SOAP note (Subjective, Objective, Assessment, Plan) with FHIR DocumentReference | A | 4h | S2-T04 | SOAP note generated, stored as FHIR DocumentReference linked to Encounter | done |
| S2-T06 | Implement confidence scoring for AI extractions: Claude returns confidence per section, flag low-confidence sections for human review | A | 3h | S2-T04 | Each section has confidence score, sections < 0.85 highlighted in UI | done |
| S2-T07 | Implement LangGraph agent for clinical documentation: state machine with audio -> transcript -> extraction -> note -> review states | A | 4h | S2-T04, S2-T05 | Agent orchestrates full pipeline, handles failures, retries, state visible | done |
| S2-T08 | Build Next.js provider workspace: encounter list, start new encounter, recording controls (start/stop/pause) | B | 6h | S2-T02 | Provider can start encounter, record audio, see encounter list | done |
| S2-T09 | Build note review UI: display generated SOAP note, inline editing per section, accept/reject/modify controls, sign-off button | B | 6h | S2-T05 | Provider sees generated note, edits inline, signs off, note saved as final | done |
| S2-T10 | Build real-time transcription status: WebSocket connection showing transcription progress, section extraction progress | B | 3h | S2-T03 | Provider sees live progress: "Transcribing... Extracting... Generating note..." | done |
| S2-T11 | Implement Langfuse integration for LLM observability: trace every Claude call, log tokens/latency/cost per encounter | A | 2h | S2-T04 | Every Claude call traced in Langfuse dashboard with cost attribution | done |
| S2-T12 | Implement AI-generated ICD-10 + CPT suggestions from clinical note: Claude analyzes SOAP note, suggests codes with confidence scores | B | 4h | S2-T05 | Code suggestions displayed alongside note, provider can accept/modify/reject | done |
| S2-T13 | Build encounter summary dashboard: list of today's encounters, AI note status (pending/review/signed), time saved metric | B | 3h | S2-T08 | Provider sees daily encounter summary with status and estimated time savings | done |
| S2-T14 | Sprint 2 end-to-end test: record sample encounter audio, run through full pipeline, review and sign note, verify FHIR resources created | BOTH | 4h | ALL | Full demo works: audio -> note -> review -> signed -> stored in FHIR | done |

### Sprint 2 Deliverables
- Working Whisper transcription endpoint on ECS GPU
- Claude-powered clinical note generation (SOAP format)
- Provider workspace UI with recording and note review
- ICD-10/CPT code suggestions from AI
- LLM observability via Langfuse

### Sprint 2 Risks
| Risk | Mitigation |
|------|------------|
| GPU instance costs (g5.xlarge = ~$1.00/hr) | Auto-scale to zero, batch transcriptions, consider Whisper API as fallback |
| Claude hallucination in clinical context | Confidence scoring, mandatory human review, multi-model validation for high-risk content |
| Audio quality in real clinical settings | Test with noisy samples, implement noise reduction preprocessing |

### Sprint 2 Definition of Done
- Demo: record 5-minute clinical encounter, get SOAP note in < 3 minutes
- Provider can review, edit, and sign off on AI-generated note
- All notes stored as FHIR DocumentReference with provenance
- ICD-10/CPT suggestions generated with confidence scores

---

## Sprint 3: Revenue Cycle v1

**Sprint Goal:** Build eligibility verification, AI coding engine, and basic claims generation -- the revenue engine that makes the platform pay for itself.

**Sprint Dates:** 2026-04-12 to 2026-04-25 (10 working days)

**Status:** done

**References:** [[EPIC-005-revenue-cycle-mvp]], [[EPIC-008-demo-polish]], [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]]

**Note:** Sprint 3 now includes both the Revenue Cycle v1 tasks and the Demo Polish epic ([[EPIC-008-demo-polish]]). EPIC-008 covers: Approvals UI, frontend-backend API integration, WebSocket real-time events, agent runner API, patient intake workflow, enhanced seed data, and vault enrichment.

**Sprint 3 Actual Delivery:**
- 200+ tests passing, ruff clean
- Approvals UI fully functional with real API data
- WebSocket real-time agent events streaming
- Agent runner API for all 3 agents
- Patient intake workflow end-to-end
- Enhanced demo seed data with 5+ patients and multiple scenarios
- Dashboard agent stats widget with sparklines
- All existing frontend pages connected to real APIs (no mock data)

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S3-T01 | Build X12 270/271 eligibility request/response parser: generate 270 from patient demographics, parse 271 response into structured benefits | B | 6h | S1-T01 | 270 request generated from FHIR Patient, 271 parsed into coverage details | done |
| S3-T02 | Integrate with eligibility clearinghouse (Availity or Change Healthcare sandbox): REST API integration, credential management via Secrets Manager | B | 4h | S3-T01 | Real-time eligibility check returns active/inactive + benefit details | done |
| S3-T03 | Build eligibility verification UI: patient search, run check, display coverage, copay, deductible, coinsurance, prior auth requirements | B | 4h | S3-T02 | Front desk can verify eligibility before appointment, results displayed clearly | done |
| S3-T04 | Build AI coding engine: Claude analyzes signed SOAP note + encounter data -> suggests ICD-10 (diagnosis) + CPT (procedure) codes with confidence and rationale | A | 6h | S2-T05 | AI suggests codes with > 85% accuracy vs human coder on test cases | done |
| S3-T05 | Build coding review UI: display AI-suggested codes alongside note, search/add/remove codes, display code descriptions, accept all or modify | B | 4h | S3-T04 | Billing staff can review AI codes, modify, and finalize for claim | done |
| S3-T06 | Implement CMS-1500 claim generation: build claim from Encounter + Patient + Provider + Diagnosis + Procedure codes + Insurance info | A | 6h | S3-T04, S3-T02 | CMS-1500 generated with all required fields populated correctly | done |
| S3-T07 | Build X12 837P (professional claim) generator: convert CMS-1500 to EDI 837P format, validate against X12 specs | A | 4h | S3-T06 | 837P file generated, passes X12 syntax validation | done |
| S3-T08 | Build claims submission pipeline: 837P -> clearinghouse -> acknowledgment tracking, store claim status in FHIR Claim resource | A | 3h | S3-T07, S3-T02 | Claim submitted to sandbox, acknowledgment received and stored | done |
| S3-T09 | Build X12 835 (remittance) parser: parse ERA into payment details, adjustment reason codes, patient responsibility amounts | B | 4h | -- | 835 parsed into structured payment data, linked to original claim | done |
| S3-T10 | Build claims dashboard: list of claims by status (pending/submitted/accepted/denied/paid), filterable by date/provider/payer | B | 4h | S3-T08 | Billing staff sees claim lifecycle, can drill into details | done |
| S3-T11 | Implement AI denial prediction: Claude analyzes claim before submission, flags likely denial reasons, suggests modifications | A | 4h | S3-T04, S3-T07 | Claims flagged with denial probability, reasons listed, modifications suggested | done |
| S3-T12 | Sprint 3 end-to-end test: encounter -> AI coding -> review -> claim generation -> submission -> acknowledgment | BOTH | 4h | ALL | Full revenue cycle works: encounter to submitted claim | done |

### Sprint 3 Deliverables
- Real-time eligibility verification (X12 270/271)
- AI coding engine (ICD-10 + CPT from clinical notes)
- CMS-1500 claim generation and X12 837P EDI
- Claims submission pipeline with status tracking
- AI denial prediction before submission

### Sprint 3 Risks
| Risk | Mitigation |
|------|------------|
| Clearinghouse sandbox access delays | Apply for sandbox access during Sprint 1; use mock responses if delayed |
| X12 EDI format complexity | Use pyx12 library for parsing/generation; extensive test fixtures |
| AI coding accuracy below target | Multi-model consensus (Claude + GPT-4 for validation), human review mandatory in v1 |

**Decision Point:** Choose first clearinghouse integration partner (Availity vs Change Healthcare vs direct payer connections). Recommendation: Availity (free tier, broad payer coverage). Decision by Sprint 3 Day 2.

### Sprint 3 Definition of Done
- Eligibility check returns real results from clearinghouse sandbox
- AI suggests codes for 10 test encounters with > 85% accuracy
- CMS-1500 claim generated and submitted to sandbox
- Claims dashboard shows full lifecycle

---

## Sprint 4: Revenue Cycle v2

**Sprint Goal:** Add prior authorization automation, denial management, revenue analytics, and complete the X12 claims pipeline (837P generation, scrubbing, 835 parsing, payment posting) -- the features that generate the highest ROI for practices.

**Sprint Dates:** 2026-04-26 to 2026-05-09 (10 working days)

**Status:** in-progress

**References:** [[EPIC-005-revenue-cycle-mvp]], [[EPIC-009-revenue-cycle-completion]], [[Prior-Authorization-Deep-Dive]], [[X12-EDI-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]]

**Note:** Sprint 4 now includes both the original Revenue Cycle v2 tasks and the Revenue Cycle Completion epic ([[EPIC-009-revenue-cycle-completion]]). EPIC-009 covers: X12 837P claims generator, claims scrubbing engine, X12 835 remittance parser, payment posting, 4 new claims pipeline MCP tools, and claims analytics frontend.

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S4-T01 | Build prior auth requirements checker: given CPT code + payer, determine if prior auth required using payer rules database | B | 4h | S3-T02 | System correctly identifies PA-required procedures for top 10 payers | pending |
| S4-T02 | Build X12 278 prior auth request generator: patient demographics + diagnosis + procedure + clinical justification -> 278 request | B | 6h | S4-T01, S3-T07 | X12 278 request generated, passes syntax validation | pending |
| S4-T03 | Build AI clinical justification generator: Claude analyzes patient chart, generates medical necessity narrative for prior auth | A | 4h | S2-T05, S4-T01 | AI generates compelling clinical justification from encounter data | pending |
| S4-T04 | Build prior auth submission pipeline: 278 request -> clearinghouse -> response tracking (approved/denied/pended/partial) | B | 3h | S4-T02, S3-T02 | PA submitted, response parsed, status tracked in FHIR ClaimResponse | pending |
| S4-T05 | Build prior auth dashboard: pending requests, status tracking, turnaround time metrics, expiration warnings | B | 4h | S4-T04 | Staff sees all PAs with status, upcoming expirations, action needed items | pending |
| S4-T06 | Build AI denial management agent: analyze denial reason codes (CARC/RARC), determine appeal strategy, draft appeal letter with clinical evidence | A | 6h | S3-T09 | Agent identifies denial reason, suggests appeal strategy, generates draft letter | pending |
| S4-T07 | Build appeal letter generator: pull relevant clinical data from patient chart, format per payer requirements, include supporting literature | A | 4h | S4-T06 | Appeal letter generated with clinical evidence, ready for provider signature | pending |
| S4-T08 | Build denial tracking dashboard: denied claims, denial reasons, appeal status, win rate metrics | B | 4h | S4-T06 | Billing staff sees denied claims, appeal pipeline, historical win rates | pending |
| S4-T09 | Build revenue analytics dashboard: total charges, collections, denial rate, days in AR, clean claim rate, payer mix | B | 6h | S3-T09, S3-T10 | Practice sees key revenue metrics with trend lines and benchmarks | pending |
| S4-T10 | Build AI underpayment detector: compare remittance amounts against contracted rates, flag underpayments | A | 4h | S3-T09 | Underpayments flagged with expected vs actual amount and variance | pending |
| S4-T11 | Implement FHIR ClaimResponse and ExplanationOfBenefit resources for tracking claim outcomes | A | 3h | S3-T08 | Claim outcomes stored as FHIR resources, searchable by status/date/payer | pending |
| S4-T12 | Sprint 4 end-to-end test: encounter -> PA required -> submit PA -> approved -> claim -> ERA -> analytics | BOTH | 4h | ALL | Full revenue cycle with PA works end-to-end, analytics reflect data | pending |
| S4-T13 | X12 837P Claims Generator: generate HIPAA-compliant 005010X222A1 professional claims from FHIR Claim resources | A | 6h | -- | 837P output valid, all required segments populated, batch mode works | in-progress |
| S4-T14 | Claims Scrubbing Rules Engine: pre-submission validation with 15+ rules, denial risk scoring | A | 4h | S4-T13 | 15+ rules implemented, denial risk score calculated, error-level blocks submission | in-progress |
| S4-T15 | X12 835 Remittance Parser: parse ERA files into structured payment, adjustment, and patient responsibility data | A | 4h | -- | Parser handles 005010X221A1, extracts CLP/CAS/SVC/PLB segments | in-progress |
| S4-T16 | Payment Posting Module: match 835 payments to claims, calculate patient responsibility, detect underpayments | A | 3h | S4-T15 | Payments matched, claim status updated, FHIR ExplanationOfBenefit created | pending |
| S4-T17 | Claims Pipeline MCP Tools: 4 new billing tools (generate, scrub, post, analytics) | A | 4h | S4-T13, S4-T14, S4-T15, S4-T16 | 4 tools registered via @hipaa_tool, total 36 MCP tools | pending |
| S4-T18 | Claims Analytics Frontend: dashboard with clean claim rate, denial breakdown, AR aging | B | 4h | S4-T13, S4-T14, S4-T16 | Charts rendered, filters working, no PHI on analytics page | pending |

### Sprint 4 Deliverables
- Prior auth automation (X12 278 request/response)
- AI-generated clinical justification for prior auth
- Denial management with AI-drafted appeal letters
- Revenue analytics dashboard
- Underpayment detection
- X12 837P claims generator (005010X222A1 compliant)
- Claims scrubbing engine (15+ rules, denial risk scoring)
- X12 835 remittance parser
- Payment posting module with underpayment detection
- 4 new claims pipeline MCP tools (total: 36 MCP tools)
- Claims analytics frontend dashboard

### Sprint 4 Risks
| Risk | Mitigation |
|------|------------|
| Payer-specific PA requirements vary wildly | Start with top 5 FL payers (BCBS, Aetna, UHC, Cigna, Humana), expand iteratively |
| Appeal letter quality concerns from providers | Mandatory provider review and signature, track win rate to prove quality |
| Analytics accuracy with limited data | Clear "demo data" labels, plan for real data calibration during pilot |

**Decision Point:** Choose first pilot specialty. Recommendation: Orthopedics (high prior auth volume, procedure-heavy, tech-forward). Decision by Sprint 4 Day 5.

### Sprint 4 Definition of Done
- Prior auth submitted via X12 278 for test case
- AI generates appeal letter with clinical justification
- Revenue analytics show accurate metrics from test data
- Underpayment detection flags discrepancies
- X12 837P generator producing valid 005010X222A1 output
- Claims scrubber catching 15+ denial types with risk scoring
- X12 835 parser extracting payment, adjustment, and patient responsibility data
- Payment posting matching 835 payments to submitted claims
- 4 new MCP tools registered (total: 36 tools)
- Claims analytics on frontend with charts
- 220+ tests passing, ruff clean

---

## Sprint 5: Pilot Preparation

**Sprint Goal:** Harden security, build onboarding workflows, create training materials, and ensure the platform is ready for real practice use.

**Sprint Dates:** 2026-05-10 to 2026-05-23 (10 working days)

**References:** [[EPIC-006-pilot-readiness]], [[HIPAA-Deep-Dive]]

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S5-T01 | HIPAA Security Risk Assessment: document all PHI flows, identify risks, document mitigations, create risk register | B | 6h | ALL | Risk assessment document complete, all high risks have mitigations | pending |
| S5-T02 | Third-party penetration test: scope, engage vendor, remediate critical/high findings | A | 8h | ALL | Pen test complete, zero critical findings, all high findings remediated | pending |
| S5-T03 | Security hardening checklist: WAF rules tuned, rate limiting configured, input validation comprehensive, error messages sanitized (no PHI in errors) | A | 4h | S0-T15 | All OWASP Top 10 mitigated, error responses contain no PHI | pending |
| S5-T04 | Implement field-level encryption for SSN and sensitive identifiers using per-tenant KMS keys | A | 4h | S0-T08, S0-T33 | SSN encrypted at field level, decrypted only with tenant key, database inspection shows ciphertext | pending |
| S5-T05 | Build tenant onboarding wizard: organization info, admin user creation, practice configuration (specialties, locations, providers, payers) | B | 6h | S0-T33 | New practice can self-onboard: org created, admin account active, basic config complete | pending |
| S5-T06 | Build data migration tool: import patient demographics from CSV/HL7v2, validate against FHIR, create Patient resources, run matching | A | 6h | S1-T06 | CSV of 1000 patients imported with dedup, < 5% manual review rate | pending |
| S5-T07 | Create EHR integration bridge: FHIR R4 client for reading patient data from Epic/Cerner sandbox, bidirectional sync for demographics | A | 6h | S1-T01 | Read patient from Epic sandbox FHIR API, create corresponding MedOS patient | pending |
| S5-T08 | Create user training materials: video walkthroughs (Loom) for provider workflow, billing workflow, admin workflow | B | 6h | S2-T09, S3-T10 | 3 training videos (< 10 min each), covering core workflows | pending |
| S5-T09 | Build practice configuration panel: manage providers, locations, fee schedules, payer contracts, specialty settings | B | 4h | S5-T05 | Admin can configure all practice parameters without technical support | pending |
| S5-T10 | Implement monitoring and alerting: CloudWatch dashboards for API latency/errors/throughput, PagerDuty integration for critical alerts | A | 3h | S0-T17 | Dashboards show key metrics, alerts fire on error rate > 5% or P99 > 2s | pending |
| S5-T11 | Create incident response playbook: procedures for data breach, system outage, security incident, on-call rotation | B | 3h | -- | Playbook documented, contact list current, tested with tabletop exercise | pending |
| S5-T12 | Load testing: simulate 50 concurrent users, 100 encounters/hour, verify system handles pilot load | A | 4h | ALL | System handles pilot load with P99 < 1s, no errors, no data loss | pending |

### Sprint 5 Deliverables
- HIPAA risk assessment completed
- Pen test passed with zero critical findings
- Tenant onboarding wizard functional
- Data migration tool tested
- Training materials created
- Monitoring and alerting operational

### Sprint 5 Risks
| Risk | Mitigation |
|------|------------|
| Pen test finding critical vulnerability | Budget 2 days for remediation; defer pilot launch if unresolvable critical exists |
| EHR sandbox access delays | Apply for Epic/Cerner sandbox during Sprint 2; use mock FHIR server if delayed |
| Training materials need clinical validation | Have clinical advisor review before pilot |

### Sprint 5 Definition of Done
- Pen test report shows zero critical, zero high unresolved
- HIPAA risk assessment documented and signed
- New practice onboards in < 30 minutes
- Monitoring alerts tested end-to-end

---

## Sprint 6: Launch

**Sprint Goal:** Set up demo environment, onboard first pilot practice, begin live operations with monitoring and support.

**Sprint Dates:** 2026-05-24 to 2026-05-29 (5 working days -- half sprint)

**References:** [[EPIC-006-pilot-readiness]], [[HEALTHCARE_OS_MASTERPLAN]]

| ID | Task | Owner | Est. | Deps | Acceptance Criteria | Status |
|----|------|-------|------|------|---------------------|--------|
| S6-T01 | Deploy production environment: Terraform apply for prod account, verify all services healthy, SSL certs valid | A | 4h | ALL | Prod environment fully deployed, all health checks passing, HTTPS working | pending |
| S6-T02 | Create demo environment with synthetic patient data: 50 patients, 200 encounters, sample claims, realistic but non-PHI | A | 3h | S6-T01 | Demo environment populated, all features demonstrable without real PHI | pending |
| S6-T03 | Pilot practice onboarding (Practice #1): run onboarding wizard, import patient demographics, configure payers, create user accounts | B | 4h | S5-T05, S5-T06 | Practice live: admin logged in, patients imported, payers configured | pending |
| S6-T04 | Conduct on-site training session with pilot practice staff: providers, billing, front desk (2-hour session per role group) | B | 6h | S5-T08, S6-T03 | Staff trained, can perform core workflows independently | pending |
| S6-T05 | Configure pilot success metrics tracking: time saved per provider, coding accuracy, claim acceptance rate, denial rate, user adoption | B | 3h | S4-T09 | Metrics dashboard configured, baseline measurements captured | pending |
| S6-T06 | Set up daily monitoring rotation: check error rates, review AI output quality, monitor costs, respond to support requests | A | 2h | S5-T10 | Monitoring rotation documented, first week schedule assigned | pending |
| S6-T07 | Configure automated backup verification: daily backup test restore, alert on failure | A | 2h | S0-T14 | Backup restored successfully to test instance, restore time < 1 hour | pending |
| S6-T08 | Create pilot feedback channel: Slack/Teams channel with practice, weekly check-in cadence, issue tracker for pilot bugs | BOTH | 1h | S6-T03 | Communication channel live, first weekly check-in scheduled | pending |
| S6-T09 | Launch day go-live support: on-call for first full day of pilot practice usage, real-time issue resolution | BOTH | 8h | ALL | First day completes with zero blocking issues, practice can see patients with system | pending |

### Sprint 6 Deliverables
- Production environment operational
- Demo environment with synthetic data
- First pilot practice onboarded and trained
- Success metrics tracking live
- Support and monitoring rotation active

### Sprint 6 Definition of Done
- Pilot practice is live and using the system for real patient encounters
- Feedback channel active with first check-in scheduled
- All monitoring green, backups verified
- Day 45 interim review scheduled on calendar

---

## Parallel Track Visualization

```
Week   | Person A (Infra/Backend)          | Person B (Domain/Business)
-------|-----------------------------------|----------------------------------
W1     | AWS Orgs, Terraform, VPC, KMS     | Auth eval, GitHub repo, CI/CD
W2     | RDS, ECS, Security, FastAPI        | Auth impl, RBAC, Roles, MFA
       | Sessions, FHIR tables, Patient     | Audit trail, API keys, Break-glass
       | CRUD, search, integration tests    | Env separation, cost monitoring
W3     | Encounter, Observation, Condition  | Practitioner, Organization, Consent
       | Patient matching, Bundles          | Event bus, FHIR validation
W4     | Event publisher, versioning        | Capability Statement
       | pgvector embeddings                | Integration tests
W5     | Whisper deploy, audio pipeline     | Provider workspace UI
       | Claude NLP, SOAP generator         | Note review UI, real-time status
W6     | Confidence scoring, LangGraph      | ICD-10/CPT suggestions UI
       | Langfuse integration               | Encounter summary dashboard
W7     | AI coding engine, CMS-1500         | Eligibility (270/271), clearinghouse
       | 837P generator, submission          | Coding review UI, 835 parser
W8     | AI denial prediction               | Claims dashboard
       | Integration tests                  | Integration tests
W9     | AI clinical justification          | PA requirements, X12 278
       | Denial mgmt agent, appeal gen      | PA submission, PA dashboard
W10    | Underpayment detector              | Revenue analytics, denial dashboard
       | ClaimResponse/EOB resources        | Integration tests
W11    | Pen test, security hardening       | HIPAA risk assessment
       | Field-level encryption, data mig   | Onboarding wizard, config panel
W12    | EHR bridge, load testing           | Training materials, playbook
       | Monitoring/alerting                | Practice configuration
W13    | Prod deploy, demo env, backups     | Pilot onboarding, training
       | Go-live support, monitoring        | Success metrics, feedback channel
```

---

## Decision Log

| Decision | Sprint | Owner | Deadline | Options | Default If Undecided |
|----------|--------|-------|----------|---------|---------------------|
| Auth provider selection | S0 | B | Day 2 (Mar 3) | Auth0, Clerk, Keycloak | Auth0 Enterprise |
| Clearinghouse partner | S3 | B | S3 Day 2 (Apr 13) | Availity, Change Healthcare, direct | Availity |
| First pilot specialty | S4 | BOTH | S4 Day 5 (Apr 30) | Orthopedics, Dermatology, Cardiology | Orthopedics |
| GPU hosting strategy | S2 | A | S2 Day 1 (Mar 29) | Self-hosted Whisper, Whisper API, Deepgram | Self-hosted (cost control) |
| Compliance tooling | S5 | B | S5 Day 1 (May 10) | Vanta, Drata, manual | Vanta |
| First pilot practice | S5 | B | S5 Day 5 (May 14) | Network contacts, cold outreach, clinical advisor intro | Clinical advisor intro |

---

## Budget Tracking

| Sprint | AWS Estimate | Tools/Services | Notes |
|--------|-------------|----------------|-------|
| S0 | $300-500 | $440 (Claude Code $400 + Cursor $40) | RDS dev instance, single NAT, minimal ECS |
| S1 | $400-600 | $440 | EventBridge, increased ECS usage |
| S2 | $800-1,200 | $640 (+ Claude API $200) | GPU instance for Whisper, Claude API calls |
| S3 | $600-900 | $640 (+ clearinghouse sandbox $0) | Clearinghouse sandbox typically free |
| S4 | $600-900 | $640 | Stable infrastructure, more API calls |
| S5 | $800-1,200 | $940 (+ Vanta $300) | Pen test vendor: $5-10K separate, load testing |
| S6 | $1,000-1,500 | $640 | Production environment, demo environment |
| **Total** | **$4,500-6,800** | **$4,380+** | **Pen test: $5-10K. Grand total: ~$15-22K** |

### Cost Controls
- Dev environment: single NAT, FARGATE_SPOT, smallest RDS instance
- GPU (Whisper): auto-scale to zero when idle, batch transcriptions
- VPC endpoints reduce NAT data transfer costs
- Reserved Instances evaluation at Month 3 if committing to current sizing
- Weekly cost review every Friday (S0-T39 budget alerts catch spikes)

---

## Appendix A: Task Count Summary

| Sprint | Total Tasks | Person A | Person B | Both | Total Hours |
|--------|------------|----------|----------|------|-------------|
| S0 | 43 | 24 | 17 | 2 | ~130h |
| S1 | 15 | 8 | 6 | 1 | ~58h |
| S2 | 14 | 8 | 5 | 1 | ~58h |
| S3 | 12 | 6 | 5 | 1 | ~53h |
| S4 | 18 | 10 | 7 | 1 | ~76h |
| S5 | 12 | 6 | 5 | 1 | ~54h |
| S6 | 9 | 4 | 4 | 1 | ~33h |
| **Total** | **123** | **66** | **49** | **8** | **~462h** |

At 8h/day per person, 10 days/sprint = 160h/sprint for the team. Each sprint uses 30-80% of available capacity, leaving buffer for unexpected issues, debugging, meetings, and pilot relationship management.

---

## Appendix B: Epic-to-Sprint Mapping

| Epic | Primary Sprint | Supporting Sprints |
|------|---------------|-------------------|
| [[EPIC-001-aws-infrastructure-foundation]] | S0 (Days 1-5) | S5 (security hardening), S6 (prod deploy) |
| [[EPIC-002-auth-identity-system]] | S0 (Days 6-10) | S5 (pen test, MFA validation) |
| [[EPIC-003-fhir-data-layer]] | S1 | S0 (base tables), S2 (DocumentReference) |
| [[EPIC-004-ai-clinical-documentation]] | S2 | S3 (AI coding uses same pipeline) |
| [[EPIC-005-revenue-cycle-mvp]] | S3, S4 | S2 (ICD-10/CPT from notes) |
| [[EPIC-006-pilot-readiness]] | S5, S6 | All sprints contribute compliance evidence |
| [[EPIC-007-mcp-sdk-refactoring]] | S2 | S3 (agent runner consumes MCP tools) |
| [[EPIC-008-demo-polish]] | S3 | S2 (builds on 32 MCP tools + 3 agents) |
| [[EPIC-009-revenue-cycle-completion]] | S4 | S3 (builds on eligibility + coding + claims base) |

---

## Appendix C: Critical Path

The critical path through Phase 1 (longest dependency chain):

```
AWS Orgs (S0-T01, Day 1)
  -> Terraform modules (S0-T06, Day 2)
    -> VPC (S0-T10, Day 3)
      -> RDS PostgreSQL (S0-T14, Day 4)
        -> FHIR base tables (S0-T32, Day 8)
          -> Patient CRUD (S0-T36, Day 9)
            -> Encounter CRUD (S1-T01, Week 3)
              -> SOAP note generator (S2-T05, Week 6)
                -> AI coding engine (S3-T04, Week 7)
                  -> CMS-1500 claim (S3-T06, Week 8)
                    -> Prior auth (S4-T02, Week 9)
                      -> Pilot onboarding (S6-T03, Week 13)
```

Any delay on this path delays the pilot. Tasks off the critical path (auth, UI, analytics) have float and can absorb delays without impacting the launch date.

---

## How to Use This Document

1. **Every morning:** Check the current sprint's task board. Pick the highest-priority unblocked task assigned to you.
2. **Every task:** Read the acceptance criteria BEFORE starting. You are done when the criteria are met, not when you feel done.
3. **Every day:** Update task status (pending -> in-progress -> done). Flag blockers immediately.
4. **Every Friday:** Review sprint progress. Identify tasks at risk. Adjust next week's plan if needed.
5. **Every sprint end:** Demo deliverables. Run integration tests. Retrospective. Plan next sprint.
6. **When blocked:** Switch to a parallel task. Never sit idle waiting for a dependency.
7. **When ahead:** Pull tasks from the next sprint. The 90-day timeline has buffer but no slack.
