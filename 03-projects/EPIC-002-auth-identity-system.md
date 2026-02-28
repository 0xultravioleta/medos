---
type: epic
date: "2026-02-27"
status: planning
priority: 2
tags:
  - project
  - epic
  - phase-1
  - auth
  - security
  - module-g
owner: ""
target-date: "2026-03-13"
---

# EPIC-002: Authentication & Identity System

> **Timeline:** Week 1-2 (2026-02-27 to 2026-03-13)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-001-aws-infrastructure-foundation]] (VPC, ECS, KMS)
> **Blocks:** [[EPIC-003-fhir-data-layer]], [[EPIC-004-ai-clinical-documentation]], [[EPIC-005-revenue-cycle-mvp]]

## Objective

Implement a HIPAA-compliant authentication and authorization system supporting SMART on FHIR OAuth2 flows, multi-tenant RBAC+ABAC, and comprehensive audit logging. This system gates every API call in the platform and must be production-hardened from day one.

---

## Timeline (Gantt)

```
Week 1 (Feb 27 - Mar 6)
|----- T1: Auth Provider Evaluation + BAA --------|
|----------- T2: RBAC + ABAC Design -------------|
|----------- T3: User Role Definitions ------------|

Week 2 (Mar 7 - Mar 13)
|---- T4: MFA Enforcement -------------------------|
|---- T5: Session Management (Redis) --------------|
|--------- T6: API Key Management -----------------|
|--------- T7: Break-the-Glass Access -------------|
|-------------- T8: Auth Audit Trail --------------|
|-------------- T9: JWT Token Structure -----------|
|------------------ T10: Integration Testing ------|
```

---

## Tasks

### T1: SMART on FHIR OAuth2 Provider Evaluation and BAA
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** None (can start immediately, infra not required for evaluation)
**References:** [[ADR-002-multi-tenancy-schema-per-tenant]], [[HIPAA-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Evaluate Auth0, Clerk, and custom Keycloak for SMART on FHIR OAuth2 compliance. The chosen provider must sign a BAA and support tenant-scoped tokens.

**Subtasks:**
- [ ] Create evaluation matrix:
  | Criteria | Auth0 | Clerk | Keycloak (self-hosted) |
  |----------|-------|-------|------------------------|
  | SMART on FHIR support | ? | ? | ? |
  | BAA availability | ? | ? | N/A (self-hosted) |
  | Multi-tenant support | ? | ? | ? |
  | Custom claims in JWT | ? | ? | ? |
  | RBAC/ABAC built-in | ? | ? | ? |
  | MFA options | ? | ? | ? |
  | Price at 1K/10K/50K users | ? | ? | ? |
  | HIPAA compliance docs | ? | ? | ? |
- [ ] Request BAA from top 2 candidates
- [ ] Test SMART on FHIR launch sequence with each candidate
- [ ] Test custom claims injection (tenant_id, roles, permissions)
- [ ] Document decision in ADR format
- [ ] Sign BAA with selected provider

**Acceptance Criteria:**
- [ ] ADR published with clear rationale
- [ ] BAA signed and stored in legal repository
- [ ] SMART on FHIR launch sequence verified with test EHR
- [ ] Cost projection documented for 1-year horizon

---

### T2: RBAC + ABAC Permission Model Design
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T1 (need to know provider capabilities)
**References:** [[ADR-002-multi-tenancy-schema-per-tenant]], [[HIPAA-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Design the permission model combining Role-Based Access Control (who you are) with Attribute-Based Access Control (context of the request). Healthcare requires fine-grained access: a provider can see their own patients but not another provider's patients in the same practice.

**Subtasks:**
- [ ] Define RBAC hierarchy:
  ```
  super-admin        (platform-level, MedOS staff only)
  ├── tenant-admin   (practice administrator)
  ├── provider       (physician, NP, PA)
  ├── nurse          (RN, LPN, MA)
  ├── billing        (billing specialist, coder)
  ├── front-desk     (scheduler, receptionist)
  └── patient        (patient portal access)
  ```
- [ ] Define permission matrix per role:
  | Permission | super-admin | tenant-admin | provider | nurse | billing | front-desk | patient |
  |---|---|---|---|---|---|---|---|
  | patient:read | all | tenant | assigned | assigned | tenant | limited | self |
  | patient:write | all | tenant | assigned | assigned | none | limited | none |
  | encounter:create | all | none | yes | yes | none | none | none |
  | billing:read | all | tenant | own | none | tenant | none | own |
  | billing:write | all | none | none | none | tenant | none | none |
  | admin:users | all | tenant | none | none | none | none | none |
  | reports:view | all | tenant | own | none | tenant | none | none |
- [ ] Define ABAC attributes:
  - `tenant_id` - tenant isolation (mandatory)
  - `provider_id` - provider-patient assignment
  - `department` - department-level access
  - `location` - multi-location practices
  - `time_of_day` - restrict access outside business hours (optional)
  - `ip_range` - restrict to practice network (optional)
- [ ] Design policy evaluation engine:
  ```
  ALLOW if: role.hasPermission(action, resource)
        AND: abac.evaluate(request.attributes, policy.conditions)
        AND: tenant.matches(user.tenant_id, resource.tenant_id)
  ```
- [ ] Document in `docs/architecture/permission-model.md`

**Acceptance Criteria:**
- [ ] Permission matrix covers all FHIR resource types used in MVP
- [ ] ABAC conditions support tenant isolation, provider assignment, and department scoping
- [ ] Policy evaluation logic documented with pseudocode
- [ ] Edge cases documented: cross-tenant access (denied), patient self-access, delegated access

---

### T3: User Role and Scope Definitions
**Complexity:** S
**Estimate:** 1 day
**Dependencies:** T2
**References:** [[HIPAA-Deep-Dive]]

**Description:**
Implement role and scope definitions in the auth provider, including FHIR-specific scopes as defined by SMART on FHIR.

**Subtasks:**
- [ ] Create roles in auth provider matching T2 hierarchy
- [ ] Define SMART on FHIR scopes:
  - `patient/*.read` - read all patient-compartment resources
  - `patient/*.write` - write all patient-compartment resources
  - `user/*.read` - read resources in user context
  - `launch/patient` - receive patient context at launch
  - `launch/encounter` - receive encounter context at launch
  - `openid`, `fhirUser`, `profile` - identity scopes
- [ ] Map roles to default scope sets
- [ ] Create scope validation middleware
- [ ] Test scope enforcement with sample API calls

**Acceptance Criteria:**
- [ ] All roles created in auth provider
- [ ] SMART on FHIR scopes defined and enforceable
- [ ] Scope validation middleware rejects requests with insufficient scopes
- [ ] Test suite covers all role-scope combinations

---

### T4: MFA Enforcement
**Complexity:** S
**Estimate:** 1 day
**Dependencies:** T1
**References:** [[HIPAA-Deep-Dive]], [[SOC2-HITRUST-Roadmap]]

**Description:**
Enforce multi-factor authentication for all clinical users. HIPAA does not mandate MFA, but it is a critical safeguard for PHI access.

**Subtasks:**
- [ ] Configure MFA policies by role:
  | Role | MFA Required | Methods |
  |------|-------------|---------|
  | super-admin | Always | TOTP, WebAuthn |
  | tenant-admin | Always | TOTP, WebAuthn, SMS |
  | provider | Always | TOTP, WebAuthn, SMS |
  | nurse | Always | TOTP, SMS |
  | billing | Always | TOTP, SMS |
  | front-desk | First login + periodic | TOTP, SMS |
  | patient | Optional (encouraged) | SMS, Email |
- [ ] Implement MFA enrollment flow in onboarding
- [ ] Configure MFA challenge on:
  - New device detection
  - IP change detection
  - Sensitive operations (user management, bulk export)
- [ ] Implement MFA bypass for break-the-glass (see T7) with audit logging
- [ ] Test MFA flow end-to-end for each method

**Acceptance Criteria:**
- [ ] Clinical users cannot access the system without MFA
- [ ] At least 2 MFA methods available
- [ ] MFA enrollment tracked per user
- [ ] Bypass mechanism exists only for emergencies (T7)

---

### T5: Session Management (Redis-backed)
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** [[EPIC-001-aws-infrastructure-foundation]] T3 (VPC), T1
**References:** [[HIPAA-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Implement server-side session management using Redis (ElastiCache) for session revocation, concurrent session control, and idle timeout enforcement.

**Subtasks:**
- [ ] Deploy ElastiCache Redis cluster:
  - Engine: Redis 7.x
  - Node type: `cache.t4g.micro` (dev), `cache.r6g.large` (prod)
  - Encryption at rest (KMS) and in transit (TLS)
  - Placed in private subnets
  - Multi-AZ: prod only
- [ ] Implement session management:
  - Session creation on login (store: user_id, tenant_id, roles, ip, user_agent)
  - Session expiration: 8 hours (clinical), 30 minutes idle timeout
  - Concurrent session limit: 3 per user (configurable per tenant)
  - Session revocation API (admin: revoke by user, bulk revoke by tenant)
- [ ] Implement session middleware:
  - Validate session on every request
  - Extend idle timeout on activity
  - Reject expired/revoked sessions immediately
- [ ] Add session events to audit log:
  - Session created, extended, expired, revoked
  - Concurrent session eviction
- [ ] Handle session migration on role change (force re-auth)

**Acceptance Criteria:**
- [ ] Sessions stored in Redis with TTL enforcement
- [ ] Idle timeout terminates session after 30 minutes of inactivity
- [ ] Admin can revoke sessions for a user or tenant
- [ ] Concurrent session limit enforced
- [ ] All session events appear in audit log

---

### T6: API Key Management
**Complexity:** M
**Estimate:** 1 day
**Dependencies:** T2
**References:** [[System-Architecture-Overview]]

**Description:**
Implement API key management for third-party integrations (lab systems, imaging, pharmacy). API keys must be tenant-scoped and auditable.

**Subtasks:**
- [ ] Design API key model:
  ```
  api_keys:
    id: uuid
    tenant_id: uuid
    name: string (human-readable label)
    key_hash: string (bcrypt hash, never store plaintext)
    prefix: string (first 8 chars for identification)
    scopes: string[] (subset of role scopes)
    created_by: uuid (user who created)
    expires_at: timestamp
    last_used_at: timestamp
    revoked_at: timestamp (null if active)
  ```
- [ ] Implement API endpoints:
  - `POST /api/v1/api-keys` - create key (returns plaintext once)
  - `GET /api/v1/api-keys` - list keys (prefix only, no secrets)
  - `DELETE /api/v1/api-keys/:id` - revoke key
  - `GET /api/v1/api-keys/:id/usage` - usage statistics
- [ ] Implement API key authentication middleware:
  - Accept via `X-API-Key` header
  - Rate limit per key: 1000 req/min (configurable)
  - Validate tenant_id and scopes
- [ ] Log all API key operations to audit trail
- [ ] Implement key rotation: create new key, deprecation period, revoke old

**Acceptance Criteria:**
- [ ] API key created, used for auth, and revoked successfully
- [ ] Plaintext key shown only once at creation
- [ ] Expired and revoked keys rejected immediately
- [ ] Rate limiting enforced per key
- [ ] All usage logged with timestamps and IP addresses

---

### T7: Break-the-Glass Emergency Access
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T2, T5
**References:** [[HIPAA-Deep-Dive]]

**Description:**
Implement emergency access override for clinical emergencies where a provider needs access to a patient's records outside their normal permissions. HIPAA allows this but requires extensive logging.

**Subtasks:**
- [ ] Design break-the-glass flow:
  1. Provider requests emergency access to patient record
  2. System presents justification form (reason dropdown + free text)
  3. Provider acknowledges audit warning
  4. System grants temporary elevated access (configurable: 2-4 hours)
  5. All actions during elevated access are flagged in audit trail
  6. Notification sent to: tenant admin, privacy officer, patient (if portal enabled)
  7. Access auto-revokes after time window
- [ ] Implement justification reasons enum:
  - Medical emergency
  - Patient unable to consent (unconscious)
  - Covering for absent provider
  - Quality review (must be approved by admin)
  - Other (requires free text)
- [ ] Create FHIR AuditEvent for break-the-glass:
  ```json
  {
    "resourceType": "AuditEvent",
    "type": { "code": "emergency-access-override" },
    "action": "E",
    "outcome": "0",
    "agent": [{ "who": { "reference": "Practitioner/123" } }],
    "entity": [{ "what": { "reference": "Patient/456" } }],
    "extension": [{
      "url": "http://medos.health/fhir/break-the-glass-justification",
      "valueString": "Medical emergency - patient presented unconscious in ED"
    }]
  }
  ```
- [ ] Create admin review dashboard for break-the-glass events
- [ ] Implement abuse detection: alert on > 3 break-the-glass events per provider per week
- [ ] Test full flow including notification delivery

**Acceptance Criteria:**
- [ ] Provider can access records outside normal scope with justification
- [ ] Every action during elevated access is audit-logged with break-the-glass flag
- [ ] Notifications delivered to admin and privacy officer
- [ ] Access auto-revokes after configured time window
- [ ] Abuse detection triggers on excessive usage

---

### T8: Auth Audit Trail (FHIR AuditEvent)
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T5, [[EPIC-001-aws-infrastructure-foundation]] T6 (RDS)
**References:** [[HIPAA-Deep-Dive]], [[FHIR-R4-Deep-Dive]]

**Description:**
Implement comprehensive audit logging for all authentication and authorization events using FHIR AuditEvent resources. This is a HIPAA requirement for access accountability.

**Subtasks:**
- [ ] Design AuditEvent schema in PostgreSQL:
  ```sql
  CREATE TABLE audit_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    fhir_resource JSONB NOT NULL,  -- Full FHIR AuditEvent
    event_type VARCHAR(50) NOT NULL,
    action CHAR(1) NOT NULL,  -- C/R/U/D/E
    outcome CHAR(1) NOT NULL, -- 0=success, 4=minor fail, 8=serious fail, 12=major fail
    agent_user_id UUID,
    entity_patient_id UUID,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    source_ip INET,
    user_agent TEXT
  );
  CREATE INDEX idx_audit_tenant_time ON audit_events (tenant_id, recorded_at DESC);
  CREATE INDEX idx_audit_agent ON audit_events (agent_user_id, recorded_at DESC);
  CREATE INDEX idx_audit_patient ON audit_events (entity_patient_id, recorded_at DESC);
  ```
- [ ] Implement audit event emitter (async, non-blocking):
  - Login success/failure
  - Logout
  - Token refresh
  - Permission denied
  - Role change
  - Session creation/revocation
  - API key usage
  - Break-the-glass activation/deactivation
  - MFA enrollment/challenge/bypass
  - Password change/reset
- [ ] Create audit query API:
  - `GET /api/v1/audit-events?user_id=X&from=DATE&to=DATE`
  - `GET /api/v1/audit-events?patient_id=X` (patient access report - HIPAA right)
  - Pagination, filtering by event type, outcome
- [ ] Implement audit event archival:
  - After 90 days: compress and move to S3
  - Retain in S3 for 7 years (HIPAA requirement)
  - Searchable via Athena for historical queries
- [ ] Create patient access report generator (HIPAA: patients can request who accessed their records)

**Acceptance Criteria:**
- [ ] Every auth event generates a FHIR-compliant AuditEvent
- [ ] Audit events are immutable (no UPDATE/DELETE permissions on audit table)
- [ ] Patient access report returns all access events for a given patient
- [ ] Archival pipeline moves events to S3 after 90 days
- [ ] Query performance: < 500ms for 30-day range per tenant

---

### T9: JWT Token Structure with Tenant Context
**Complexity:** S
**Estimate:** 1 day
**Dependencies:** T1, T2
**References:** [[ADR-002-multi-tenancy-schema-per-tenant]], [[System-Architecture-Overview]]

**Description:**
Define and implement the JWT token structure that carries tenant context, roles, and FHIR scopes for every authenticated request.

**Subtasks:**
- [ ] Define JWT claims structure:
  ```json
  {
    "sub": "user-uuid",
    "iss": "https://auth.medos.health",
    "aud": "https://api.medos.health",
    "exp": 1709136000,
    "iat": 1709107200,
    "tenant_id": "tenant-uuid",
    "tenant_schema": "tenant_abc123",
    "roles": ["provider"],
    "permissions": ["patient:read:assigned", "encounter:create"],
    "fhir_user": "Practitioner/pract-uuid",
    "fhir_scopes": ["patient/*.read", "patient/Encounter.write"],
    "mfa_verified": true,
    "session_id": "session-uuid",
    "department": "cardiology",
    "location_id": "loc-uuid"
  }
  ```
- [ ] Implement JWT validation middleware:
  - Verify signature (RS256 with JWKS endpoint)
  - Validate `exp`, `iat`, `iss`, `aud`
  - Extract tenant context and set on request
  - Set PostgreSQL `search_path` to tenant schema (from `tenant_schema` claim)
  - Reject tokens without `tenant_id` (except super-admin)
- [ ] Implement token refresh flow:
  - Refresh token stored server-side (Redis)
  - Refresh extends session (T5)
  - Refresh rejected if session revoked
- [ ] Configure JWKS endpoint caching (5 minute TTL)
- [ ] Write unit tests for all validation edge cases

**Acceptance Criteria:**
- [ ] JWT contains all required claims for tenant isolation and RBAC
- [ ] Middleware sets PostgreSQL search_path per-request based on token
- [ ] Invalid/expired tokens return 401 with no PHI in error response
- [ ] Token refresh works and extends session
- [ ] JWKS key rotation does not cause downtime

---

### T10: Integration Testing
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1-T9
**References:** [[System-Architecture-Overview]]

**Description:**
End-to-end integration tests for the complete auth flow, including cross-tenant isolation verification.

**Subtasks:**
- [ ] Create test tenants: `tenant-alpha`, `tenant-beta`
- [ ] Create test users across roles in both tenants
- [ ] Test matrix:
  | Scenario | Expected |
  |---|---|
  | Provider in tenant-alpha reads own patient | 200 OK |
  | Provider in tenant-alpha reads tenant-beta patient | 403 Forbidden |
  | Nurse reads patient not assigned to their provider | 403 Forbidden |
  | Admin lists all users in own tenant | 200 OK |
  | Admin lists users in other tenant | 403 Forbidden |
  | Billing reads clinical notes | 403 Forbidden |
  | Patient reads own record | 200 OK |
  | Patient reads another patient's record | 403 Forbidden |
  | Break-the-glass: provider accesses unassigned patient | 200 OK + audit |
  | Expired token used | 401 Unauthorized |
  | Revoked session used | 401 Unauthorized |
  | API key with limited scopes | Scoped access |
  | MFA not completed | 403 MFA Required |
- [ ] Create load test for auth middleware (target: < 5ms overhead per request)
- [ ] Create security test: SQL injection in auth parameters
- [ ] Create security test: JWT tampering (modified claims)
- [ ] Document all test results

**Acceptance Criteria:**
- [ ] All cross-tenant isolation tests pass
- [ ] All role-based access tests pass
- [ ] Auth middleware adds < 5ms latency per request
- [ ] No security vulnerabilities found in auth flow
- [ ] Test suite runs in CI pipeline

---

## Dependencies Map

```
T1 (Auth Provider) ──> T2 (RBAC/ABAC) ──> T3 (Roles)
                   └──> T4 (MFA)          └──> T6 (API Keys)
                   └──> T9 (JWT)          └──> T7 (Break-Glass)

T5 (Sessions) requires EPIC-001 T3 (VPC) + T1
T8 (Audit) requires T5 + EPIC-001 T6 (RDS)
T10 (Integration) requires T1-T9
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| JWT middleware with tenant context (T9) | [[EPIC-003-fhir-data-layer]] (schema-per-tenant routing) |
| RBAC permission checks (T2) | [[EPIC-003-fhir-data-layer]] (FHIR resource access control) |
| FHIR AuditEvent pipeline (T8) | [[EPIC-003-fhir-data-layer]], [[EPIC-004-ai-clinical-documentation]] |
| API key system (T6) | [[EPIC-005-revenue-cycle-mvp]] (clearinghouse integrations) |
| Auth provider + MFA (T1, T4) | [[EPIC-006-pilot-readiness]] (security hardening) |
| Break-the-glass (T7) | [[EPIC-006-pilot-readiness]] (compliance checklist) |

| This Epic Requires | Provided By |
|---|---|
| VPC + private subnets | [[EPIC-001-aws-infrastructure-foundation]] T3 |
| RDS PostgreSQL | [[EPIC-001-aws-infrastructure-foundation]] T6 |
| KMS keys | [[EPIC-001-aws-infrastructure-foundation]] T4 |
| ECS cluster | [[EPIC-001-aws-infrastructure-foundation]] T7 |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Auth provider does not sign BAA in time | Medium | Critical | Start BAA process day 1; have Keycloak self-hosted as fallback |
| SMART on FHIR implementation complexity | Medium | High | Start with basic OAuth2 and add SMART-specific flows incrementally |
| Redis ElastiCache cost in multi-AZ | Low | Medium | Single-node in dev; session fallback to DB if Redis unavailable |
| JWT token size exceeds limits (large permission sets) | Medium | Medium | Use permission IDs instead of full permission strings; lazy-load full permissions |
| Break-the-glass abuse by providers | Low | High | Automated abuse detection + weekly review by privacy officer |
| MFA friction leads to provider pushback | Medium | Medium | Offer WebAuthn (biometric) as low-friction option; session duration reduces re-auth |

---

## Definition of Done

- [ ] All clinical users authenticate with MFA
- [ ] Cross-tenant data access is impossible (verified by integration tests)
- [ ] Every auth event generates a FHIR AuditEvent
- [ ] Break-the-glass flow works end-to-end with audit trail
- [ ] JWT tokens carry tenant context and set PostgreSQL search_path
- [ ] Session management enforces idle timeout and concurrent limits
- [ ] API keys work for third-party integrations with scoped access
- [ ] Auth middleware adds < 5ms latency per request
- [ ] BAA signed with auth provider
- [ ] Security test suite passes in CI
