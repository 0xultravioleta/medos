---
type: adr
date: "2026-02-27"
status: accepted
tags:
  - adr
  - architecture
  - backend
  - fastapi
  - python
deciders: ["founding-team"]
---

# ADR-004: FastAPI as Primary Backend Framework

## Status

**Accepted** -- 2026-02-27

## Context

MedOS requires a backend framework that can serve FHIR-compliant REST APIs, orchestrate AI agent workflows, handle real-time WebSocket connections for clinical interfaces, and integrate with the Python-dominant healthcare and AI/ML ecosystem.

The backend must:

1. **Serve FHIR R4 APIs** -- RESTful endpoints that return valid FHIR Bundle, Patient, Encounter, Claim, and other resource types as defined in [[ADR-001-fhir-native-data-model]].
2. **Support async I/O** -- AI agent calls to Claude can take 2-10 seconds. Database queries, external API calls (payer portals, clearinghouses), and FHIR server interactions must not block the event loop.
3. **Integrate with AI/ML ecosystem** -- LangGraph, Anthropic SDK, embeddings generation, and future ML models are all Python-native (see [[ADR-003-ai-agent-framework]]).
4. **Provide type safety** -- Healthcare data handling requires strict validation. A malformed Patient resource must be rejected at the API boundary, not discovered at the database layer.
5. **Support multi-tenancy** -- Middleware must extract tenant context from every request and bind it to the database session (see [[ADR-002-multi-tenancy-schema-per-tenant]]).
6. **Enable rapid development** -- As a startup, we need to move fast. Framework boilerplate should be minimal while maintaining production quality.

## Decision

**We will use FastAPI (Python 3.12+) as the primary backend framework, with SQLAlchemy 2.0 for async database access, Pydantic v2 for data validation, and a modular project structure aligned with our service modules.**

### Technology Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Framework | FastAPI | 0.115+ | Async-native, OpenAPI auto-gen, dependency injection |
| Language | Python | 3.12+ | AI/ML ecosystem, healthcare libraries, team expertise |
| Validation | Pydantic v2 | 2.9+ | FHIR resource models, 5-50x faster than v1 |
| ORM | SQLAlchemy | 2.0+ | Async sessions, JSONB support, migration ecosystem |
| Migrations | Alembic | 1.14+ | Multi-schema tenant migrations |
| HTTP Server | Uvicorn + Gunicorn | Latest | ASGI server with worker management |
| Task Queue | Celery + Redis | Latest | Background jobs (claim processing, batch AI) |
| Caching | Redis | 7+ | Session cache, rate limiting, FHIR search cache |
| Testing | pytest + httpx | Latest | Async test support, API testing |

### Project Structure

```
medos-backend/
|-- app/
|   |-- __init__.py
|   |-- main.py                     # FastAPI app factory
|   |-- config.py                   # Settings (pydantic-settings)
|   |-- dependencies.py             # Shared dependency injection
|   |
|   |-- core/                       # Cross-cutting concerns
|   |   |-- security.py             # OAuth2, SMART on FHIR, JWT
|   |   |-- middleware.py           # Tenant, logging, CORS, rate limiting
|   |   |-- exceptions.py          # FHIR OperationOutcome error responses
|   |   |-- audit.py               # HIPAA audit trail logging
|   |   |-- events.py              # Event bus (publish/subscribe)
|   |
|   |-- db/                         # Database layer
|   |   |-- session.py             # Async session factory, tenant binding
|   |   |-- base.py                # SQLAlchemy base model
|   |   |-- repositories/          # Data access objects
|   |   |   |-- fhir_repository.py
|   |   |   |-- audit_repository.py
|   |
|   |-- fhir/                       # FHIR layer
|   |   |-- models/                # Pydantic FHIR resource models
|   |   |   |-- patient.py
|   |   |   |-- encounter.py
|   |   |   |-- claim.py
|   |   |   |-- ...
|   |   |-- search.py              # FHIR search parameter parsing
|   |   |-- bundle.py              # FHIR Bundle construction
|   |   |-- validators.py          # FHIR profile validation
|   |
|   |-- modules/                    # Business logic modules
|   |   |-- clinical/              # Module A: Clinical Operations
|   |   |   |-- router.py
|   |   |   |-- service.py
|   |   |   |-- schemas.py
|   |   |-- ai_engine/             # Module B: AI Engine
|   |   |   |-- router.py
|   |   |   |-- agents/
|   |   |   |   |-- coding_agent.py
|   |   |   |   |-- documentation_agent.py
|   |   |   |-- service.py
|   |   |-- rcm/                   # Module C: Revenue Cycle
|   |   |   |-- router.py
|   |   |   |-- service.py
|   |   |   |-- schemas.py
|   |   |-- scheduling/            # Module D: Patient Flow
|   |   |   |-- router.py
|   |   |   |-- service.py
|   |   |-- engagement/            # Module E: Patient Engagement
|   |   |   |-- router.py
|   |   |   |-- service.py
|   |   |-- analytics/             # Module F: Analytics
|   |   |   |-- router.py
|   |   |   |-- service.py
|   |   |-- admin/                 # Module G: Platform Admin
|   |   |   |-- router.py
|   |   |   |-- service.py
|   |   |-- integrations/          # Module H: Integrations
|   |   |   |-- router.py
|   |   |   |-- ehr_connector.py
|   |   |   |-- clearinghouse.py
|   |
|   |-- workers/                    # Background task workers
|   |   |-- claim_processor.py
|   |   |-- batch_coding.py
|   |   |-- eligibility_checker.py
|
|-- migrations/                     # Alembic migrations
|   |-- env.py                     # Multi-tenant migration config
|   |-- versions/
|
|-- tests/
|   |-- conftest.py                # Fixtures: test DB, test tenant, mock LLM
|   |-- test_fhir/
|   |-- test_modules/
|   |-- test_integration/
|
|-- Dockerfile
|-- docker-compose.yml
|-- pyproject.toml
|-- .env.example
```

### Core Application Factory

```python
# app/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from app.core.middleware import TenantMiddleware, AuditMiddleware, RequestIdMiddleware
from app.db.session import init_db, close_db

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifecycle: startup and shutdown."""
    await init_db()
    yield
    await close_db()

def create_app() -> FastAPI:
    app = FastAPI(
        title="MedOS Healthcare Platform",
        version="0.1.0",
        lifespan=lifespan,
        docs_url="/api/docs",       # Swagger UI
        redoc_url="/api/redoc",     # ReDoc
        openapi_url="/api/openapi.json",
    )

    # Middleware (order matters: outermost first)
    app.add_middleware(RequestIdMiddleware)
    app.add_middleware(AuditMiddleware)
    app.add_middleware(TenantMiddleware)

    # Mount module routers
    from app.modules.clinical.router import router as clinical_router
    from app.modules.ai_engine.router import router as ai_router
    from app.modules.rcm.router import router as rcm_router
    from app.modules.scheduling.router import router as scheduling_router
    from app.modules.engagement.router import router as engagement_router
    from app.modules.analytics.router import router as analytics_router
    from app.modules.admin.router import router as admin_router
    from app.modules.integrations.router import router as integrations_router

    app.include_router(clinical_router, prefix="/api/v1/clinical", tags=["Clinical"])
    app.include_router(ai_router, prefix="/api/v1/ai", tags=["AI Engine"])
    app.include_router(rcm_router, prefix="/api/v1/rcm", tags=["Revenue Cycle"])
    app.include_router(scheduling_router, prefix="/api/v1/scheduling", tags=["Scheduling"])
    app.include_router(engagement_router, prefix="/api/v1/engagement", tags=["Engagement"])
    app.include_router(analytics_router, prefix="/api/v1/analytics", tags=["Analytics"])
    app.include_router(admin_router, prefix="/api/v1/admin", tags=["Admin"])
    app.include_router(integrations_router, prefix="/api/v1/integrations", tags=["Integrations"])

    # FHIR-native endpoints (separate prefix)
    from app.fhir.router import router as fhir_router
    app.include_router(fhir_router, prefix="/fhir/r4", tags=["FHIR R4"])

    return app
```

### FHIR Endpoint Example

```python
# app/fhir/router.py
from fastapi import APIRouter, Depends, Query, Request
from app.fhir.models.patient import PatientResource
from app.fhir.bundle import create_search_bundle
from app.db.repositories.fhir_repository import FHIRRepository
from app.core.security import require_scope

router = APIRouter()

@router.get(
    "/Patient/{patient_id}",
    response_model=PatientResource,
    responses={404: {"description": "Patient not found"}},
)
async def get_patient(
    patient_id: str,
    request: Request,
    repo: FHIRRepository = Depends(),
    _auth=Depends(require_scope("patient/*.read")),
):
    """Read a Patient resource by ID (FHIR read interaction)."""
    resource = await repo.read("Patient", patient_id, tenant=request.state.tenant)
    if not resource:
        raise FHIRNotFoundError("Patient", patient_id)
    return resource

@router.get("/Patient", response_model=dict)
async def search_patients(
    request: Request,
    family: str | None = Query(None, alias="family"),
    given: str | None = Query(None, alias="given"),
    birthdate: str | None = Query(None, alias="birthdate"),
    identifier: str | None = Query(None, alias="identifier"),
    _count: int = Query(20, alias="_count", le=100),
    repo: FHIRRepository = Depends(),
    _auth=Depends(require_scope("patient/*.read")),
):
    """Search Patient resources (FHIR search interaction)."""
    params = {k: v for k, v in {
        "family": family, "given": given,
        "birthdate": birthdate, "identifier": identifier,
    }.items() if v is not None}

    results = await repo.search("Patient", params, limit=_count, tenant=request.state.tenant)
    return create_search_bundle("Patient", results, request.url)
```

### Dependency Injection Pattern

```python
# app/dependencies.py
from fastapi import Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import get_session, set_tenant_context

async def get_tenant_session(request: Request) -> AsyncSession:
    """Provide a tenant-scoped database session."""
    async with get_session() as session:
        await set_tenant_context(
            session,
            tenant_id=str(request.state.tenant.id),
            tenant_schema=request.state.tenant.schema_name,
        )
        yield session
```

## Consequences

### Positive

- **AI ecosystem alignment** -- Python is the lingua franca of AI/ML. LangGraph, Anthropic SDK, sentence-transformers, and every major AI library is Python-first. No FFI bridges or language interop needed.
- **Type safety with Pydantic v2** -- FHIR resources are validated at the API boundary with auto-generated OpenAPI schemas. Invalid data is rejected before reaching the service layer. Pydantic v2's Rust-based core provides validation speeds comparable to Go.
- **Async-native** -- FastAPI on ASGI (Uvicorn) handles concurrent AI API calls, database queries, and external integrations without thread pool exhaustion. Critical for a system making multiple Claude API calls per request.
- **Auto-generated API documentation** -- OpenAPI spec is generated from type annotations. External integrators (EHR vendors, clearinghouses) get interactive API docs without separate documentation effort.
- **Dependency injection** -- FastAPI's `Depends()` system cleanly handles tenant context, database sessions, authentication, and authorization without global state or thread-local hacks.
- **Healthcare library ecosystem** -- `fhir.resources` (Pydantic FHIR models), `python-hl7` (HL7v2 parsing), `pydicom` (DICOM imaging) are all Python-native.

### Negative

- **Python performance ceiling** -- CPU-bound operations (PDF generation, large data transformations) are slower than Go or Rust. Mitigation: offload CPU-bound work to Celery workers; use Rust extensions via PyO3 for hot paths if needed; the primary bottleneck is I/O (database, LLM APIs), not CPU.
- **Concurrency model** -- Python's GIL limits true parallel execution within a single process. Mitigation: run multiple Uvicorn workers (one per CPU core) behind Gunicorn; async I/O handles the concurrency that matters (network calls).
- **Deployment size** -- Python containers are larger than Go binaries. Mitigation: multi-stage Docker builds with slim base images; acceptable trade-off for development velocity.

### Risks

- **Framework immaturity perception** -- FastAPI is newer than Django/Flask. Mitigation: FastAPI is production-proven at Netflix, Microsoft, Uber, and multiple healthcare companies. It is built on Starlette (battle-tested ASGI toolkit) and Pydantic (ubiquitous in Python).
- **Team scaling** -- Python's flexibility can lead to inconsistent code patterns. Mitigation: strict project structure (above), mandatory type annotations, ruff for linting, mypy for type checking, enforced via CI.

## Alternatives Considered

### 1. Django + Django REST Framework

The most popular Python web framework with a mature ecosystem.

- **Rejected because**: Django is synchronous by default. Django's async support (added in 3.1+) is still incomplete -- the ORM is largely synchronous, and DRF has limited async support. For a system making concurrent AI API calls, this would require wrapping everything in `sync_to_async`, adding complexity without benefit. Django's "batteries included" philosophy also brings ORM patterns that conflict with our FHIR-native JSONB approach.

### 2. Node.js (Express/Fastify)

JavaScript/TypeScript backend with strong async I/O.

- **Rejected because**: The AI/ML ecosystem is Python-dominant. LangGraph, Anthropic SDK, and healthcare libraries (fhir.resources, python-hl7) have no mature Node.js equivalents. We would need to maintain a separate Python service for AI anyway, doubling our operational surface.

### 3. Go (Gin/Echo)

Compiled language with excellent performance and concurrency.

- **Rejected because**: Go's AI/ML ecosystem is minimal. No LangGraph equivalent, limited Anthropic SDK support, no FHIR resource libraries. The team would spend more time building infrastructure than features. Go is excellent for performance-critical microservices and could be adopted for specific hot-path services in the future.

### 4. Rust (Axum/Actix)

Maximum performance with memory safety.

- **Rejected because**: Rust's development velocity for CRUD-heavy API development is significantly slower than Python. The AI/ML ecosystem is nascent. Rust is appropriate for specific performance-critical components (data pipeline, real-time processing) but not for a rapid-iteration API platform. Could be adopted for specific services later.

## References

- [[HEALTHCARE_OS_MASTERPLAN]] -- Technology decisions and module definitions
- [[ADR-001-fhir-native-data-model]] -- Data layer accessed by the backend
- [[ADR-002-multi-tenancy-schema-per-tenant]] -- Multi-tenancy middleware implemented in FastAPI
- [[ADR-003-ai-agent-framework]] -- AI agents orchestrated from the backend
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Pydantic v2](https://docs.pydantic.dev/latest/)
