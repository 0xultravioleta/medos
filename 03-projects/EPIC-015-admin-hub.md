---
type: epic
date: "2026-03-01"
status: done
tags:
  - epic
  - frontend
  - admin
  - module-g
  - phase-1
---

# EPIC-015: Admin Hub (/admin)

> Unified administration hub consolidating platform management, project tracking, security compliance, system observability, and tenant configuration into a single surface.

**Sprint:** S3F (Frontend Phase 2)
**Owner:** A + AI
**Related:** [[admin-page-design-analysis]] | [[EPIC-014-admin-system-monitoring]] | [[System-Architecture-Overview]]

---

## Scope

13 admin sections organized into 4 navigation groups:

### Operations
| Section | Route | Lines | Status |
|---------|-------|-------|--------|
| Admin Dashboard | `/admin/dashboard` | 475 | done |
| Monitoring & Observability | `/admin/monitoring` | 831 | done |
| Project Management | `/admin/project` | 1012 | done |

### Practice
| Section | Route | Lines | Status |
|---------|-------|-------|--------|
| Tenant Management | `/admin/tenants` | 759 | done |
| User & Role Management | `/admin/users` | 752 | done |
| Billing Configuration | `/admin/billing` | 686 | done |
| Integration Management | `/admin/integrations` | 839 | done |

### Platform
| Section | Route | Lines | Status |
|---------|-------|-------|--------|
| MCP Server Management | `/admin/mcp` | 789 | done |
| AI Agent Configuration | `/admin/agents` | 675 | done |
| System Configuration | `/admin/system` | 574 | done |
| Feature Flags | `/admin/features` | 520 | done |

### Compliance
| Section | Route | Lines | Status |
|---------|-------|-------|--------|
| Security & Compliance | `/admin/security` | 906 | done |
| Data Management | `/admin/data` | 743 | done |

---

## Architecture

- **Layout:** `/admin/layout.tsx` -- collapsible left sidebar with 4 grouped sections
- **Root redirect:** `/admin/page.tsx` -> `/admin/dashboard`
- **Auth guard:** admin role required (enforced in parent dashboard layout)
- **Data:** Inline mock data (Phase 1), MCP tool integration (Phase 2)
- **Pattern:** Same as existing settings pages -- "use client", MedOS CSS vars, lucide-react

---

## Tasks

| ID | Task | Status |
|----|------|--------|
| S3F-T11 | Admin layout + sidebar navigation | done |
| S3F-T12 | Admin Dashboard (executive overview) | done |
| S3F-T13 | Sidebar update (add Admin link) | done |
| S3F-T14 | Monitoring & Observability (5 tabs) | done |
| S3F-T15 | Project Management (migrate from /project) | done |
| S3F-T16 | Tenant Management (3 tabs + onboarding wizard) | done |
| S3F-T17 | User & Role Management (5 tabs + RBAC matrix) | done |
| S3F-T18 | Billing Configuration (5 tabs) | done |
| S3F-T19 | Integration Management (4 tabs) | done |
| S3F-T20 | MCP Server Management (4 tabs + 44 tools) | done |
| S3F-T21 | AI Agent Configuration (4 tabs) | done |
| S3F-T22 | System Configuration (4 tabs) | done |
| S3F-T23 | Feature Flags (4 categories) | done |
| S3F-T24 | Security & Compliance (5 tabs + HIPAA dashboard) | done |
| S3F-T25 | Data Management (5 tabs) | done |
| S3F-T26 | E2E test update for admin sections | done |
| S3F-T27 | E2E video recording with all admin sections | in-progress |

---

## Acceptance Criteria

- [x] All 13 admin sections render with mock data
- [x] Admin sidebar navigates between all sections
- [x] Sidebar collapses/expands
- [x] Tab navigation works within each section
- [x] Tables have hover states and proper styling
- [x] MedOS design system applied consistently
- [x] TypeScript compiles with 0 errors
- [x] E2E test covers all admin routes (ACT 13, 14 Acts total)
- [ ] E2E video demonstrates full admin hub (recording in progress)

---

## Estimated Size

9,732 lines across 15 files (13 pages + layout + root redirect) -- exceeded estimate by 9%

## Dog-Fooding

The admin hub demonstrates MedOS eating its own dog food:
- Dashboard metrics come from MCP tool schemas
- Agent config mirrors actual LangGraph agent parameters
- MCP inventory reflects real 44-tool architecture
- Security compliance mirrors actual HIPAA controls
- Project tracker uses real 141 tasks from execution plan
