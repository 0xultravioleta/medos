---
type: epic
date: "2026-02-27"
status: planning
priority: 3
tags:
  - project
  - epic
  - phase-1
  - fhir
  - database
  - module-a
owner: ""
target-date: "2026-03-27"
---

# EPIC-003: FHIR Data Layer

> **Timeline:** Week 2-4 (2026-03-06 to 2026-03-27)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-001-aws-infrastructure-foundation]] (RDS, VPC), [[EPIC-002-auth-identity-system]] (JWT, RBAC)
> **Blocks:** [[EPIC-004-ai-clinical-documentation]], [[EPIC-005-revenue-cycle-mvp]], [[EPIC-006-pilot-readiness]]

## Objective

Build the core FHIR-native data layer on PostgreSQL that stores, validates, queries, and serves FHIR R4 resources. This is the data foundation for every module in MedOS -- clinical documentation, billing, scheduling, and AI features all depend on this layer.

---

## Timeline (Gantt)

```
Week 2 (Mar 6 - Mar 13)
|----- T1: FHIR JSONB Schema -------------------|
|----- T2: Schema-per-Tenant Provisioning -------|
|----------- T3: FHIR Validation Layer ----------|

Week 3 (Mar 14 - Mar 20)
|----- T4: FHIR CRUD API Endpoints --------------|
|----- T5: FHIR Search Parameters ---------------|
|----------- T6: Patient Identity Matching -------|
|----------- T7: FHIR Bundle Transactions --------|

Week 4 (Mar 21 - Mar 27)
|----- T8: Event Emission System -----------------|
|----- T9: pgvector Embeddings Pipeline ----------|
|----------- T10: Data Migration Tooling ---------|
|----------- T11: FHIR $everything Operation -----|
```

---

## Tasks

### T1: PostgreSQL FHIR JSONB Schema
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** [[EPIC-001-aws-infrastructure-foundation]] T6 (RDS PostgreSQL)
**References:** [[ADR-001-fhir-native-data-model]], [[FHIR-R4-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Design and implement the core PostgreSQL schema for storing FHIR R4 resources as JSONB with proper indexing for search performance. Following [[ADR-001-fhir-native-data-model]], resources are stored as native JSONB with GIN indexes.

**Subtasks:**
- [ ] Create base resource table structure:
  ```sql
  CREATE TABLE fhir_resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type VARCHAR(64) NOT NULL,
    resource_id VARCHAR(255) NOT NULL,
    version_id INTEGER NOT NULL DEFAULT 1,
    resource JSONB NOT NULL,
    tenant_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,  -- soft delete for FHIR history
    created_by UUID,
    meta JSONB,  -- FHIR Meta (versionId, lastUpdated, profile, security, tag)
    UNIQUE (tenant_id, resource_type, resource_id, version_id)
  );
  ```
- [ ] Create resource-type-specific tables for high-volume types:
  - `fhir_patient` - Patient resources (most queried)
  - `fhir_encounter` - Encounter resources (high volume)
  - `fhir_observation` - Observation resources (highest volume)
  - `fhir_claim` - Claim resources (billing queries)
  - `fhir_condition` - Condition resources (clinical queries)
  - `fhir_procedure` - Procedure resources
  - `fhir_medication_request` - MedicationRequest resources
  - All inherit from base `fhir_resources` schema pattern
- [ ] Create GIN indexes on JSONB for common search paths:
  ```sql
  CREATE INDEX idx_patient_name ON fhir_patient USING GIN ((resource->'name'));
  CREATE INDEX idx_patient_identifier ON fhir_patient USING GIN ((resource->'identifier'));
  CREATE INDEX idx_patient_birthdate ON fhir_patient ((resource->>'birthDate'));
  CREATE INDEX idx_encounter_patient ON fhir_encounter ((resource->'subject'->>'reference'));
  CREATE INDEX idx_encounter_date ON fhir_encounter ((resource->'period'->>'start'));
  CREATE INDEX idx_observation_patient ON fhir_observation ((resource->'subject'->>'reference'));
  CREATE INDEX idx_observation_code ON fhir_observation USING GIN ((resource->'code'->'coding'));
  CREATE INDEX idx_claim_patient ON fhir_claim ((resource->'patient'->>'reference'));
  ```
- [ ] Create FHIR resource history table for versioning:
  ```sql
  CREATE TABLE fhir_resource_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id UUID NOT NULL REFERENCES fhir_resources(id),
    version_id INTEGER NOT NULL,
    resource JSONB NOT NULL,
    changed_by UUID,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    change_type VARCHAR(10) NOT NULL  -- CREATE, UPDATE, DELETE
  );
  ```
- [ ] Create database functions:
  - `upsert_fhir_resource(resource_type, resource_json)` - insert or update with version increment
  - `get_fhir_resource(resource_type, resource_id)` - get latest version
  - `get_fhir_resource_history(resource_type, resource_id)` - get all versions
  - `soft_delete_fhir_resource(resource_type, resource_id)` - set deleted_at
- [ ] Create migration files using a migration tool (e.g., `sqlx-migrate` or `refinery`)

**Acceptance Criteria:**
- [ ] Schema supports all FHIR R4 resource types required for MVP
- [ ] JSONB indexes provide < 50ms query time for common search patterns on 100K resources
- [ ] Resource versioning works correctly (version_id increments on update)
- [ ] Soft delete preserves history per FHIR spec
- [ ] Migrations run idempotently

---

### T2: Schema-per-Tenant Provisioning System
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T1, [[EPIC-002-auth-identity-system]] T9 (JWT tenant context)
**References:** [[ADR-002-multi-tenancy-schema-per-tenant]]

**Description:**
Implement the schema-per-tenant isolation model where each tenant (medical practice) gets their own PostgreSQL schema. The JWT tenant context from [[EPIC-002-auth-identity-system]] drives schema routing.

**Subtasks:**
- [ ] Create tenant management table (in `public` schema):
  ```sql
  CREATE TABLE public.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(63) NOT NULL UNIQUE,  -- used as schema name
    schema_name VARCHAR(63) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'provisioning',  -- provisioning, active, suspended, deactivated
    kms_key_arn TEXT,  -- per-tenant KMS key from EPIC-001 T4
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );
  ```
- [ ] Create tenant provisioning service:
  1. Insert tenant record in `public.tenants`
  2. Create PostgreSQL schema: `CREATE SCHEMA tenant_{slug}`
  3. Run all FHIR table migrations within tenant schema
  4. Create tenant-specific database role with schema-limited permissions
  5. Provision KMS key (call [[EPIC-001-aws-infrastructure-foundation]] T4)
  6. Seed reference data (FHIR CodeSystems, ValueSets)
  7. Update tenant status to `active`
- [ ] Implement schema routing middleware:
  ```
  On each request:
  1. Extract tenant_id from JWT (T9)
  2. Look up schema_name from tenants table (cached in Redis)
  3. SET search_path TO tenant_{slug}, public;
  4. Execute query
  5. RESET search_path
  ```
- [ ] Create tenant deprovisioning workflow:
  1. Set status to `deactivated`
  2. Export all data (FHIR $everything for each patient)
  3. Schedule KMS key deletion (30-day waiting period)
  4. Drop schema after key deletion
- [ ] Implement tenant settings API:
  - `POST /api/v1/tenants` - create tenant (super-admin only)
  - `GET /api/v1/tenants/:id` - get tenant details
  - `PATCH /api/v1/tenants/:id` - update settings
  - `DELETE /api/v1/tenants/:id` - deactivate tenant
- [ ] Connection pooling strategy: PgBouncer with schema routing support

**Acceptance Criteria:**
- [ ] New tenant provisioned in < 30 seconds (schema + tables + seed data)
- [ ] Queries in tenant-A cannot access tenant-B data (verified by test)
- [ ] Schema routing adds < 2ms overhead per request
- [ ] Tenant deprovisioning exports data and schedules key deletion
- [ ] 50+ concurrent tenants supported without connection pool exhaustion

---

### T3: FHIR Validation Layer
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1
**References:** [[FHIR-R4-Deep-Dive]], [[ADR-001-fhir-native-data-model]]

**Description:**
Implement FHIR resource validation against R4 base profiles and US Core profiles. Every resource written to the database must be structurally valid.

**Subtasks:**
- [ ] Implement FHIR R4 base validation:
  - Required fields present (resourceType, id on response)
  - Data type validation (dateTime, instant, code, Coding, Reference)
  - Reference integrity (referenced resources exist within tenant)
  - Cardinality enforcement (min/max occurrences)
- [ ] Implement US Core profile validation for MVP resources:
  - US Core Patient (race, ethnicity, birth sex extensions)
  - US Core Encounter
  - US Core Condition
  - US Core Procedure
  - US Core Observation (vitals, labs, social history)
  - US Core AllergyIntolerance
  - US Core MedicationRequest
- [ ] Create validation error response format:
  ```json
  {
    "resourceType": "OperationOutcome",
    "issue": [{
      "severity": "error",
      "code": "structure",
      "diagnostics": "Patient.name is required by US Core profile",
      "location": ["Patient.name"]
    }]
  }
  ```
- [ ] Implement validation modes:
  - `strict` - reject on any validation error (default for API writes)
  - `lenient` - accept with warnings (for data migration)
  - `none` - skip validation (internal system writes only)
- [ ] Cache profile definitions for performance
- [ ] Create validation test suite with valid and invalid resource examples

**Acceptance Criteria:**
- [ ] All required US Core elements enforced on writes
- [ ] Invalid resources return OperationOutcome with specific error locations
- [ ] Validation adds < 10ms per resource
- [ ] Lenient mode logs warnings but accepts resources
- [ ] Test suite covers 50+ validation scenarios

---

### T4: FHIR Resource CRUD API Endpoints
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T1, T2, T3, [[EPIC-002-auth-identity-system]] T2 (RBAC)
**References:** [[FHIR-R4-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Implement RESTful FHIR CRUD endpoints following the FHIR R4 HTTP API specification. Each endpoint must enforce tenant isolation and RBAC.

**Subtasks:**
- [ ] Implement FHIR interactions:
  | Interaction | Method | URL | Description |
  |---|---|---|---|
  | read | GET | `/fhir/{type}/{id}` | Read a resource |
  | vread | GET | `/fhir/{type}/{id}/_history/{vid}` | Read specific version |
  | update | PUT | `/fhir/{type}/{id}` | Update a resource |
  | patch | PATCH | `/fhir/{type}/{id}` | Partial update (JSON Patch) |
  | delete | DELETE | `/fhir/{type}/{id}` | Soft delete |
  | create | POST | `/fhir/{type}` | Create a resource |
  | search | GET | `/fhir/{type}?params` | Search resources |
  | history | GET | `/fhir/{type}/{id}/_history` | Resource history |
- [ ] Implement FHIR HTTP headers:
  - `Content-Type: application/fhir+json`
  - `ETag` for optimistic concurrency
  - `If-Match` for conditional updates
  - `If-None-Exist` for conditional creates
  - `Last-Modified`
  - `Location` header on create (201)
- [ ] Implement FHIR response codes:
  - 200 OK (read, search)
  - 201 Created (create)
  - 204 No Content (delete)
  - 400 Bad Request (validation error -> OperationOutcome)
  - 401 Unauthorized
  - 403 Forbidden (RBAC/ABAC denial)
  - 404 Not Found
  - 409 Conflict (version conflict on update)
  - 410 Gone (deleted resource)
  - 422 Unprocessable Entity (business rule violation)
- [ ] Integrate auth middleware:
  - Tenant isolation via search_path (from T2)
  - RBAC check per resource type and action (from [[EPIC-002-auth-identity-system]] T2)
  - ABAC check for patient-compartment resources (provider assignment)
- [ ] Implement CapabilityStatement endpoint:
  - `GET /fhir/metadata` - returns server capabilities
  - Lists supported resource types, interactions, search parameters
- [ ] Implement content negotiation (JSON only for MVP, XML stretch goal)

**Acceptance Criteria:**
- [ ] All CRUD operations work for Patient, Encounter, Condition, Observation, Claim
- [ ] Optimistic concurrency prevents lost updates (ETag/If-Match)
- [ ] RBAC enforced on every endpoint
- [ ] Tenant isolation verified (cross-tenant read returns 404)
- [ ] CapabilityStatement accurately reflects server capabilities
- [ ] API response times < 100ms for single resource operations

---

### T5: FHIR Search Parameter Implementation
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T4
**References:** [[FHIR-R4-Deep-Dive]]

**Description:**
Implement FHIR search functionality for the most commonly used search parameters. FHIR search is complex -- start with the parameters needed for clinical workflows.

**Subtasks:**
- [ ] Implement search parameter types:
  - `string` - case-insensitive, starts-with by default (name search)
  - `token` - code system|code (identifier, code searches)
  - `date` - date/dateTime comparisons (eq, lt, gt, ge, le)
  - `reference` - reference to another resource (subject, patient)
  - `number` - numeric comparisons
  - `quantity` - number with unit
  - `composite` - combination parameters
- [ ] Implement search parameters per resource:
  | Resource | Parameters |
  |---|---|
  | Patient | `name`, `family`, `given`, `identifier`, `birthdate`, `gender`, `phone`, `email`, `address`, `_id` |
  | Encounter | `patient`, `date`, `status`, `class`, `type`, `participant`, `_id` |
  | Condition | `patient`, `code`, `clinical-status`, `onset-date`, `category`, `_id` |
  | Observation | `patient`, `code`, `date`, `category`, `status`, `value-quantity`, `_id` |
  | Claim | `patient`, `created`, `status`, `provider`, `_id` |
  | Procedure | `patient`, `code`, `date`, `status`, `_id` |
  | MedicationRequest | `patient`, `medication`, `status`, `authoredon`, `_id` |
  | AllergyIntolerance | `patient`, `clinical-status`, `code`, `_id` |
- [ ] Implement search modifiers:
  - `:exact` - exact string match
  - `:contains` - substring match
  - `:missing` - parameter is absent
  - `:not` - negation
  - `:text` - text search on CodeableConcept
- [ ] Implement search result features:
  - `_count` - page size (default 20, max 100)
  - `_offset` - pagination offset
  - `_sort` - sort by search parameter
  - `_include` - include referenced resources
  - `_revinclude` - include resources that reference results
  - `_total` - include total count (optional, expensive)
  - `_summary` - return summary view
  - `_elements` - return specific elements only
- [ ] Implement Bundle response for search results:
  ```json
  {
    "resourceType": "Bundle",
    "type": "searchset",
    "total": 42,
    "link": [
      { "relation": "self", "url": "..." },
      { "relation": "next", "url": "..." }
    ],
    "entry": [{ "resource": {...}, "search": { "mode": "match" } }]
  }
  ```
- [ ] Build SQL query generator from FHIR search parameters (JSONB path queries)
- [ ] Implement search parameter combination (AND for different params, OR for same param)
- [ ] Add query execution plan logging for slow queries (> 200ms)

**Acceptance Criteria:**
- [ ] All listed search parameters functional
- [ ] Pagination works correctly with consistent results
- [ ] `_include` resolves references in single query
- [ ] Search response time < 200ms for typical queries (< 1000 results)
- [ ] SQL injection impossible (parameterized queries verified)
- [ ] Search on 100K+ resources performs within SLA

---

### T6: Patient Identity Matching Engine
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T4, T5
**References:** [[FHIR-R4-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Implement probabilistic patient matching to prevent duplicate patient records and support patient identity resolution across data sources. Critical for data migration and health information exchange.

**Subtasks:**
- [ ] Define matching attributes and weights:
  | Attribute | Weight | Match Type |
  |---|---|---|
  | SSN (last 4) | 0.25 | Exact |
  | Date of Birth | 0.20 | Exact |
  | Last Name | 0.15 | Phonetic (Soundex/Metaphone) |
  | First Name | 0.10 | Phonetic + nickname table |
  | Gender | 0.05 | Exact |
  | Phone | 0.10 | Normalized exact |
  | Address (ZIP) | 0.05 | Exact |
  | MRN | 0.10 | Exact (within source system) |
- [ ] Implement matching algorithm:
  1. Blocking: narrow candidates by DOB + first letter of last name
  2. Scoring: calculate weighted match score for each candidate
  3. Classification:
     - Score >= 0.85: AUTO-MERGE (definite match)
     - Score 0.65-0.84: REVIEW (probable match, needs human confirmation)
     - Score < 0.65: NO MATCH (create new patient)
- [ ] Create FHIR $match operation:
  ```
  POST /fhir/Patient/$match
  Content-Type: application/fhir+json

  {
    "resourceType": "Parameters",
    "parameter": [{
      "name": "resource",
      "resource": { "resourceType": "Patient", ... }
    }, {
      "name": "onlyCertainMatches",
      "valueBoolean": false
    }]
  }
  ```
- [ ] Implement patient merge workflow:
  - Master Patient Index (MPI) table tracking merged identities
  - FHIR Patient.link for merged records
  - Cascade updates to all referencing resources (Encounter.subject, etc.)
  - Audit trail for all merge operations
- [ ] Create duplicate review queue UI endpoint:
  - `GET /api/v1/patient-matches/pending` - probable matches awaiting review
  - `POST /api/v1/patient-matches/:id/merge` - confirm merge
  - `POST /api/v1/patient-matches/:id/dismiss` - not a match
- [ ] Implement nickname table (Robert/Bob, William/Bill, etc.)
- [ ] Name normalization: remove prefixes (Mr/Mrs/Dr), handle hyphens, Unicode

**Acceptance Criteria:**
- [ ] Matching algorithm identifies 95%+ of true duplicates in test dataset
- [ ] False positive rate < 5%
- [ ] $match operation returns results in < 500ms
- [ ] Merge operation updates all referencing resources
- [ ] Audit trail records every merge/dismiss decision
- [ ] Phonetic matching handles common name variations

---

### T7: FHIR Bundle Transaction Support
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T4
**References:** [[FHIR-R4-Deep-Dive]]

**Description:**
Implement FHIR Bundle transaction and batch processing. Transactions are atomic -- all-or-nothing. Batches process independently. Essential for clinical workflows where multiple resources are created together (e.g., Encounter + Conditions + Observations).

**Subtasks:**
- [ ] Implement transaction Bundle processing:
  ```
  POST /fhir
  Content-Type: application/fhir+json

  {
    "resourceType": "Bundle",
    "type": "transaction",
    "entry": [
      { "request": { "method": "POST", "url": "Encounter" }, "resource": {...} },
      { "request": { "method": "POST", "url": "Condition" }, "resource": {...} },
      { "request": { "method": "PUT", "url": "Patient/123" }, "resource": {...} }
    ]
  }
  ```
- [ ] Transaction processing rules:
  - Wrap in database transaction (BEGIN...COMMIT/ROLLBACK)
  - Process in order: DELETE, POST, PUT, GET
  - Resolve internal references (`urn:uuid:xxx` -> actual resource IDs)
  - If any entry fails, rollback entire transaction
  - Return transaction-response Bundle with individual outcomes
- [ ] Implement batch Bundle processing:
  - Each entry processed independently
  - Failed entries don't affect others
  - Return batch-response Bundle with individual outcomes
- [ ] Handle conditional references in transactions:
  - `Patient?identifier=MRN|12345` resolves to existing patient
  - Create if not found (conditional create)
- [ ] Implement transaction size limits:
  - Max 100 entries per transaction (configurable)
  - Max 1MB payload
- [ ] Performance: transaction with 50 entries completes in < 2s

**Acceptance Criteria:**
- [ ] Transaction is atomic (all succeed or all fail)
- [ ] Internal references (urn:uuid) resolve correctly
- [ ] Batch processes entries independently
- [ ] Conditional references resolve existing resources
- [ ] Transaction response includes all individual outcomes
- [ ] Size limits enforced with clear error messages

---

### T8: Event Emission System
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T4
**References:** [[System-Architecture-Overview]], [[ADR-003-ai-agent-framework]]

**Description:**
Implement an event system that emits events on FHIR resource changes. Downstream consumers (AI pipeline, billing, notifications) subscribe to these events.

**Subtasks:**
- [ ] Design event schema:
  ```json
  {
    "event_id": "uuid",
    "event_type": "fhir.resource.created",  // created, updated, deleted
    "resource_type": "Encounter",
    "resource_id": "uuid",
    "tenant_id": "uuid",
    "version_id": 1,
    "changed_by": "user-uuid",
    "timestamp": "2026-03-20T10:30:00Z",
    "resource": { ... },  // full resource (optional, configurable)
    "changes": { ... }    // diff from previous version (optional)
  }
  ```
- [ ] Implement event emission using PostgreSQL LISTEN/NOTIFY for real-time:
  - Trigger on INSERT/UPDATE/DELETE on fhir_resources tables
  - Notify channel per resource type: `fhir_patient_changes`, `fhir_encounter_changes`
- [ ] Implement event persistence for reliable delivery:
  - `event_outbox` table (transactional outbox pattern)
  - Event publisher reads outbox and publishes to SQS/SNS
  - Mark events as published after confirmed delivery
  - Retry failed deliveries with exponential backoff
- [ ] Create event subscription API:
  - `POST /api/v1/subscriptions` - create subscription (webhook URL + resource types + filters)
  - `GET /api/v1/subscriptions` - list subscriptions
  - `DELETE /api/v1/subscriptions/:id` - remove subscription
- [ ] Implement FHIR Subscription resource (R4):
  - Channel types: `rest-hook` (webhook), `websocket` (future)
  - Criteria filtering: `Encounter?status=finished`
- [ ] Configure event consumers for MVP:
  - AI pipeline: listen for `Encounter.status=finished` (trigger note generation)
  - Billing: listen for `Encounter.status=finished` (trigger charge capture)
  - Audit: listen for all events (compliance logging)

**Acceptance Criteria:**
- [ ] Events emitted for every CRUD operation on FHIR resources
- [ ] Event delivery is reliable (no lost events, verified by outbox)
- [ ] Consumers receive events within 5 seconds of change
- [ ] Event filtering works (consumers only receive subscribed events)
- [ ] Event system adds < 10ms overhead to write operations

---

### T9: pgvector Embeddings Pipeline
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1, [[EPIC-001-aws-infrastructure-foundation]] T6 (pgvector extension)
**References:** [[ADR-001-fhir-native-data-model]], [[ADR-003-ai-agent-framework]]

**Description:**
Build the pipeline that generates vector embeddings for clinical text (clinical notes, conditions, observations) and stores them in pgvector for semantic search and AI context retrieval.

**Subtasks:**
- [ ] Create embeddings table:
  ```sql
  CREATE TABLE fhir_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type VARCHAR(64) NOT NULL,
    resource_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    chunk_index INTEGER NOT NULL DEFAULT 0,  -- for long texts split into chunks
    text_content TEXT NOT NULL,  -- original text that was embedded
    embedding vector(1536) NOT NULL,  -- OpenAI ada-002 / 1536 dimensions
    model_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (resource_id, chunk_index)
  );
  CREATE INDEX idx_embeddings_vector ON fhir_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
  CREATE INDEX idx_embeddings_resource ON fhir_embeddings (resource_type, resource_id);
  CREATE INDEX idx_embeddings_tenant ON fhir_embeddings (tenant_id);
  ```
- [ ] Implement embedding generation pipeline:
  1. Listen for FHIR resource events (T8) for text-containing resources
  2. Extract text content:
     - Encounter: note narrative
     - Condition: code display + note
     - Observation: value + interpretation + note
     - DiagnosticReport: conclusion + presentedForm text
  3. Chunk long texts (max 512 tokens per chunk with 50 token overlap)
  4. Generate embeddings via API (OpenAI ada-002 or self-hosted model)
  5. Store in fhir_embeddings table
- [ ] Implement semantic search API:
  ```
  POST /api/v1/search/semantic
  {
    "query": "patient with chest pain and elevated troponin",
    "resource_types": ["Encounter", "Condition", "Observation"],
    "limit": 10,
    "similarity_threshold": 0.75
  }
  ```
  ```sql
  SELECT resource_type, resource_id, text_content,
         1 - (embedding <=> $1) as similarity
  FROM fhir_embeddings
  WHERE tenant_id = $2
    AND resource_type = ANY($3)
    AND 1 - (embedding <=> $1) > $4
  ORDER BY embedding <=> $1
  LIMIT $5;
  ```
- [ ] Implement batch embedding for existing data (migration tool)
- [ ] Implement embedding cache to avoid re-embedding unchanged text
- [ ] Monitor embedding API costs and implement rate limiting

**Acceptance Criteria:**
- [ ] New clinical text resources are auto-embedded within 30 seconds
- [ ] Semantic search returns relevant results with cosine similarity scoring
- [ ] Search latency < 200ms for queries across 100K embeddings
- [ ] Chunking handles long clinical narratives correctly
- [ ] Cost monitoring in place for embedding API calls

---

### T10: Data Migration Tooling
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T4, T6, T7
**References:** [[System-Architecture-Overview]]

**Description:**
Build tooling to import existing patient data from practices joining MedOS. Practices may export from other EHRs in FHIR, CSV, HL7v2, or CCDAformats.

**Subtasks:**
- [ ] Create migration framework:
  ```
  migration/
  ├── importers/
  │   ├── fhir_bundle.rs      # FHIR Bundle JSON import
  │   ├── csv_patient.rs      # CSV patient demographics
  │   ├── ccda.rs             # C-CDA XML import (stretch)
  │   └── hl7v2.rs            # HL7v2 message import (stretch)
  ├── transformers/
  │   ├── normalize.rs        # Normalize names, phones, addresses
  │   ├── code_mapping.rs     # Map local codes to standard terminologies
  │   └── dedup.rs            # Duplicate detection using T6
  ├── validators/
  │   └── pre_import.rs       # Validate before import (lenient mode)
  └── reports/
      └── migration_report.rs # Generate import summary
  ```
- [ ] Implement FHIR Bundle importer:
  - Accept FHIR Bundle (type: collection or transaction)
  - Validate each resource (lenient mode - accept with warnings)
  - Run patient matching (T6) to detect duplicates
  - Import as FHIR transaction (T7)
  - Generate migration report
- [ ] Implement CSV patient importer:
  - Standard column mapping (configurable)
  - Transform to FHIR Patient resources
  - Validate US Core compliance
  - Import via FHIR API
- [ ] Create migration CLI:
  ```bash
  medos-migrate import \
    --tenant-id UUID \
    --source fhir-bundle \
    --file patients.json \
    --mode lenient \
    --dry-run
  ```
- [ ] Implement dry-run mode (validate and report without writing)
- [ ] Implement rollback support (tag imported resources for bulk deletion)
- [ ] Generate migration report:
  - Total records processed
  - Successfully imported
  - Failed (with reasons)
  - Duplicates detected (with match scores)
  - Warnings issued

**Acceptance Criteria:**
- [ ] FHIR Bundle import processes 1000 patients in < 5 minutes
- [ ] CSV import maps columns to FHIR Patient correctly
- [ ] Dry-run mode validates without side effects
- [ ] Migration report provides actionable information
- [ ] Duplicate detection prevents patient record duplication
- [ ] Rollback removes all imported resources cleanly

---

### T11: FHIR $everything Operation
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T4, T5
**References:** [[FHIR-R4-Deep-Dive]], [[HIPAA-Deep-Dive]]

**Description:**
Implement the FHIR Patient/$everything operation that returns all data associated with a patient. Required for HIPAA patient data access requests and for data portability.

**Subtasks:**
- [ ] Implement `GET /fhir/Patient/{id}/$everything`:
  - Returns a Bundle containing:
    - Patient resource
    - All Encounters
    - All Conditions
    - All Observations
    - All Procedures
    - All MedicationRequests
    - All AllergyIntolerances
    - All Immunizations
    - All DiagnosticReports
    - All Claims (if requestor has billing permission)
    - All Provenance resources
  - Support `_since` parameter (only resources updated since date)
  - Support `_type` parameter (filter by resource type)
  - Support `_count` parameter (pagination)
- [ ] Implement async export for large patient records:
  - If estimated response > 10MB, return 202 Accepted
  - Generate export file in background
  - Store in S3 (tenant bucket, encrypted with tenant KMS key)
  - Return content-location header for polling
  - `GET /api/v1/exports/{id}/status` - check export status
  - `GET /api/v1/exports/{id}/download` - download export file (pre-signed S3 URL)
- [ ] Implement FHIR Bulk Data Export (FHIR $export):
  - `GET /fhir/$export` - all data for tenant
  - `GET /fhir/Patient/$export` - all patients
  - Output format: NDJSON (newline-delimited JSON)
  - Async processing with status polling
- [ ] HIPAA compliance:
  - Log every $everything request in audit trail
  - Patient can request their own data (patient portal)
  - Generate human-readable summary alongside FHIR export
- [ ] Rate limiting: max 1 $everything per patient per hour

**Acceptance Criteria:**
- [ ] $everything returns all patient-compartment resources
- [ ] Pagination works for patients with large records
- [ ] Async export handles records > 10MB
- [ ] Bulk export generates valid NDJSON files
- [ ] Audit trail captures every export request
- [ ] Export completes within 60 seconds for typical patient (< 1000 resources)

---

## Dependencies Map

```
T1 (FHIR Schema) ──> T2 (Tenant Provisioning) ──> T4 (CRUD API) ──> T5 (Search)
                 └──> T3 (Validation) ──────────────┘              └──> T11 ($everything)
                 └──> T9 (pgvector)                                └──> T6 (Patient Matching)
                                                    T4 ──> T7 (Bundles)
                                                    T4 ──> T8 (Events) ──> T9 (pgvector trigger)
                                                    T4 + T6 + T7 ──> T10 (Migration)
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| FHIR CRUD API (T4) | [[EPIC-004-ai-clinical-documentation]] (write generated resources) |
| Event emission (T8) | [[EPIC-004-ai-clinical-documentation]] (trigger AI on encounter close) |
| pgvector search (T9) | [[EPIC-004-ai-clinical-documentation]] (retrieve clinical context) |
| Patient matching (T6) | [[EPIC-006-pilot-readiness]] (data migration for pilot practices) |
| FHIR search (T5) | [[EPIC-005-revenue-cycle-mvp]] (find claims, encounters) |
| Claims resource (T4) | [[EPIC-005-revenue-cycle-mvp]] (FHIR Claim CRUD) |
| $everything (T11) | [[EPIC-006-pilot-readiness]] (patient data export, HIPAA compliance) |
| Bundle transactions (T7) | [[EPIC-004-ai-clinical-documentation]] (atomic write of generated resources) |
| Data migration (T10) | [[EPIC-006-pilot-readiness]] (onboard pilot practices) |

| This Epic Requires | Provided By |
|---|---|
| RDS PostgreSQL with pgvector | [[EPIC-001-aws-infrastructure-foundation]] T6 |
| VPC + private subnets | [[EPIC-001-aws-infrastructure-foundation]] T3 |
| KMS per-tenant keys | [[EPIC-001-aws-infrastructure-foundation]] T4 |
| JWT with tenant context | [[EPIC-002-auth-identity-system]] T9 |
| RBAC permission checks | [[EPIC-002-auth-identity-system]] T2 |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| FHIR search query complexity causes slow queries | High | High | Start with most-used parameters; add indexes based on query patterns; query plan logging |
| Schema-per-tenant causes connection pool exhaustion at scale | Medium | Critical | PgBouncer with transaction-mode pooling; connection limit per tenant; horizontal sharding plan for 100+ tenants |
| pgvector index rebuild time grows with data volume | Medium | Medium | Use IVFFlat with appropriate lists parameter; schedule rebuilds during off-hours |
| Patient matching false positives cause data corruption | Medium | Critical | Default to REVIEW (not auto-merge); require human confirmation for uncertain matches; audit trail for all merges |
| FHIR validation too strict for real-world data | High | Medium | Lenient mode for migration; clear error messages; iterative profile refinement |
| Embedding API cost at scale (1M+ clinical texts) | Medium | Medium | Batch processing; cache unchanged texts; evaluate self-hosted embedding models |

---

## Definition of Done

- [ ] All MVP FHIR resource types have CRUD + search + validation
- [ ] Schema-per-tenant isolation verified with cross-tenant security tests
- [ ] Patient matching identifies duplicates with > 95% recall
- [ ] FHIR Bundle transactions are atomic
- [ ] Events emitted for all resource changes with reliable delivery
- [ ] pgvector semantic search returns relevant clinical results
- [ ] Data migration imports FHIR Bundles and CSV files
- [ ] $everything exports all patient data for HIPAA requests
- [ ] Performance: < 100ms CRUD, < 200ms search, < 500ms $match
- [ ] All endpoints documented in OpenAPI spec and FHIR CapabilityStatement
