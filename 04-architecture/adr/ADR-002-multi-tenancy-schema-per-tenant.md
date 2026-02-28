---
type: adr
date: "2026-02-27"
status: accepted
tags:
  - adr
  - architecture
  - multi-tenancy
  - security
  - module-g
deciders: ["founding-team"]
---

# ADR-002: Schema-Per-Tenant Multi-Tenancy with Per-Tenant KMS Keys

## Status

**Accepted** -- 2026-02-27

## Context

MedOS serves multiple healthcare organizations (clinics, hospitals, practice groups) from a single platform deployment. Healthcare data isolation is not merely a business preference -- it is a regulatory requirement. [[HIPAA-Deep-Dive]] mandates that Protected Health Information (PHI) from one Covered Entity must never be accessible to another, even through application bugs, SQL injection, or misconfigured queries.

Our multi-tenancy strategy must satisfy:

1. **Regulatory isolation** -- PHI from Tenant A must be inaccessible to Tenant B at the database level, not just the application level.
2. **Encryption independence** -- Compromising one tenant's encryption keys must not expose another tenant's data. A breach at one clinic must not cascade.
3. **Operational simplicity** -- Tenant provisioning should be automated and fast (under 30 seconds for a new clinic).
4. **Performance** -- Tenant isolation must not degrade query performance compared to single-tenant operation.
5. **Backup/restore independence** -- Individual tenants must be restorable without affecting others.
6. **Compliance auditability** -- Auditors must be able to verify isolation boundaries exist and are enforced.

The data model defined in [[ADR-001-fhir-native-data-model]] uses FHIR JSONB storage in PostgreSQL. This ADR defines how that storage is partitioned across tenants.

## Decision

**We will use PostgreSQL schema-per-tenant isolation combined with Row Level Security (RLS) as a defense-in-depth layer, and AWS KMS keys per tenant for encryption-at-rest independence.**

### Schema Isolation Architecture

Each tenant gets a dedicated PostgreSQL schema containing identical table structures:

```sql
-- Tenant provisioning: create schema and tables
CREATE OR REPLACE FUNCTION provision_tenant(
    p_tenant_id UUID,
    p_tenant_slug TEXT
) RETURNS VOID AS $$
DECLARE
    schema_name TEXT := 'tenant_' || replace(p_tenant_id::text, '-', '_');
BEGIN
    -- Create isolated schema
    EXECUTE format('CREATE SCHEMA IF NOT EXISTS %I', schema_name);

    -- Create FHIR resource table within tenant schema
    EXECUTE format('
        CREATE TABLE %I.fhir_resources (
            id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            resource_type   TEXT NOT NULL,
            resource_id     TEXT NOT NULL,
            version_id      INTEGER NOT NULL DEFAULT 1,
            resource        JSONB NOT NULL,
            txid            BIGINT NOT NULL DEFAULT txid_current(),
            ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
            status          TEXT NOT NULL DEFAULT ''active'',
            tenant_id       UUID NOT NULL DEFAULT %L,
            UNIQUE (resource_type, resource_id, version_id)
        )', schema_name, p_tenant_id);

    -- GIN index for FHIR search
    EXECUTE format('CREATE INDEX ON %I.fhir_resources USING GIN (resource jsonb_path_ops)', schema_name);

    -- Additional indexes matching ADR-001 schema
    EXECUTE format('CREATE INDEX ON %I.fhir_resources (resource_type)', schema_name);
    EXECUTE format('CREATE INDEX ON %I.fhir_resources (ts)', schema_name);

    -- Enable RLS as defense-in-depth
    EXECUTE format('ALTER TABLE %I.fhir_resources ENABLE ROW LEVEL SECURITY', schema_name);
    EXECUTE format('ALTER TABLE %I.fhir_resources FORCE ROW LEVEL SECURITY', schema_name);

    -- RLS policy: only the tenant''s service role can access
    EXECUTE format('
        CREATE POLICY tenant_isolation ON %I.fhir_resources
        FOR ALL
        USING (tenant_id = current_setting(''app.current_tenant_id'')::uuid)
        WITH CHECK (tenant_id = current_setting(''app.current_tenant_id'')::uuid)
    ', schema_name);

    -- Audit log table
    EXECUTE format('
        CREATE TABLE %I.audit_log (
            id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            event_type      TEXT NOT NULL,
            resource_type   TEXT,
            resource_id     TEXT,
            actor_id        UUID NOT NULL,
            actor_role      TEXT NOT NULL,
            ip_address      INET,
            user_agent      TEXT,
            request_id      UUID,
            details         JSONB,
            ts              TIMESTAMPTZ NOT NULL DEFAULT now()
        )', schema_name);

    EXECUTE format('CREATE INDEX ON %I.audit_log (ts)', schema_name);
    EXECUTE format('CREATE INDEX ON %I.audit_log (actor_id)', schema_name);
    EXECUTE format('CREATE INDEX ON %I.audit_log (resource_type, resource_id)', schema_name);

    -- Register tenant in central registry
    INSERT INTO public.tenants (id, slug, schema_name, status, created_at)
    VALUES (p_tenant_id, p_tenant_slug, schema_name, 'active', now());
END;
$$ LANGUAGE plpgsql;
```

### Central Tenant Registry

```sql
-- Public schema: tenant registry (not PHI, no RLS needed)
CREATE TABLE public.tenants (
    id              UUID PRIMARY KEY,
    slug            TEXT UNIQUE NOT NULL,
    schema_name     TEXT UNIQUE NOT NULL,
    display_name    TEXT,
    kms_key_arn     TEXT NOT NULL,      -- Per-tenant KMS key ARN
    status          TEXT NOT NULL DEFAULT 'active',
    plan_tier       TEXT NOT NULL DEFAULT 'standard',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Connection-Level Tenant Binding

Every database connection is bound to a specific tenant at the middleware level:

```python
# FastAPI middleware for tenant context (see ADR-004)
from fastapi import Request
from sqlalchemy.ext.asyncio import AsyncSession

async def set_tenant_context(session: AsyncSession, tenant_id: str, tenant_schema: str):
    """Bind a database session to a specific tenant."""
    # Set search_path to tenant schema (primary isolation)
    await session.execute(text(f"SET search_path TO {tenant_schema}, public"))
    # Set RLS variable (defense-in-depth)
    await session.execute(text(f"SET app.current_tenant_id TO '{tenant_id}'"))

class TenantMiddleware:
    async def __call__(self, request: Request, call_next):
        tenant_id = extract_tenant_from_request(request)  # From JWT, subdomain, or header
        tenant = await get_tenant(tenant_id)
        request.state.tenant = tenant
        request.state.schema = tenant.schema_name
        response = await call_next(request)
        return response
```

### Per-Tenant KMS Encryption

Each tenant is provisioned with a dedicated AWS KMS key:

```python
import boto3

async def provision_tenant_kms(tenant_id: str, tenant_slug: str) -> str:
    """Create a dedicated KMS key for a tenant."""
    kms = boto3.client('kms')

    response = kms.create_key(
        Description=f'MedOS PHI encryption key for tenant {tenant_slug}',
        KeyUsage='ENCRYPT_DECRYPT',
        KeySpec='SYMMETRIC_DEFAULT',
        Tags=[
            {'TagKey': 'tenant_id', 'TagValue': tenant_id},
            {'TagKey': 'application', 'TagValue': 'medos'},
            {'TagKey': 'data_classification', 'TagValue': 'phi'},
        ],
        Policy=json.dumps({
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "TenantKeyAccess",
                    "Effect": "Allow",
                    "Principal": {"AWS": MEDOS_SERVICE_ROLE_ARN},
                    "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "kms:EncryptionContext:tenant_id": tenant_id
                        }
                    }
                }
            ]
        })
    )

    return response['KeyMetadata']['Arn']
```

### Defense-in-Depth Layers

The isolation strategy is layered, so that any single layer's failure does not expose data:

| Layer | Mechanism | Protects Against |
|-------|-----------|-----------------|
| 1. Application | Tenant middleware extracts tenant from JWT | Accidental cross-tenant API calls |
| 2. Schema | `SET search_path` to tenant schema | SQL queries hitting wrong tables |
| 3. RLS | `current_setting('app.current_tenant_id')` | Queries within the right schema but missing WHERE clause |
| 4. Encryption | Per-tenant KMS key with encryption context | Database dump or storage-level breach |
| 5. Audit | All access logged with tenant context | Post-breach forensics and compliance |

## Consequences

### Positive

- **Regulatory-grade isolation** -- Schema separation provides physical query boundary. Even a SQL injection cannot cross schema boundaries because `search_path` is set at the connection level.
- **Independent encryption** -- Revoking a tenant's KMS key immediately renders their data unreadable without affecting other tenants. This supports contractual data destruction requirements.
- **Independent backup/restore** -- `pg_dump --schema=tenant_xxx` allows per-tenant backup and restore operations without impacting the platform.
- **Audit clarity** -- Compliance auditors can verify isolation by examining schema boundaries, RLS policies, and KMS key policies independently.
- **No noisy neighbor** -- Each tenant's indexes are independent, so one tenant's data volume does not degrade another's query performance.

### Negative

- **Schema proliferation** -- At 1000+ tenants, the PostgreSQL catalog becomes large. Mitigation: connection pooling with PgBouncer, and schema-per-tenant works well up to approximately 10,000 schemas. Beyond that, we would shard across database instances.
- **Migration complexity** -- Schema changes must be applied to every tenant schema. Mitigation: automated migration runner that iterates over all tenant schemas in parallel, with rollback capability.
- **Connection overhead** -- Each tenant context requires `SET search_path`, adding a round-trip. Mitigation: connection pooling with prepared tenant contexts; the overhead is sub-millisecond.

### Risks

- **Migration failures** -- A schema migration that fails partway through leaves tenants on different schema versions. Mitigation: migration runner with per-tenant version tracking, retry logic, and alerting.
- **KMS cost** -- Each KMS key costs $1/month. At 10,000 tenants, this is $10K/month for keys alone. Mitigation: acceptable cost for healthcare-grade isolation; can use KMS aliases and key hierarchy for cost optimization at scale.

## Alternatives Considered

### 1. Shared Schema with `tenant_id` Column

A single set of tables with a `tenant_id` column and RLS policies.

- **Rejected because**: A single RLS misconfiguration or a query that forgets the tenant filter exposes ALL tenants' PHI simultaneously. The blast radius is unacceptable for healthcare. Additionally, shared indexes degrade as tenant count grows.

### 2. Database-Per-Tenant

Each tenant gets a separate PostgreSQL database.

- **Rejected because**: Operational overhead is prohibitive. Connection pooling becomes complex (one pool per database), migrations require connecting to each database individually, and cross-tenant analytics (anonymized, aggregated) becomes nearly impossible. Schema-per-tenant provides equivalent isolation with lower operational cost.

### 3. Separate Infrastructure Per Tenant

Each tenant gets their own compute and database instances.

- **Rejected because**: Cost-prohibitive for small clinics (our primary market). A 3-provider clinic cannot justify dedicated infrastructure. This model only works for large hospital systems and would price out our target market.

## References

- [[HEALTHCARE_OS_MASTERPLAN]] -- Multi-tenancy requirements and market positioning
- [[HIPAA-Deep-Dive]] -- PHI isolation requirements and breach notification rules
- [[ADR-001-fhir-native-data-model]] -- FHIR data model that lives within each tenant schema
- [[ADR-004-fastapi-backend-architecture]] -- Backend framework implementing tenant middleware
- [PostgreSQL Row Level Security](https://www.postgresql.org/docs/17/ddl-rowsecurity.html)
- [PostgreSQL Schemas](https://www.postgresql.org/docs/17/ddl-schemas.html)
- [AWS KMS Encryption Context](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#encrypt_context)
