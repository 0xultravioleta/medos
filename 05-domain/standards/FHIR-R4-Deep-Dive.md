---
type: domain-knowledge
date: "2026-02-27"
tags:
  - domain
  - standards
  - fhir
  - module-a
  - module-h
category: standards
confidence: high
sources: []
---

# FHIR R4 Deep Dive

> Comprehensive reference for the Fast Healthcare Interoperability Resources (FHIR) R4 standard as it applies to [[HEALTHCARE_OS_MASTERPLAN]]. This document covers the data model, operations, integration patterns, and practical implementation details MedOS needs to become a FHIR-native Healthcare OS.

Related notes: [[MOC-Architecture]], [[HIPAA-Deep-Dive]], [[X12-EDI-Deep-Dive]], [[Prior-Authorization-Deep-Dive]]

---

## 1. What is FHIR R4

FHIR (Fast Healthcare Interoperability Resources, pronounced "fire") is the dominant standard for exchanging healthcare data electronically. Published by HL7 International, FHIR R4 (Release 4, v4.0.1) is the first **normatively stable** release -- meaning the core Patient, Observation, and other foundational resources have a formal stability guarantee and will not have breaking changes in future versions.

**Why FHIR matters to us:**
- **CMS mandates it.** The CMS Interoperability and Patient Access Final Rule (CMS-9115-F), the CMS-0057 rule (Advancing Interoperability), and ONC HTI-1 all require FHIR R4 APIs for payer-to-payer exchange, patient access, and provider directory access.
- **Every major EHR supports it.** Epic, Cerner (Oracle Health), MEDITECH, athenahealth, and Allscripts all expose FHIR R4 endpoints.
- **It is RESTful JSON.** Unlike older HL7v2 (pipe-delimited) or CDA (XML-heavy), FHIR uses standard REST + JSON, making it accessible to modern web developers.
- **R4 is the market standard.** Per the 2024 State of FHIR survey, 22 out of 38 respondents use FHIR R4 as their primary version. R5 exists but adoption is nascent.

**Key concepts for developers new to healthcare:**

| Concept | What it means |
|---------|--------------|
| **Resource** | A discrete unit of healthcare data (Patient, Claim, Observation, etc.). Think of it as a domain entity with a defined JSON schema. |
| **Reference** | A pointer from one resource to another, like a foreign key. Example: `"subject": {"reference": "Patient/123"}` |
| **CodeableConcept** | A value from a terminology system (ICD-10, SNOMED CT, CPT, LOINC). Contains `system`, `code`, and `display`. |
| **Bundle** | A collection of resources, used for search results, transactions, and batch operations. |
| **CapabilityStatement** | A machine-readable declaration of what a FHIR server supports (which resources, which operations, which search parameters). |
| **Profile** | A set of constraints on a base resource. US Core profiles add required fields (e.g., US Core Patient requires `race` and `ethnicity` extensions). |
| **Extension** | A mechanism to add fields not in the base spec. Identified by a URL. |

---

## 2. Core Resources for MedOS

These are the FHIR resources MedOS must support across Module A (Administrative/Claims) and Module H (Clinical/Health Records).

### 2.1 Patient

**What it represents:** Demographics and administrative information about an individual receiving care.

**Key fields:**
- `identifier[]` -- MRN, SSN, payer member ID. Each has a `system` (namespace URI) and `value`.
- `name[]` -- `family`, `given[]`, `use` (official, nickname, old).
- `gender` -- `male | female | other | unknown` (administrative gender, not clinical sex).
- `birthDate` -- ISO 8601 date.
- `address[]` -- `line[]`, `city`, `state`, `postalCode`, `country`.
- `telecom[]` -- phone, email, fax with `system` and `use`.
- `communication[]` -- preferred languages.
- `generalPractitioner` -- Reference to Practitioner or Organization.

**When we use it:** Every interaction. Patient is the anchor resource -- nearly every other resource references it. In MedOS, a Patient maps to a "member" in the payer context and a "patient" in the clinical context.

### 2.2 Encounter

**What it represents:** An interaction between a patient and healthcare provider(s) for the purpose of providing healthcare services or assessing health status. Covers inpatient stays, outpatient visits, ER visits, telehealth sessions.

**Key fields:**
- `status` -- `planned | arrived | triaged | in-progress | onleave | finished | cancelled`
- `class` -- `AMB` (ambulatory), `EMER` (emergency), `IMP` (inpatient), `VR` (virtual)
- `type[]` -- CodeableConcept describing the encounter type
- `subject` -- Reference to Patient
- `participant[]` -- practitioners involved, with roles
- `period` -- `start` and `end` timestamps
- `diagnosis[]` -- conditions addressed, with rank
- `hospitalization` -- admit/discharge details
- `serviceProvider` -- Reference to Organization
- `location[]` -- where the encounter occurred

**When we use it:** Linking clinical events to a specific visit. Claims reference Encounters. Prior auth requests reference planned Encounters.

### 2.3 Condition

**What it represents:** A clinical condition, problem, diagnosis, or other health-related event/situation.

**Key fields:**
- `clinicalStatus` -- `active | recurrence | relapse | inactive | remission | resolved`
- `verificationStatus` -- `unconfirmed | provisional | differential | confirmed | refuted | entered-in-error`
- `category[]` -- `problem-list-item` or `encounter-diagnosis`
- `code` -- CodeableConcept using ICD-10-CM or SNOMED CT
- `subject` -- Reference to Patient
- `encounter` -- Reference to Encounter
- `onsetDateTime` / `abatementDateTime`
- `recorder` -- who recorded it

**When we use it:** Diagnosis lists for claims, clinical decision support, risk adjustment (HCC coding), care gap analysis.

### 2.4 Procedure

**What it represents:** An action performed on or for a patient for diagnostic or therapeutic purposes.

**Key fields:**
- `status` -- `preparation | in-progress | not-done | on-hold | stopped | completed`
- `code` -- CPT, HCPCS, or SNOMED CT code
- `subject` -- Reference to Patient
- `encounter` -- Reference to Encounter
- `performedDateTime` / `performedPeriod`
- `performer[]` -- who performed it
- `reasonCode[]` / `reasonReference[]` -- why it was done (links to Condition)
- `bodySite[]` -- anatomical location

**When we use it:** Procedure details on claims, surgical history, prior authorization requirements.

### 2.5 Claim

**What it represents:** A request for payment or adjudication of healthcare services. Can represent professional, institutional, oral, pharmacy, or vision claims.

**Key fields:**
- `status` -- `active | cancelled | draft | entered-in-error`
- `type` -- `institutional | oral | pharmacy | professional | vision`
- `use` -- `claim | preauthorization | predetermination`
- `patient` -- Reference to Patient
- `billablePeriod` -- service date range
- `created` -- when the claim was created
- `provider` -- Reference to Practitioner or Organization
- `priority` -- urgency
- `insurance[]` -- Coverage references, `focal` flag for primary
- `diagnosis[]` -- ICD-10 codes with `sequence` and `type` (admitting, principal, etc.)
- `procedure[]` -- procedures performed
- `item[]` -- line items with `productOrService` (CPT/HCPCS), `quantity`, `unitPrice`, `net`
- `total` -- total claim charge

**When we use it:** Submitting claims for adjudication, prior authorization requests (`use: "preauthorization"`), cost estimation (`use: "predetermination"`). This is the core of Module A.

### 2.6 Coverage

**What it represents:** A patient's insurance coverage -- the link between a patient and their health plan.

**Key fields:**
- `status` -- `active | cancelled | draft | entered-in-error`
- `type` -- group, individual, Medicare, Medicaid, etc.
- `subscriber` -- Reference to Patient (the policyholder)
- `beneficiary` -- Reference to Patient (the covered individual)
- `dependent` -- dependent number
- `relationship` -- `self | spouse | child | other`
- `period` -- coverage effective dates
- `payor[]` -- Reference to Organization (the insurance company)
- `class[]` -- group, plan, subplan, class, subclass identifiers
- `network` -- network name
- `costToBeneficiary[]` -- copays, coinsurance

**When we use it:** Eligibility verification, claims submission (every Claim references Coverage), member enrollment tracking.

### 2.7 ExplanationOfBenefit (EOB)

**What it represents:** The result of a claim adjudication. Combines information from Claim + ClaimResponse + Coverage to tell the patient/provider what was paid, denied, or adjusted.

**Key fields:**
- `status` -- `active | cancelled | draft | entered-in-error`
- `type` -- same as Claim type
- `use` -- `claim | preauthorization | predetermination`
- `patient` -- Reference to Patient
- `billablePeriod`
- `created`
- `insurer` -- Reference to Organization
- `provider` -- Reference to Practitioner/Organization
- `outcome` -- `queued | complete | error | partial`
- `insurance[]` -- Coverage references
- `item[]` -- line items with `adjudication[]`:
  - Each adjudication has a `category` (submitted, eligible, deductible, copay, benefit) and `amount`
- `total[]` -- summary amounts by category
- `payment` -- amount paid, date, to whom
- `processNote[]` -- adjudicator notes

**When we use it:** This is the single most important resource for MedOS Module A. Patient access APIs (CMS mandate) serve EOBs. CARIN Blue Button profiles define the exact shape. Every adjudicated claim produces an EOB.

### 2.8 Observation

**What it represents:** Measurements, assessments, and simple assertions about a patient. Covers lab results, vital signs, social history, surveys.

**Key fields:**
- `status` -- `registered | preliminary | final | amended | corrected | cancelled`
- `category[]` -- `vital-signs | laboratory | social-history | survey | exam`
- `code` -- LOINC code (e.g., `8867-4` for heart rate)
- `subject` -- Reference to Patient
- `effectiveDateTime`
- `valueQuantity` -- `value`, `unit`, `system`, `code` (UCUM units)
- `valueCodeableConcept` -- for coded results
- `interpretation[]` -- `H` (high), `L` (low), `N` (normal)
- `referenceRange[]` -- normal ranges
- `component[]` -- for multi-component observations (e.g., blood pressure has systolic + diastolic)

**When we use it:** Lab results, vitals, SDOH screening scores, clinical measurements for care gap analysis and quality measures.

### 2.9 DiagnosticReport

**What it represents:** A collection of observations and interpretations -- the "report" from a lab, radiology, pathology, etc.

**Key fields:**
- `status` -- `registered | partial | preliminary | final | amended | corrected | appended | cancelled`
- `category[]` -- `LAB | RAD | PAT` etc.
- `code` -- LOINC or local code for the report type
- `subject` -- Reference to Patient
- `encounter` -- Reference to Encounter
- `effectiveDateTime`
- `issued` -- when the report was released
- `performer[]` -- who issued the report
- `result[]` -- References to Observation resources
- `presentedForm[]` -- attached PDF/document

**When we use it:** Aggregating lab panels, radiology reports, pathology results. DiagnosticReport groups related Observations.

### 2.10 MedicationRequest

**What it represents:** An order or prescription for medication for a patient.

**Key fields:**
- `status` -- `active | on-hold | cancelled | completed | entered-in-error | stopped | draft`
- `intent` -- `proposal | plan | order | original-order | reflex-order | filler-order | instance-order`
- `medicationCodeableConcept` or `medicationReference` -- RxNorm code or Reference to Medication
- `subject` -- Reference to Patient
- `encounter` -- Reference to Encounter
- `authoredOn` -- when prescribed
- `requester` -- who prescribed
- `dosageInstruction[]` -- timing, route, dose
- `dispenseRequest` -- quantity, refills, validity period
- `substitution` -- whether generic substitution is allowed

**When we use it:** Medication history, pharmacy claims reconciliation, drug interaction checks, formulary management.

### 2.11 AllergyIntolerance

**What it represents:** A risk of harmful or undesirable physiological response to a substance.

**Key fields:**
- `clinicalStatus` -- `active | inactive | resolved`
- `verificationStatus` -- `unconfirmed | confirmed | refuted | entered-in-error`
- `type` -- `allergy | intolerance`
- `category[]` -- `food | medication | environment | biologic`
- `criticality` -- `low | high | unable-to-assess`
- `code` -- the allergen (SNOMED CT or RxNorm)
- `patient` -- Reference to Patient
- `reaction[]` -- manifestation, severity, substance

**When we use it:** Patient safety, clinical decision support, medication checks.

### 2.12 Immunization

**What it represents:** A record of a vaccination event.

**Key fields:**
- `status` -- `completed | entered-in-error | not-done`
- `vaccineCode` -- CVX code
- `patient` -- Reference to Patient
- `occurrenceDateTime`
- `lotNumber`, `expirationDate`
- `site`, `route`
- `performer[]` -- who administered
- `reasonCode[]` -- why given
- `protocolApplied[]` -- series, dose number

**When we use it:** Immunization records, compliance tracking, public health reporting.

### 2.13 Provenance

**What it represents:** A record of who did what to which resource, when, and why. The audit trail of data lineage.

**Key fields:**
- `target[]` -- References to the resources this provenance applies to
- `recorded` -- timestamp
- `agent[]` -- who was involved (`type`: author, performer, verifier, etc.), with `who` reference
- `entity[]` -- entities used in the activity (source documents, derived-from references)
- `signature[]` -- digital signatures

**When we use it:** [[HIPAA-Deep-Dive]] compliance. Every resource we create or modify should have a Provenance record. Critical for audit trails and data lineage.

### 2.14 AuditEvent

**What it represents:** A record of a security-relevant event -- who accessed what, when, from where.

**Key fields:**
- `type` -- event type (rest, export, login, etc.)
- `subtype[]` -- CRUD operation type
- `action` -- `C | R | U | D | E` (Create, Read, Update, Delete, Execute)
- `recorded` -- timestamp
- `outcome` -- `0` (success) through `12` (major failure)
- `agent[]` -- who did it (user, system), with network info
- `source` -- the system that recorded the event
- `entity[]` -- what was accessed (Patient reference, resource type)

**When we use it:** HIPAA audit logging, security monitoring, compliance reporting. Every API call to MedOS should generate an AuditEvent.

---

## 3. FHIR Operations

### 3.1 CRUD Operations (RESTful)

FHIR uses standard HTTP verbs mapped to resource lifecycle:

```
# Create a new Patient
POST /fhir/Patient
Content-Type: application/fhir+json

{...patient resource...}
# Returns: 201 Created, Location header with new ID

# Read a Patient
GET /fhir/Patient/123
# Returns: 200 OK with the resource

# Update a Patient (full replacement)
PUT /fhir/Patient/123
Content-Type: application/fhir+json

{...complete patient resource with id: "123"...}
# Returns: 200 OK

# Patch a Patient (partial update)
PATCH /fhir/Patient/123
Content-Type: application/json-patch+json

[{"op": "replace", "path": "/birthDate", "value": "1990-01-15"}]

# Delete a Patient
DELETE /fhir/Patient/123
# Returns: 204 No Content (logical delete -- resource still exists but marked deleted)

# Version Read (get specific version)
GET /fhir/Patient/123/_history/2

# History (all versions)
GET /fhir/Patient/123/_history
```

### 3.2 Search

Search is the most complex and powerful part of the FHIR API. Searches return a Bundle of type `searchset`.

```
# Search by identifier
GET /fhir/Patient?identifier=http://hospital.org/mrn|12345

# Search by name
GET /fhir/Patient?name=Smith&birthdate=1990-01-15

# Search by date range
GET /fhir/ExplanationOfBenefit?created=ge2025-01-01&created=le2025-12-31

# Search with include (pull in referenced resources)
GET /fhir/ExplanationOfBenefit?patient=Patient/123&_include=ExplanationOfBenefit:provider

# Search with revinclude (pull in resources that reference these)
GET /fhir/Patient?_id=123&_revinclude=Condition:subject

# Chained search (search by referenced resource fields)
GET /fhir/Observation?subject.name=Smith

# Token search (code systems)
GET /fhir/Condition?code=http://hl7.org/fhir/sid/icd-10-cm|E11.9

# Composite search parameters
GET /fhir/Observation?code-value-quantity=http://loinc.org|8867-4$gt60

# Pagination
GET /fhir/Patient?_count=50&_offset=100

# Sort
GET /fhir/Patient?_sort=-birthdate,name

# Summary (return only summary fields)
GET /fhir/Patient?_summary=true

# Elements (return only specific fields)
GET /fhir/Patient?_elements=name,birthDate,identifier
```

**Search parameter types:** `string`, `token`, `date`, `reference`, `quantity`, `uri`, `number`, `composite`, `special`.

**Modifiers:** `:exact`, `:contains`, `:not`, `:missing`, `:above`, `:below`, `:text`, `:of-type`.

**Prefixes for date/quantity:** `eq`, `ne`, `gt`, `lt`, `ge`, `le`, `sa` (starts after), `eb` (ends before), `ap` (approximately).

### 3.3 $everything Operation

Returns all data related to a patient in a single Bundle:

```
GET /fhir/Patient/123/$everything
GET /fhir/Patient/123/$everything?start=2025-01-01&end=2025-12-31
GET /fhir/Patient/123/$everything?_type=Condition,MedicationRequest,AllergyIntolerance
```

This is expensive and should be used carefully. It is useful for patient data export and payer-to-payer exchange.

### 3.4 Bulk Data Export

For large-scale data extraction (population health, analytics, reporting). Uses the FHIR Async Request Pattern:

```
# Kick off export
GET /fhir/Patient/$export
Prefer: respond-async
Accept: application/fhir+ndjson

# Server responds with 202 Accepted + Content-Location header
# Poll the status URL
GET /fhir/$export-poll-status/abc123

# When complete, server returns 200 with download links
{
  "transactionTime": "2026-02-27T00:00:00Z",
  "request": "/fhir/Patient/$export",
  "requiresAccessToken": true,
  "output": [
    {"type": "Patient", "url": "https://server/download/patients.ndjson"},
    {"type": "Condition", "url": "https://server/download/conditions.ndjson"}
  ]
}
```

Output format is NDJSON (newline-delimited JSON) -- one resource per line. This is critical for MedOS analytics pipelines and data ingestion.

### 3.5 Subscriptions

FHIR R4 Subscriptions enable real-time notifications when resources change. R4 uses a channel-based model:

```json
{
  "resourceType": "Subscription",
  "status": "requested",
  "criteria": "Observation?code=http://loinc.org|8867-4",
  "channel": {
    "type": "rest-hook",
    "endpoint": "https://medos.example.com/webhooks/fhir",
    "payload": "application/fhir+json",
    "header": ["Authorization: Bearer {{token}}"]
  }
}
```

Channel types: `rest-hook`, `websocket`, `email`, `message`. For MedOS, `rest-hook` is the primary pattern -- the FHIR server POSTs notifications to our webhook endpoint when matching resources are created or updated.

> **Note:** FHIR R5 introduces a significantly redesigned Subscription model using `SubscriptionTopic` resources. Plan for migration when R5 adoption grows.

---

## 4. FHIR Profiles and US Core

### What is a Profile?

A Profile is a StructureDefinition that constrains a base FHIR resource. It can:
- Make optional fields required (`0..1` becomes `1..1`)
- Restrict value sets (e.g., must use ICD-10-CM, not ICD-9)
- Add must-support flags (implementers MUST be able to store and return these fields)
- Add extensions (new fields specific to a jurisdiction or use case)
- Remove unused fields

### US Core Implementation Guide

US Core is THE foundational profile set for the United States. It implements USCDI (U.S. Core Data for Interoperability) requirements.

**Current versions in play:**
- **US Core 3.1.1** -- baseline for CMS Patient Access API
- **US Core 6.1.0** -- required by ONC HTI-1 as of January 1, 2026
- **US Core 7.0.0** -- latest, supported by Da Vinci PDex 2.1.1

**What US Core mandates (key profiles):**

| Profile | Must-Support Fields Added |
|---------|--------------------------|
| US Core Patient | `race` extension, `ethnicity` extension, `birthsex` extension, at least one `identifier`, at least one `name` |
| US Core Condition | Must use SNOMED CT or ICD-10-CM for `code` |
| US Core Observation (Lab) | Must use LOINC for `code`, UCUM for units |
| US Core Vital Signs | Follows FHIR core vital signs profiles, LOINC codes |
| US Core Medication | RxNorm required |
| US Core Procedure | CPT, SNOMED CT, or HCPCS for `code` |
| US Core Encounter | Must include `type`, `status`, `class` |

**Why it matters for compliance:** If MedOS exposes FHIR APIs (and it must, per CMS mandates), those APIs must conform to US Core profiles. Payer-to-payer data exchange, patient access APIs, and provider directory APIs all require US Core compliance. Non-compliance means failing ONC certification and CMS audits.

### Must-Support Semantics

"Must-support" in US Core does NOT mean "must always have a value." It means:
- **Senders** must include the element if they have it
- **Receivers** must be able to handle the element without breaking
- Missing must-support elements should be interpreted as "not available," not "false" or "none"

---

## 5. FHIR-Native Data Model

### Why Store Natively as FHIR JSONB

Traditional approach: receive FHIR, translate to relational tables, translate back to FHIR on output. This creates:
- Lossy translations (extensions get dropped, edge cases get mangled)
- Impedance mismatch (FHIR's polymorphic types do not map cleanly to columns)
- Double maintenance (update FHIR mapping AND database schema for every change)
- Profile drift (your internal model diverges from the standard)

**MedOS approach:** Store FHIR resources as-is in PostgreSQL JSONB columns. The resource IS the source of truth.

### PostgreSQL + JSONB Pattern

```sql
-- Core resource table (one table per resource type, following FHIRbase pattern)
CREATE TABLE fhir_patient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fhir_id         TEXT UNIQUE NOT NULL,          -- FHIR logical ID
    version_id      INTEGER NOT NULL DEFAULT 1,
    resource        JSONB NOT NULL,                -- The complete FHIR resource
    resource_type   TEXT NOT NULL DEFAULT 'Patient',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    tenant_id       UUID NOT NULL                  -- Multi-tenancy for payer orgs
);

-- History table for versioning
CREATE TABLE fhir_patient_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fhir_id         TEXT NOT NULL,
    version_id      INTEGER NOT NULL,
    resource        JSONB NOT NULL,
    resource_type   TEXT NOT NULL DEFAULT 'Patient',
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tenant_id       UUID NOT NULL,
    UNIQUE(fhir_id, version_id, tenant_id)
);

-- Same pattern for other resource types
CREATE TABLE fhir_claim (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fhir_id         TEXT UNIQUE NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    resource        JSONB NOT NULL,
    resource_type   TEXT NOT NULL DEFAULT 'Claim',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    tenant_id       UUID NOT NULL
);

CREATE TABLE fhir_explanationofbenefit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fhir_id         TEXT UNIQUE NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    resource        JSONB NOT NULL,
    resource_type   TEXT NOT NULL DEFAULT 'ExplanationOfBenefit',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    tenant_id       UUID NOT NULL
);
```

### pgvector for Clinical Embeddings

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Clinical embeddings table
CREATE TABLE clinical_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   TEXT NOT NULL,
    resource_id     TEXT NOT NULL,
    tenant_id       UUID NOT NULL,
    chunk_text      TEXT NOT NULL,                  -- The text that was embedded
    chunk_index     INTEGER NOT NULL DEFAULT 0,     -- Position within the resource
    embedding       vector(1536) NOT NULL,          -- OpenAI ada-002 dimensionality
    metadata        JSONB,                          -- Additional context
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(resource_type, resource_id, chunk_index, tenant_id)
);

-- HNSW index for fast similarity search (better recall than IVFFlat, no training needed)
CREATE INDEX idx_clinical_embeddings_hnsw
    ON clinical_embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Example: semantic search for clinical notes
-- "Find patients with symptoms similar to this description"
SELECT resource_id, resource_type, chunk_text,
       1 - (embedding <=> $1::vector) AS similarity
FROM clinical_embeddings
WHERE tenant_id = $2
  AND resource_type = 'DiagnosticReport'
ORDER BY embedding <=> $1::vector
LIMIT 20;
```

### Indexing Strategies for FHIR Search Parameters

```sql
-- GIN index on entire JSONB resource (supports @>, ?, ?| operators)
CREATE INDEX idx_patient_resource_gin ON fhir_patient USING GIN (resource);

-- Targeted indexes for common FHIR search parameters
-- Patient.identifier (token search)
CREATE INDEX idx_patient_identifier ON fhir_patient USING GIN (
    (resource -> 'identifier') jsonb_path_ops
);

-- Patient.name (string search -- extract for btree)
CREATE INDEX idx_patient_family_name ON fhir_patient (
    (resource #>> '{name,0,family}')
);

-- Patient.birthDate (date search)
CREATE INDEX idx_patient_birthdate ON fhir_patient (
    (resource ->> 'birthDate')
);

-- ExplanationOfBenefit.patient (reference search)
CREATE INDEX idx_eob_patient ON fhir_explanationofbenefit (
    (resource #>> '{patient,reference}')
);

-- EOB.created (date search, very common for pagination)
CREATE INDEX idx_eob_created ON fhir_explanationofbenefit (
    (resource ->> 'created')
);

-- Claim.status (token search)
CREATE INDEX idx_claim_status ON fhir_claim (
    (resource ->> 'status')
);

-- Composite index for tenant + status queries (most common pattern)
CREATE INDEX idx_patient_tenant_active ON fhir_patient (
    tenant_id,
    ((resource ->> 'active')::boolean)
) WHERE NOT is_deleted;

-- Partial index for active claims only (huge performance win)
CREATE INDEX idx_claim_active ON fhir_claim (
    tenant_id,
    (resource ->> 'created')
) WHERE (resource ->> 'status') = 'active' AND NOT is_deleted;
```

**Key insight:** Do NOT try to index every FHIR search parameter. Profile your actual query patterns and index those. The 80/20 rule applies heavily -- `patient`, `date`, `status`, `identifier`, and `code` handle the vast majority of searches.

### Querying FHIR JSONB

```sql
-- Find patient by MRN
SELECT resource FROM fhir_patient
WHERE resource @> '{"identifier": [{"system": "http://hospital.org/mrn", "value": "12345"}]}'
  AND tenant_id = $1;

-- Find all EOBs for a patient in a date range
SELECT resource FROM fhir_explanationofbenefit
WHERE resource #>> '{patient,reference}' = 'Patient/123'
  AND (resource ->> 'created')::date BETWEEN '2025-01-01' AND '2025-12-31'
  AND tenant_id = $1
ORDER BY (resource ->> 'created') DESC;

-- Find all active conditions with a specific ICD-10 code
SELECT resource FROM fhir_condition
WHERE resource @> '{"clinicalStatus": {"coding": [{"code": "active"}]}}'
  AND resource -> 'code' -> 'coding' @> '[{"system": "http://hl7.org/fhir/sid/icd-10-cm", "code": "E11.9"}]'
  AND tenant_id = $1;

-- Aggregate: count claims by type
SELECT resource ->> 'type' as claim_type, COUNT(*)
FROM fhir_claim
WHERE tenant_id = $1
  AND NOT is_deleted
GROUP BY resource ->> 'type';
```

---

## 6. FHIR Integration Patterns

### 6.1 SMART on FHIR

SMART (Substitutable Medical Applications, Reusable Technologies) on FHIR is the authorization framework for FHIR APIs. It extends OAuth 2.0 with healthcare-specific scopes and launch contexts.

**Authorization flow:**
1. App registered with FHIR server (client_id, redirect_uri, scopes)
2. User launches app from EHR or standalone
3. App discovers authorization endpoints from `.well-known/smart-configuration`
4. App redirects user to authorization server
5. User authenticates and consents
6. App receives authorization code
7. App exchanges code for access token
8. App uses access token to call FHIR API

**Scopes follow this pattern:**
```
patient/Patient.read          -- Read the current patient
patient/Observation.read      -- Read observations for current patient
user/Patient.read             -- Read any patient the user has access to
system/Patient.read           -- Backend service, read all patients
launch/patient                -- Receive patient context on launch
openid fhirUser               -- Get user identity
```

**SMART App Launch sequence:**
```
GET /fhir/.well-known/smart-configuration
{
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "capabilities": ["launch-ehr", "launch-standalone", "client-public", "client-confidential-symmetric"],
  "scopes_supported": ["patient/*.read", "user/*.read", "openid", "fhirUser"]
}
```

### 6.2 CDS Hooks

Clinical Decision Support Hooks enable real-time decision support integrated into EHR workflows. When a clinician takes an action (opening a chart, ordering a medication), the EHR fires a hook to external CDS services.

**Standard hooks:**
- `patient-view` -- clinician opens a patient chart
- `order-select` -- clinician selects an order
- `order-sign` -- clinician signs an order
- `appointment-book` -- scheduling an appointment
- `encounter-start` / `encounter-discharge`

**Flow:**
```
EHR                          CDS Service (MedOS)
 |                                |
 |-- POST /cds-services/         |  (discovery)
 |<-- list of available hooks     |
 |                                |
 |-- POST /cds-services/pa-check |  (hook fired)
 |   {                           |
 |     "hook": "order-sign",     |
 |     "context": {              |
 |       "patientId": "123",     |
 |       "draftOrders": {...}    |
 |     },                        |
 |     "prefetch": {             |
 |       "patient": {...}        |
 |     }                         |
 |   }                           |
 |<-- {                          |
 |      "cards": [{              |
 |        "summary": "Prior auth required for MRI",
 |        "indicator": "warning",|
 |        "source": {"label": "MedOS PA Engine"},
 |        "suggestions": [...]   |
 |      }]                       |
 |    }                          |
```

This is directly relevant to [[Prior-Authorization-Deep-Dive]] -- MedOS can surface PA requirements at order time.

### 6.3 Bulk Data Access

Covered in section 3.4 above. Key integration patterns:
- **Payer-to-payer exchange:** Bulk export from old payer, bulk import to new payer
- **Analytics pipeline:** Nightly bulk export to data warehouse
- **Quality measure calculation:** Export population data for HEDIS/STARS measures

Uses SMART Backend Services (client_credentials grant with JWT assertion) for authentication -- no user interaction needed.

### 6.4 Subscriptions (R4)

Covered in section 3.5. Key integration patterns:
- **ADT notifications:** Subscribe to Encounter changes for admission/discharge/transfer alerts
- **Lab result delivery:** Subscribe to Observation creates for specific LOINC codes
- **Claim status updates:** Subscribe to Claim/EOB status changes for real-time adjudication tracking

---

## 7. FHIR-MCP Server

The Model Context Protocol (MCP) provides a standardized way for AI/LLM applications to access external data and tools. Several open-source FHIR MCP servers now exist that bridge FHIR APIs with AI agents:

**Key implementations:**

| Project | Maintainer | Description |
|---------|-----------|-------------|
| `fhir-mcp-server` | Momentum AI | Python-based, works with Claude, Cursor, any MCP client. Supports full FHIR CRUD + search. |
| `mcp-fhir` | Flexpa | TypeScript-based MCP server connecting to any FHIR endpoint. |
| `healthlake-mcp-server` | AWS Labs | MCP server for AWS HealthLake with 11 FHIR tools + auto datastore discovery. |
| `fhir-mcp-server` | WSO2 | Exposes any FHIR server as an MCP server. |

**How it works for MedOS:**

```
User (natural language)
    |
    v
AI Agent (Claude / MedOS AI)
    |
    v
MCP Client (in the AI agent)
    |
    v
FHIR MCP Server (translates MCP tool calls to FHIR API calls)
    |
    v
FHIR Server (MedOS backend / external EHR)
```

**Example MCP tool calls the AI agent can make:**
- `search_patients(name="Smith", birthdate="1990-01-15")` -- searches FHIR Patient endpoint
- `read_resource(resource_type="ExplanationOfBenefit", id="eob-123")` -- reads a specific EOB
- `create_resource(resource_type="Claim", resource={...})` -- submits a claim
- `search_observations(patient="Patient/123", code="http://loinc.org|8867-4")` -- finds heart rate observations

**Why this matters:** MedOS can expose its FHIR API through an MCP server, enabling AI-powered clinical decision support, natural-language querying of claims data, and automated workflow agents -- all while maintaining FHIR compliance and [[HIPAA-Deep-Dive]] audit trails.

**Security considerations:** The MCP server must enforce the same SMART on FHIR scopes and RBAC as the underlying FHIR API. The AI agent should never have broader access than the authenticated user. Every MCP tool call must generate an AuditEvent.

---

## 8. Key Implementation Guides

### 8.1 US Core (covered in section 4)

The foundation. Everything else builds on top of US Core profiles.

### 8.2 Da Vinci Implementation Guides

Da Vinci is a private-sector initiative creating payer-focused FHIR IGs. Critical for MedOS:

**PDex (Payer Data Exchange):**
- Enables payer-to-payer clinical data exchange (when a member switches plans)
- Supports CMS-0057 rule compliance
- Uses US Core clinical profiles (Condition, Procedure, Observation, etc.)
- Current version: 2.1.1, supporting US Core 3.1.1, 6.1.0, and 7.0.0
- Defines CDS Hooks integration for provider-payer data sharing

**PAS (Prior Authorization Support):**
- Maps FHIR Claim (`use: "preauthorization"`) to X12 278 transactions
- Enables electronic prior auth submission and response
- Directly relevant to [[Prior-Authorization-Deep-Dive]]
- Reduces phone/fax burden (the industry still does 60%+ of PAs manually)

**CDex (Clinical Data Exchange):**
- How to get clinical data OUT of EHRs and into payer/other provider hands
- Supports three exchange methods: direct query, task-based, and attachments
- Used for: payer-to-payer APIs, prior auth supporting docs, risk adjustment data collection
- The "payload" specification -- PDex says WHAT to exchange, CDex says HOW

**HRex (Health Record Exchange):**
- Shared infrastructure IG used by PDex, CDex, PAS, and others
- Defines common security, consent, and endpoint discovery patterns
- Member matching operation (`$member-match`) for cross-payer patient linking

### 8.3 CARIN Blue Button

**CARIN Consumer Directed Payer Data Exchange (CARIN IG for Blue Button):**
- Defines how payers expose claims/EOB data to consumers via FHIR APIs
- Profiles ExplanationOfBenefit into specific types:
  - `C4BB-ExplanationOfBenefit-Inpatient-Institutional`
  - `C4BB-ExplanationOfBenefit-Outpatient-Institutional`
  - `C4BB-ExplanationOfBenefit-Professional-NonClinician`
  - `C4BB-ExplanationOfBenefit-Pharmacy`
  - `C4BB-ExplanationOfBenefit-Oral`
- Required for CMS Patient Access API compliance
- This is the primary IG for MedOS Module A's external-facing API

**Relationship between IGs:**
```
                    US Core (clinical profiles)
                         |
                    HRex (shared infrastructure)
                    /    |    \
                PDex   CDex   PAS
                  |      |
            CARIN Blue Button (consumer-facing claims)
```

---

## 9. Practical Examples

### 9.1 Patient Resource

```json
{
  "resourceType": "Patient",
  "id": "member-a1b2c3",
  "meta": {
    "versionId": "3",
    "lastUpdated": "2026-02-15T10:30:00.000Z",
    "profile": [
      "http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient"
    ]
  },
  "extension": [
    {
      "url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-race",
      "extension": [
        {
          "url": "ombCategory",
          "valueCoding": {
            "system": "urn:oid:2.16.840.1.113883.6.238",
            "code": "2106-3",
            "display": "White"
          }
        },
        {
          "url": "text",
          "valueString": "White"
        }
      ]
    },
    {
      "url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-ethnicity",
      "extension": [
        {
          "url": "ombCategory",
          "valueCoding": {
            "system": "urn:oid:2.16.840.1.113883.6.238",
            "code": "2186-5",
            "display": "Not Hispanic or Latino"
          }
        },
        {
          "url": "text",
          "valueString": "Not Hispanic or Latino"
        }
      ]
    },
    {
      "url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-birthsex",
      "valueCode": "M"
    }
  ],
  "identifier": [
    {
      "use": "usual",
      "type": {
        "coding": [
          {
            "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
            "code": "MB",
            "display": "Member Number"
          }
        ]
      },
      "system": "https://medos.example.com/member-id",
      "value": "MED-2026-001234"
    },
    {
      "use": "official",
      "type": {
        "coding": [
          {
            "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
            "code": "MR",
            "display": "Medical Record Number"
          }
        ]
      },
      "system": "https://hospital-a.example.com/mrn",
      "value": "H-987654"
    }
  ],
  "active": true,
  "name": [
    {
      "use": "official",
      "family": "Garcia",
      "given": ["Maria", "Elena"]
    }
  ],
  "telecom": [
    {
      "system": "phone",
      "value": "555-867-5309",
      "use": "mobile"
    },
    {
      "system": "email",
      "value": "maria.garcia@example.com",
      "use": "home"
    }
  ],
  "gender": "female",
  "birthDate": "1985-03-22",
  "address": [
    {
      "use": "home",
      "type": "physical",
      "line": ["742 Evergreen Terrace"],
      "city": "Austin",
      "state": "TX",
      "postalCode": "78701",
      "country": "US"
    }
  ],
  "communication": [
    {
      "language": {
        "coding": [
          {
            "system": "urn:ietf:bcp:47",
            "code": "es",
            "display": "Spanish"
          }
        ]
      },
      "preferred": true
    },
    {
      "language": {
        "coding": [
          {
            "system": "urn:ietf:bcp:47",
            "code": "en",
            "display": "English"
          }
        ]
      }
    }
  ],
  "generalPractitioner": [
    {
      "reference": "Practitioner/dr-smith-456",
      "display": "Dr. Robert Smith"
    }
  ]
}
```

### 9.2 Claim Resource (Professional)

```json
{
  "resourceType": "Claim",
  "id": "claim-prof-20260215",
  "meta": {
    "profile": [
      "http://hl7.org/fhir/us/carin-bb/StructureDefinition/C4BB-Claim-Professional"
    ]
  },
  "status": "active",
  "type": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/claim-type",
        "code": "professional",
        "display": "Professional"
      }
    ]
  },
  "use": "claim",
  "patient": {
    "reference": "Patient/member-a1b2c3"
  },
  "billablePeriod": {
    "start": "2026-02-15",
    "end": "2026-02-15"
  },
  "created": "2026-02-16",
  "provider": {
    "reference": "Organization/provider-org-789",
    "display": "Austin Medical Associates"
  },
  "priority": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/processpriority",
        "code": "normal"
      }
    ]
  },
  "insurance": [
    {
      "sequence": 1,
      "focal": true,
      "coverage": {
        "reference": "Coverage/cov-med-2026-001234"
      }
    }
  ],
  "diagnosis": [
    {
      "sequence": 1,
      "diagnosisCodeableConcept": {
        "coding": [
          {
            "system": "http://hl7.org/fhir/sid/icd-10-cm",
            "code": "E11.9",
            "display": "Type 2 diabetes mellitus without complications"
          }
        ]
      },
      "type": [
        {
          "coding": [
            {
              "system": "http://terminology.hl7.org/CodeSystem/ex-diagnosistype",
              "code": "principal"
            }
          ]
        }
      ]
    },
    {
      "sequence": 2,
      "diagnosisCodeableConcept": {
        "coding": [
          {
            "system": "http://hl7.org/fhir/sid/icd-10-cm",
            "code": "I10",
            "display": "Essential (primary) hypertension"
          }
        ]
      }
    }
  ],
  "item": [
    {
      "sequence": 1,
      "productOrService": {
        "coding": [
          {
            "system": "http://www.ama-assn.org/go/cpt",
            "code": "99214",
            "display": "Office visit, established patient, moderate complexity"
          }
        ]
      },
      "servicedDate": "2026-02-15",
      "locationCodeableConcept": {
        "coding": [
          {
            "system": "https://www.cms.gov/Medicare/Coding/place-of-service-codes",
            "code": "11",
            "display": "Office"
          }
        ]
      },
      "unitPrice": {
        "value": 175.00,
        "currency": "USD"
      },
      "net": {
        "value": 175.00,
        "currency": "USD"
      },
      "diagnosisSequence": [1, 2]
    },
    {
      "sequence": 2,
      "productOrService": {
        "coding": [
          {
            "system": "http://www.ama-assn.org/go/cpt",
            "code": "83036",
            "display": "Hemoglobin A1c"
          }
        ]
      },
      "servicedDate": "2026-02-15",
      "unitPrice": {
        "value": 45.00,
        "currency": "USD"
      },
      "net": {
        "value": 45.00,
        "currency": "USD"
      },
      "diagnosisSequence": [1]
    }
  ],
  "total": {
    "value": 220.00,
    "currency": "USD"
  }
}
```

### 9.3 ExplanationOfBenefit Resource

```json
{
  "resourceType": "ExplanationOfBenefit",
  "id": "eob-prof-20260215-adj",
  "meta": {
    "lastUpdated": "2026-02-20T14:30:00.000Z",
    "profile": [
      "http://hl7.org/fhir/us/carin-bb/StructureDefinition/C4BB-ExplanationOfBenefit-Professional-NonClinician"
    ]
  },
  "status": "active",
  "type": {
    "coding": [
      {
        "system": "http://terminology.hl7.org/CodeSystem/claim-type",
        "code": "professional",
        "display": "Professional"
      }
    ]
  },
  "use": "claim",
  "patient": {
    "reference": "Patient/member-a1b2c3"
  },
  "billablePeriod": {
    "start": "2026-02-15",
    "end": "2026-02-15"
  },
  "created": "2026-02-20",
  "insurer": {
    "reference": "Organization/medos-health-plan",
    "display": "MedOS Health Plan"
  },
  "provider": {
    "reference": "Organization/provider-org-789",
    "display": "Austin Medical Associates"
  },
  "outcome": "complete",
  "insurance": [
    {
      "focal": true,
      "coverage": {
        "reference": "Coverage/cov-med-2026-001234"
      }
    }
  ],
  "diagnosis": [
    {
      "sequence": 1,
      "diagnosisCodeableConcept": {
        "coding": [
          {
            "system": "http://hl7.org/fhir/sid/icd-10-cm",
            "code": "E11.9",
            "display": "Type 2 diabetes mellitus without complications"
          }
        ]
      },
      "type": [
        {
          "coding": [
            {
              "system": "http://terminology.hl7.org/CodeSystem/ex-diagnosistype",
              "code": "principal"
            }
          ]
        }
      ]
    }
  ],
  "item": [
    {
      "sequence": 1,
      "productOrService": {
        "coding": [
          {
            "system": "http://www.ama-assn.org/go/cpt",
            "code": "99214",
            "display": "Office visit, established patient, moderate complexity"
          }
        ]
      },
      "servicedDate": "2026-02-15",
      "adjudication": [
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "submitted",
                "display": "Submitted Amount"
              }
            ]
          },
          "amount": {
            "value": 175.00,
            "currency": "USD"
          }
        },
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "eligible",
                "display": "Eligible Amount"
              }
            ]
          },
          "amount": {
            "value": 150.00,
            "currency": "USD"
          }
        },
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "deductible",
                "display": "Deductible"
              }
            ]
          },
          "amount": {
            "value": 0.00,
            "currency": "USD"
          }
        },
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "copay",
                "display": "Copay"
              }
            ]
          },
          "amount": {
            "value": 30.00,
            "currency": "USD"
          }
        },
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "benefit",
                "display": "Benefit Amount"
              }
            ]
          },
          "amount": {
            "value": 120.00,
            "currency": "USD"
          }
        }
      ]
    },
    {
      "sequence": 2,
      "productOrService": {
        "coding": [
          {
            "system": "http://www.ama-assn.org/go/cpt",
            "code": "83036",
            "display": "Hemoglobin A1c"
          }
        ]
      },
      "servicedDate": "2026-02-15",
      "adjudication": [
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "submitted"
              }
            ]
          },
          "amount": {
            "value": 45.00,
            "currency": "USD"
          }
        },
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "eligible"
              }
            ]
          },
          "amount": {
            "value": 38.00,
            "currency": "USD"
          }
        },
        {
          "category": {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/adjudication",
                "code": "benefit"
              }
            ]
          },
          "amount": {
            "value": 38.00,
            "currency": "USD"
          }
        }
      ]
    }
  ],
  "total": [
    {
      "category": {
        "coding": [
          {
            "system": "http://terminology.hl7.org/CodeSystem/adjudication",
            "code": "submitted"
          }
        ]
      },
      "amount": {
        "value": 220.00,
        "currency": "USD"
      }
    },
    {
      "category": {
        "coding": [
          {
            "system": "http://terminology.hl7.org/CodeSystem/adjudication",
            "code": "benefit"
          }
        ]
      },
      "amount": {
        "value": 158.00,
        "currency": "USD"
      }
    }
  ],
  "payment": {
    "type": {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/ex-paymenttype",
          "code": "complete"
        }
      ]
    },
    "date": "2026-02-25",
    "amount": {
      "value": 158.00,
      "currency": "USD"
    }
  },
  "processNote": [
    {
      "number": 1,
      "type": "display",
      "text": "Allowed amount adjusted per contracted rate schedule. Member responsibility: $30 copay + $32 coinsurance on balance."
    }
  ]
}
```

---

## 10. Gotchas and Lessons Learned

### 10.1 Data Quality Issues

**Problem:** FHIR assumes source systems have clean, structured data. Reality: EHRs store tons of free text, proprietary fields, and inconsistent coding.
**Mitigation:** Build robust validation on ingest. Use FHIR validators (e.g., the HL7 Java validator or Firely .NET SDK) before persisting. Accept resources with warnings but flag them for review. Never silently drop data.

**Problem:** Terminology mismatches. A provider sends SNOMED CT codes, but your system expects ICD-10-CM for the same concepts.
**Mitigation:** Maintain concept maps (FHIR ConceptMap resource). Use FHIR terminology services (`$translate`, `$lookup`, `$validate-code`). Never assume 1:1 mappings exist between code systems.

### 10.2 Search Implementation Complexity

**Problem:** The FHIR search spec is enormous. Supporting all search parameters, modifiers, prefixes, chained searches, includes, and reverse includes is a multi-year project.
**Mitigation:** Start with the search parameters required by US Core and CARIN Blue Button. Use the CapabilityStatement to honestly declare what you support. Reject unsupported parameters with a 400 error and an OperationOutcome explaining what is not supported.

**Common search parameters to prioritize:**
- `_id`, `_lastUpdated`, `_count`, `_offset`
- Patient: `identifier`, `name`, `birthdate`, `gender`
- EOB: `patient`, `created`, `type`, `status`
- Claim: `patient`, `created`, `status`, `use`
- Condition: `patient`, `clinical-status`, `code`
- Observation: `patient`, `code`, `date`, `category`

### 10.3 Reference Resolution

**Problem:** FHIR references are just strings like `"Patient/123"`. When a resource is received from an external system, those references point to THEIR server, not yours.
**Mitigation:** On import, rewrite references to point to your internal IDs. Maintain a mapping table: `(source_system, source_id) -> internal_fhir_id`. Use Conditional References (`identifier`-based) when possible to avoid ID collisions.

### 10.4 Versioning and Concurrency

**Problem:** Two systems update the same resource simultaneously.
**Mitigation:** Use `ETag` / `If-Match` headers for optimistic locking. Always increment `meta.versionId` on update. Store all versions in the history table (as shown in section 5).

### 10.5 Extension Handling

**Problem:** Extensions are the wild west of FHIR. Every implementation adds custom extensions, and your system needs to store them without understanding them.
**Mitigation:** This is where FHIR-native JSONB storage shines. Since you store the entire resource, unknown extensions are preserved automatically. Do NOT try to map every extension to a relational column. Only extract extensions you actively use into indexed fields.

### 10.6 Must-Support Misunderstandings

**Problem:** Teams interpret "must-support" as "required." They fail validation on missing must-support elements.
**Mitigation:** Must-support means "must be able to handle," NOT "must always be present." A Patient without a phone number is valid; your system just needs to handle that case gracefully.

### 10.7 Date/Time Handling

**Problem:** FHIR dates can be partial: `2026`, `2026-02`, `2026-02-15`, `2026-02-15T10:30:00Z`. Your database queries need to handle all precisions.
**Mitigation:** Store dates as-is in JSONB (preserve the original precision). For indexed date searches, parse into PostgreSQL `tstzrange` intervals: `2026-02` becomes `[2026-02-01, 2026-03-01)`. This correctly handles range comparisons across precisions.

### 10.8 Bundle Transaction Ordering

**Problem:** A transaction Bundle with interdependent resources (e.g., Patient + Condition that references the Patient) must be processed in dependency order.
**Mitigation:** Implement a topological sort of Bundle entries based on their references before processing. Process entries with no dependencies first, then entries whose dependencies have been resolved.

### 10.9 Pagination Consistency

**Problem:** A patient's data changes between paginated search requests, causing duplicates or missing resources.
**Mitigation:** Use snapshot-based pagination where possible (generate all matching IDs at query time, then serve pages from that snapshot). If that is too expensive, use `_lastUpdated` cursors instead of offset-based pagination.

### 10.10 Regulatory Compliance Pitfalls

**Problem:** CMS rules have specific timelines and version requirements. Missing a deadline means audit findings.
**Key dates (as of early 2026):**
- ONC HTI-1: US Core 6.1.0 required as of January 1, 2026
- CMS-0057: Payer-to-payer exchange, prior auth APIs, provider directory APIs with specific compliance timelines
- Patient Access API: Must serve EOBs conforming to CARIN Blue Button profiles

**Mitigation:** Track CMS rulemaking closely. Build your FHIR API with profile-switching capability so you can adopt new IG versions without rewriting the server. Test with the Inferno test suite (mandated by ONC for certification).

### 10.11 Performance at Scale

**Problem:** A payer with 5 million members, each with 10+ years of claims history, generates hundreds of millions of EOBs. Querying `GET /ExplanationOfBenefit?patient=Patient/X` must return in under 2 seconds.
**Mitigation:**
- Partition tables by tenant_id and date range
- Use read replicas for search-heavy workloads
- Cache hot Patient + Coverage lookups in Redis
- Implement `_count` limits and require pagination
- Pre-compute patient-level summaries for `$everything` operations
- Consider TimescaleDB hypertables for time-series observation data

---

## Quick Reference Card

| What | Where to Look |
|------|--------------|
| FHIR R4 spec | https://hl7.org/fhir/R4/ |
| US Core IG | https://hl7.org/fhir/us/core/ |
| CARIN Blue Button | https://hl7.org/fhir/us/carin-bb/ |
| Da Vinci PDex | https://hl7.org/fhir/us/davinci-pdex/ |
| Da Vinci PAS | https://hl7.org/fhir/us/davinci-pas/ |
| Da Vinci CDex | https://hl7.org/fhir/us/davinci-cdex/ |
| CDS Hooks spec | https://cds-hooks.hl7.org/ |
| SMART on FHIR | https://docs.smarthealthit.org/ |
| Bulk Data Access | https://hl7.org/fhir/uv/bulkdata/ |
| Inferno test suite | https://inferno.healthit.gov/ |
| FHIRbase (PG storage) | https://github.com/HealthSamurai/fhirbase |
| FHIR MCP Server | https://github.com/themomentum-team/fhir-mcp-server |
| X12-FHIR mapping | [[X12-EDI-Deep-Dive]] |
| HIPAA compliance | [[HIPAA-Deep-Dive]] |
| Prior auth workflows | [[Prior-Authorization-Deep-Dive]] |
| System architecture | [[MOC-Architecture]] |
