# CLAUDE.md - MedOS Healthcare OS

> Instrucciones operativas para Claude al trabajar en el proyecto MedOS.
> Actualizado: 2026-02-28 (Sprint 6 in-progress)

---

## PROYECTO

**MedOS** = Healthcare OS -- sistema operativo AI-native para healthcare en USA.
- **Capital:** $100M family office (Miami, FL)
- **Pitch a:** Dr. Di Reze (Technical Operations)
- **Target:** Mid-size specialty practices (5-30 providers) en Florida
- **Especialidad inicial:** Orthopedics o Dermatology
- **Equipo:** 2 personas + AI tools (Claude Code + Cursor = 5-10x multiplier)
- **Timeline:** 90 dias Foundation -> Pilot (Sprint 0 = 2026-03-01)

**Vault root:** `Z:\medos\` -- este es un Obsidian vault Y repo Git.

---

## FLUJO DE EJECUCION

El trabajo sigue esta jerarquia (de estrategico a operativo):

```
HEALTHCARE_OS_MASTERPLAN.md    -> Vision, estrategia, numeros
  └── PRE-DEV-CHECKLIST.md     -> Day 0: legal, cuentas, tools (ANTES de codear)
      └── PHASE-1-EXECUTION-PLAN.md  -> 117 tasks, 7 sprints, dia-a-dia
          └── EPIC-001 a EPIC-006    -> Detalle por area con acceptance criteria
```

### Task IDs
Formato: `S{sprint}-T{numero}` (ej: `S0-T01`, `S1-T15`, `S4-T06`)
- Sprint 0 tiene 43 tasks (las mas granulares, dia por dia)
- Cada task tiene: Owner (A/B), horas estimadas, dependencias, acceptance criteria

### Tracking de Progreso
Cuando una task se complete, actualizar `Status` en la tabla del execution plan:
- `pending` -> `in-progress` -> `done`
- Tambien actualizar el frontmatter `status` de los EPICs cuando cambien

### Sprints
| Sprint | Semanas | Foco | Status |
|--------|---------|------|--------|
| S0 | W1-2 | AWS, CI/CD, Auth, DB, FastAPI, FHIR Patient CRUD | done |
| S1 | W3-4 | FHIR resources, patient matching, event bus | done |
| S2 | W5-6 | MCP SDK refactoring, 32 tools, 3 agents, HIPAAFastMCP | done |
| S3 | W7-8 | Demo polish: Approvals UI, WebSocket, agent runner, intake workflow | done |
| S4 | W9-10 | Revenue cycle completion: X12 837P, scrubbing, 835 parser, PA, denials, analytics | done |
| S5 | W11-12 | Security hardening, field encryption, tenant onboarding, EHR bridge, monitoring, load testing | done |
| S6 | W13 | Prod deploy, pilot onboarding, go-live | **in-progress** |
| S4F | W9-10 | Theoria Medical pilot: 13 pages, 7 agents, public docs, Bring Your Agent | done |

### Platform Metrics (Sprint 6)
| Metric | Count |
|--------|-------|
| Tests passing | 407+ |
| MCP tools | 36 |
| LangGraph agents | 5 |
| Frontend routes | 53+ |
| Vault docs | 50+ |

---

## TECH STACK

| Capa | Tecnologia |
|------|-----------|
| Backend | FastAPI (Python 3.12+), SQLAlchemy 2.0, Pydantic v2 |
| Frontend | Next.js 15 (App Router, Server Components para PHI) |
| DB principal | PostgreSQL 17 + pgvector (FHIR JSONB nativo) |
| Cache | Redis 7+ |
| Events | EventBridge |
| LLM | Claude API (con HIPAA BAA) |
| Agents | LangGraph |
| Speech | Whisper v3 (self-hosted GPU) |
| Cloud | AWS (HIPAA BAA) |
| IaC | Terraform (NUNCA CloudFormation) |
| CI/CD | GitHub Actions |
| Multi-tenancy | Schema-per-tenant + per-tenant KMS |
| Observability | Langfuse (LLM), CloudWatch (infra) |

---

## MAPA DE ARCHIVOS CLAVE

### Ejecucion (que hacer)
| Archivo | Proposito |
|---------|-----------|
| `03-projects/PRE-DEV-CHECKLIST.md` | Todo ANTES de Sprint 0 |
| `03-projects/PHASE-1-EXECUTION-PLAN.md` | 117 tasks, dia-a-dia |
| `03-projects/EPIC-001` a `EPIC-011` | Detalle por area (001-006 Foundation, 007 MCP SDK, 008 Demo Polish, 009 Revenue Cycle Completion, 010 Security & Pilot Readiness, 011 Launch & Go-Live) |

### Arquitectura (como hacerlo)
| Archivo | Decision |
|---------|----------|
| `04-architecture/adr/ADR-001-*` | FHIR-native JSONB en PostgreSQL |
| `04-architecture/adr/ADR-002-*` | Schema-per-tenant + RLS + KMS |
| `04-architecture/adr/ADR-003-*` | LangGraph + Claude agents |
| `04-architecture/adr/ADR-004-*` | FastAPI backend structure |
| `04-architecture/adr/ADR-005-*` | HIPAAFastMCP hybrid MCP SDK approach |
| `04-architecture/system-design/System-Architecture-Overview.md` | Arquitectura completa |

### Domain Knowledge (por que asi)
| Archivo | Dominio |
|---------|---------|
| `05-domain/standards/FHIR-R4-Deep-Dive.md` | FHIR R4: recursos, search, profiles |
| `05-domain/standards/X12-EDI-Deep-Dive.md` | X12: 270, 837P, 835 con ejemplos |
| `05-domain/regulatory/HIPAA-Deep-Dive.md` | HIPAA completo + Day 1 checklist |
| `05-domain/regulatory/SOC2-HITRUST-Roadmap.md` | Compliance timeline + costos |
| `05-domain/billing/Revenue-Cycle-Deep-Dive.md` | RCM lifecycle completo |
| `05-domain/billing/Prior-Authorization-Deep-Dive.md` | PA automation + Da Vinci PAS |
| `05-domain/clinical/Ambient-AI-Documentation.md` | Pipeline ambient AI |
| `05-domain/clinical/Clinical-Workflows-Overview.md` | Patient journey map |
| `05-domain/clinical/Patient-Engagement-Patterns.md` | RPM, telehealth, scheduling |
| `05-domain/clinical/Population-Health-Analytics.md` | HCC, HEDIS, MIPS |

### Sprint 2 Files
| Archivo | Proposito |
|---------|-----------|
| `03-projects/EPIC-007-*` | MCP SDK refactoring + Sprint 2 scope |
| `04-architecture/adr/ADR-005-*` | HIPAAFastMCP decision |

### Sprint 3 Files
| Archivo | Proposito |
|---------|-----------|
| `03-projects/EPIC-008-*` | Demo polish: Approvals UI, WebSocket, agent runner, intake workflow |

### Sprint 4 Files
| Archivo | Proposito |
|---------|-----------|
| `03-projects/EPIC-009-*` | Revenue cycle completion: X12 837P, scrubbing, 835 parser, payment posting, claims MCP tools |

### Sprint 5 Files
| Archivo | Proposito |
|---------|-----------|
| `03-projects/EPIC-010-*` | Security hardening & pilot readiness: HIPAA risk assessment, pen test, onboarding wizard, EHR bridge, training |

### Sprint 6 Files
| Archivo | Proposito |
|---------|-----------|
| `03-projects/EPIC-011-*` | Launch & go-live: prod deploy, demo data, monitoring rotation, communication plan, pilot onboarding |
| `09-handoffs/operations/monitoring-rotation-runbook.md` | Daily monitoring procedures, alert response, rotation schedule |
| `09-handoffs/operations/pilot-communication-plan.md` | Communication channels, check-in cadence, escalation matrix, stakeholder updates |

### Engineering (guias de implementacion)
| Archivo | Guia |
|---------|------|
| `06-engineering/infrastructure/AWS-HIPAA-Infrastructure.md` | Terraform + AWS setup |
| `06-engineering/security/Auth-SMART-on-FHIR.md` | Auth0, RBAC, audit trail |
| `06-engineering/security/NextJS-Healthcare-Frontend.md` | Server Components, PHI safety |
| `06-engineering/api-specs/LangGraph-Agent-Implementation.md` | 5 agents con codigo |

### Sprint 5 Security & Compliance
| Archivo | Proposito |
|---------|-----------|
| `04-architecture/system-design/System-Architecture-Overview.md` | Security middleware, field encryption, monitoring sections |
| `04-architecture/system-design/agent-architecture.md` | Agent Security Model (credential injection, PHI policies, safety pipeline) |
| `03-projects/EPIC-010-security-pilot-readiness.md` | Sprint 5 task tracking (12 tasks) |

---

## REGLAS DE OBSIDIAN

### Notas
1. **SIEMPRE** frontmatter YAML con `type`, `date`, `tags`
2. **SIEMPRE** `[[wikilinks]]` para notas internas, `[text](url)` solo para URLs externas
3. **SIEMPRE** poner notas en la carpeta correcta (ver tabla abajo)
4. **NUNCA** crear notas sueltas en root (excepto CLAUDE.md y HEALTHCARE_OS_MASTERPLAN.md)
5. **NUNCA** borrar frontmatter existente

### Donde va cada nota
| Tipo | Carpeta | Template |
|------|---------|----------|
| Ideas rapidas | `00-inbox/` | ninguno |
| Work log diario | `01-daily/YYYY/MM/` | `tpl-daily` |
| Reuniones | `02-meetings/` | `tpl-meeting` |
| Epics/features | `03-projects/` | `tpl-epic` |
| ADRs | `04-architecture/adr/` | `tpl-adr` |
| Healthcare knowledge | `05-domain/<sub>/` | `tpl-domain` |
| Bug investigations | donde corresponda | `tpl-bug` |
| Retros | `08-retrospectives/` | libre |

### Tags (kebab-case, en frontmatter)
```
#daily #meeting #adr #epic #bug #domain #retro
#module-a ... #module-h #frontend #backend #infra #ai
#phase-1 #phase-2 #phase-3
#clinical #regulatory #billing #standards #fhir #hipaa #x12
#urgent #blocked #review-needed #decision #learning #risk
```

---

## REGLAS DE CODIGO (para cuando empecemos Sprint 0)

### Repos
- `medos-platform` -- monorepo: FastAPI backend + Next.js frontend
- `medos-terraform` -- infraestructura IaC

### Convenciones
- **Python:** FastAPI + async, type hints everywhere, ruff linter
- **Frontend:** TypeScript strict, Server Components por default, PHI NUNCA en Client Components
- **Public routes:** `/docs` served from `(public)` route group -- no auth required, for external agent integration docs
- **FHIR:** todo recurso se almacena como JSONB nativo, NUNCA modelo relacional traducido
- **Tests:** pytest (backend), Vitest + Playwright (frontend)
- **Commits:** conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`)
- **Secrets:** AWS Secrets Manager, NUNCA en codigo, NUNCA en .env commiteado

### Healthcare compliance en codigo
- **NUNCA** PHI en logs (los 18 HIPAA identifiers)
- **NUNCA** PHI en error messages
- **NUNCA** PHI en Client Components de Next.js
- **SIEMPRE** audit trail (FHIR AuditEvent) para acceso a PHI
- **SIEMPRE** tenant isolation verificada en cada request
- **SIEMPRE** confidence scoring en outputs de AI (< 0.85 = human review)

---

## PLUGINS OBSIDIAN

| Plugin | Status | Para que |
|--------|--------|----------|
| Dataview | Instalado | Queries en MOCs |
| Templater | Instalado | Templates con variables |
| Calendar | Pendiente | Daily notes visual |
| Git | Pendiente | Backup automatico |

---

## VAULT MAINTENANCE PROTOCOL

When working in the MedOS Obsidian vault, proactively maintain architectural coherence:

- **Cross-reference updates:** When creating or modifying code, update corresponding vault docs (`04-architecture/`, `03-projects/`) with `[[wikilinks]]` to related files
- **Diagram freshness:** Keep Mermaid diagrams in `04-architecture/system-design/` in sync with actual code architecture. Update diagrams when adding new MCP servers, agents, or routes
- **Execution plan tracking:** Update task status in `PHASE-1-EXECUTION-PLAN.md` as tasks complete: `pending` -> `in-progress` -> `done`
- **ADR discipline:** For architectural decisions (framework changes, integration patterns), create a new ADR in `04-architecture/adr/` before implementing
- **Epic coverage:** Ensure every significant feature maps to an Epic in `03-projects/`
- **Wikilink hygiene:** Use `[[wikilinks]]` for internal references, never raw file paths
- **Frontmatter compliance:** Every new `.md` file gets YAML frontmatter with `type`, `date`, `tags`

---

## DOG-FOODING PROTOCOL

MedOS must use its own tools internally -- every feature we build should be demonstrated inside the platform itself.

- **Admin dashboard** should consume MCP tools for live data (not hardcoded mock data in production)
- **Agent monitoring** should use the platform's own observability stack (Langfuse, health_dashboard)
- **Project tracker** at `/project` should track MedOS's own development (141 tasks, all sprints)
- **Context freshness** should monitor its own contexts -- the freshness system monitors itself
- **Device integration** dashboard should display real data when connected to test devices
- **Billing pipeline** should process test claims through the full X12 pipeline end-to-end
- **Every new feature** gets a corresponding demo scenario in the E2E suite proving it works inside the platform

This is not optional. If we build a feature and do not use it ourselves, we will not catch real usability issues before pilot practices do. The platform is both the product and its own best test environment.

---

## QUICK REFERENCE

```
Ctrl+O     Quick Switcher (abrir nota por nombre)
Ctrl+P     Command Palette
Ctrl+T     Insertar template
Ctrl+G     Graph View
Ctrl+E     Toggle edit/preview
[[         Iniciar wikilink
```
