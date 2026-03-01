---
type: adr
date: "2026-02-27"
status: accepted
tags:
  - adr
  - architecture
  - fhir
  - database
  - module-a
deciders: ["founding-team"]
---

# ADR-001: FHIR-Native Data Model with PostgreSQL JSONB

## Status

**Accepted** -- 2026-02-27

## Context

Healthcare OS (MedOS) is being designed from the ground up as a healthcare platform that must speak FHIR R4 natively. The Fast Healthcare Interoperability Resources (FHIR) standard defines over 150 resource types (Patient, Encounter, Claim, Observation, etc.) with deeply nested structures, polymorphic fields, and extension mechanisms. See [[FHIR-R4-Deep-Dive]] for a comprehensive analysis of the standard.

Traditional healthcare systems store data in rigid relational schemas and then translate to/from FHIR at the API boundary. This translation layer is the single largest source of bugs, data loss, and maintenance burden in healthcare IT. Every time the FHIR spec evolves or a new extension is needed, the relational schema must be migrated and the translation layer must be updated.

Our requirements are:

1. **FHIR-first API surface** -- All external APIs must serve valid FHIR R4 resources without transformation.
2. **Schema flexibility** -- FHIR extensions and profiles must be storable without DDL changes.
3. **AI readiness** -- Clinical data must be queryable for AI agent pipelines (see [[ADR-003-ai-agent-framework]]).
4. **Performance** -- Sub-100ms reads for individual resources, sub-500ms for search bundles.
5. **Regulatory compliance** -- Full audit trail and data lineage as required by [[HIPAA-Deep-Dive]].

## Decision

**We will store all clinical and administrative data natively as FHIR R4 JSON documents in PostgreSQL 17 JSONB columns, complemented by GIN indexes, extracted search columns, and pgvector for AI embeddings.**

### Core Schema Design

```sql
-- Core FHIR resource storage table (per-tenant schema, see ADR-002)
CREATE TABLE fhir_resources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   TEXT NOT NULL,
    resource_id     TEXT NOT NULL,
    version_id      INTEGER NOT NULL DEFAULT 1,
    resource        JSONB NOT NULL,
    txid            BIGINT NOT NULL DEFAULT txid_current(),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'active',  -- active, deleted
    tenant_id       UUID NOT NULL,

    -- Extracted search fields for high-frequency queries
    patient_id      TEXT GENERATED ALWAYS AS (
                        resource->>'subject'  -- simplified; actual extraction is more complex
                    ) STORED,
    encounter_id    TEXT GENERATED ALWAYS AS (
                        resource->'encounter'->>'reference'
                    ) STORED,
    effective_date  TIMESTAMPTZ GENERATED ALWAYS AS (
                        (resource->>'effectiveDateTime')::timestamptz
                    ) STORED,

    UNIQUE (resource_type, resource_id, version_id)
);

-- GIN index for arbitrary FHIR search parameters
CREATE INDEX idx_fhir_resources_gin ON fhir_resources USING GIN (resource jsonb_path_ops);

-- B-tree indexes for extracted columns
CREATE INDEX idx_fhir_resources_type ON fhir_resources (resource_type);
CREATE INDEX idx_fhir_resources_patient ON fhir_resources (patient_id) WHERE patient_id IS NOT NULL;
CREATE INDEX idx_fhir_resources_encounter ON fhir_resources (encounter_id) WHERE encounter_id IS NOT NULL;
CREATE INDEX idx_fhir_resources_effective ON fhir_resources (effective_date) WHERE effective_date IS NOT NULL;
CREATE INDEX idx_fhir_resources_ts ON fhir_resources (ts);

-- AI embedding storage for clinical resources
CREATE TABLE fhir_resource_embeddings (
    resource_id     UUID PRIMARY KEY REFERENCES fhir_resources(id),
    embedding       vector(1536) NOT NULL,  -- pgvector for Claude/OpenAI embeddings
    model_version   TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_hnsw ON fhir_resource_embeddings
    USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

### FHIR History Table

FHIR requires version history for resources. We maintain this through an immutable append pattern:

```sql
-- History is maintained via version_id increments
-- Current resource is always MAX(version_id) for a given (resource_type, resource_id)
CREATE VIEW fhir_resources_current AS
SELECT DISTINCT ON (resource_type, resource_id) *
FROM fhir_resources
WHERE status = 'active'
ORDER BY resource_type, resource_id, version_id DESC;
```

### FHIR Search Parameter Index Table

```sql
-- Materialized search parameters for FHIR search API compliance
CREATE TABLE fhir_search_params (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES fhir_resources(id),
    resource_type   TEXT NOT NULL,
    param_name      TEXT NOT NULL,
    param_type      TEXT NOT NULL,  -- string, token, reference, date, quantity, uri
    value_string    TEXT,
    value_date      TIMESTAMPTZ,
    value_quantity  NUMERIC,
    value_code      TEXT,
    value_system    TEXT,
    value_reference TEXT
);

CREATE INDEX idx_search_params_lookup
    ON fhir_search_params (resource_type, param_name, value_string);
CREATE INDEX idx_search_params_token
    ON fhir_search_params (resource_type, param_name, value_system, value_code);
CREATE INDEX idx_search_params_date
    ON fhir_search_params (resource_type, param_name, value_date);
```

### Technology Stack

| Component      | Choice              | Rationale                                        |
| -------------- | ------------------- | ------------------------------------------------ |
| Database       | PostgreSQL 17       | JSONB maturity, GIN indexes, partitioning, RLS   |
| Vector Search  | pgvector 0.8+       | In-database similarity search for AI pipelines   |
| ORM/Query      | SQLAlchemy 2.0      | Async support, JSONB operators (see [[ADR-004-fastapi-backend-architecture]]) |
| Validation     | Pydantic v2 + FHIR  | FHIR resource models generated from StructureDefinitions |
| Migration      | Alembic             | Schema versioning with tenant awareness          |

## Consequences

### Positive

- **Zero translation overhead** -- FHIR resources stored as-is means the API layer simply reads and writes JSONB. No ORM mapping, no field-by-field translation. A GET /Patient/123 is a single indexed lookup returning the stored JSON directly.
- **Schema evolution without migrations** -- New FHIR extensions, profiles, and custom fields are stored without any DDL changes. The JSONB column accepts any valid FHIR structure.
- **Direct AI pipeline integration** -- Clinical resources can be fed directly to AI agents (see [[ADR-003-ai-agent-framework]]) without transformation. The pgvector embeddings enable semantic search across the clinical record.
- **Full FHIR history support** -- The append-only version pattern natively supports FHIR's /_history endpoint.
- **Regulatory alignment** -- Immutable resource versions provide built-in audit trail capability required by [[HIPAA-Deep-Dive]].

### Negative

- **Query complexity** -- JSONB queries require familiarity with PostgreSQL JSON operators (`->`, `->>`, `@>`, `jsonb_path_query`). Developers must learn these patterns. Mitigation: we provide a query builder layer in the service code.
- **Indexing strategy is critical** -- Without proper GIN and extracted-column indexes, JSONB queries can be slow. We must maintain a disciplined indexing strategy and monitor query plans. Mitigation: the `fhir_search_params` table provides pre-extracted search indexes.
- **Storage overhead** -- JSONB is less space-efficient than normalized relational storage. A Patient resource with 20 fields stores field names redundantly. Mitigation: PostgreSQL TOAST compression reduces this significantly; storage cost is minimal compared to development velocity gained.
- **Reporting complexity** -- Ad-hoc reporting requires JSONB extraction. Mitigation: we will build materialized views for common reporting patterns and consider a read-replica with denormalized reporting tables.

### Risks

- **Performance at scale** -- JSONB GIN indexes can become large with millions of resources. Mitigation: table partitioning by `resource_type` and `ts` (time-based), combined with connection pooling via PgBouncer.
- **FHIR validation burden** -- The database accepts any JSONB; validation must be enforced at the application layer. Mitigation: Pydantic FHIR models validate on write; a background job validates periodically.

## Alternatives Considered

### 1. Pure Relational Model with FHIR Facade

A traditional approach with normalized tables (patients, encounters, observations, etc.) and a translation layer.

- **Rejected because**: The translation layer is the highest-maintenance component in healthcare systems. Every FHIR profile change requires schema migration + code change. Our team would spend more time maintaining the facade than building features.

### 2. MongoDB or Document Database

A document database seems natural for FHIR's JSON-based resources.

- **Rejected because**: PostgreSQL offers JSONB with equivalent document capabilities PLUS relational features (joins, transactions, RLS, generated columns), pgvector for AI, and the operational maturity our team already has. MongoDB would add operational complexity without meaningful benefit.

### 3. HAPI FHIR Server (Java)

The open-source reference implementation of FHIR.

- **Rejected because**: HAPI is a Java monolith optimized for FHIR compliance testing, not for building a commercial platform. It lacks multi-tenancy, AI integration, and the extensibility we need. We would spend more time working around HAPI than building on it.

### 4. Hybrid (Relational Core + FHIR Cache)

Store relationally, cache FHIR representations.

- **Rejected because**: Cache invalidation in healthcare is dangerous. Stale clinical data can cause patient harm. The complexity of keeping two representations in sync is not worth the marginal query simplicity.

## References

- [[HEALTHCARE_OS_MASTERPLAN]] -- Overall system vision and module definitions
- [[FHIR-R4-Deep-Dive]] -- FHIR standard analysis and resource catalog
- [[ADR-002-multi-tenancy-schema-per-tenant]] -- Tenant isolation strategy that builds on this schema
- [[ADR-003-ai-agent-framework]] -- AI agent architecture that consumes FHIR data
- [[ADR-004-fastapi-backend-architecture]] -- Backend framework that implements the data access layer
- [[EPIC-003-fhir-data-layer]] -- FHIR data layer implementation tasks
- [[EPIC-007-mcp-sdk-refactoring]] -- FHIR MCP server migration to @hipaa_tool
- [[EPIC-008-demo-polish]] -- Sprint 3 seed data and frontend FHIR integration
- [PostgreSQL JSONB Documentation](https://www.postgresql.org/docs/17/datatype-json.html)
- [pgvector](https://github.com/pgvector/pgvector)
- [HL7 FHIR R4 Specification](https://hl7.org/fhir/R4/)
