# CLAUDE.md - MedOS Healthcare OS

> Instrucciones operativas para Claude al trabajar en el proyecto MedOS.
> Actualizado: 2026-02-28

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
| Sprint | Semanas | Foco |
|--------|---------|------|
| S0 | W1-2 | AWS, CI/CD, Auth, DB, FastAPI, FHIR Patient CRUD |
| S1 | W3-4 | FHIR resources, patient matching, event bus |
| S2 | W5-6 | Whisper, Claude NLP, SOAP notes, provider UI |
| S3 | W7-8 | Eligibility, AI coding, claims (X12 837P) |
| S4 | W9-10 | Prior auth (X12 278), denials, analytics |
| S5 | W11-12 | Security hardening, pen test, onboarding |
| S6 | W13 | Prod deploy, pilot onboarding, go-live |

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
| `03-projects/EPIC-001` a `EPIC-006` | Detalle por area |

### Arquitectura (como hacerlo)
| Archivo | Decision |
|---------|----------|
| `04-architecture/adr/ADR-001-*` | FHIR-native JSONB en PostgreSQL |
| `04-architecture/adr/ADR-002-*` | Schema-per-tenant + RLS + KMS |
| `04-architecture/adr/ADR-003-*` | LangGraph + Claude agents |
| `04-architecture/adr/ADR-004-*` | FastAPI backend structure |
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

### Engineering (guias de implementacion)
| Archivo | Guia |
|---------|------|
| `06-engineering/infrastructure/AWS-HIPAA-Infrastructure.md` | Terraform + AWS setup |
| `06-engineering/security/Auth-SMART-on-FHIR.md` | Auth0, RBAC, audit trail |
| `06-engineering/security/NextJS-Healthcare-Frontend.md` | Server Components, PHI safety |
| `06-engineering/api-specs/LangGraph-Agent-Implementation.md` | 4 agents con codigo |

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

## QUICK REFERENCE

```
Ctrl+O     Quick Switcher (abrir nota por nombre)
Ctrl+P     Command Palette
Ctrl+T     Insertar template
Ctrl+G     Graph View
Ctrl+E     Toggle edit/preview
[[         Iniciar wikilink
```
