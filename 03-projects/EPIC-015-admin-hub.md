---
type: epic
date: "2026-03-01"
status: in-progress
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
| Section | Route | Status |
|---------|-------|--------|
| Admin Dashboard | `/admin/dashboard` | done |
| Monitoring & Observability | `/admin/monitoring` | in-progress |
| Project Management | `/admin/project` | in-progress |

### Practice
| Section | Route | Status |
|---------|-------|--------|
| Tenant Management | `/admin/tenants` | in-progress |
| User & Role Management | `/admin/users` | in-progress |
| Billing Configuration | `/admin/billing` | in-progress |
| Integration Management | `/admin/integrations` | in-progress |

### Platform
| Section | Route | Status |
|---------|-------|--------|
| MCP Server Management | `/admin/mcp` | in-progress |
| AI Agent Configuration | `/admin/agents` | in-progress |
| System Configuration | `/admin/system` | in-progress |
| Feature Flags | `/admin/features` | in-progress |

### Compliance
| Section | Route | Status |
|---------|-------|--------|
| Security & Compliance | `/admin/security` | in-progress |
| Data Management | `/admin/data` | in-progress |

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
| S3F-T14 | Monitoring & Observability (5 tabs) | in-progress |
| S3F-T15 | Project Management (migrate from /project) | in-progress |
| S3F-T16 | Tenant Management (3 tabs + onboarding wizard) | in-progress |
| S3F-T17 | User & Role Management (5 tabs + RBAC matrix) | in-progress |
| S3F-T18 | Billing Configuration (5 tabs) | in-progress |
| S3F-T19 | Integration Management (4 tabs) | in-progress |
| S3F-T20 | MCP Server Management (4 tabs + 44 tools) | in-progress |
| S3F-T21 | AI Agent Configuration (4 tabs) | in-progress |
| S3F-T22 | System Configuration (4 tabs) | in-progress |
| S3F-T23 | Feature Flags (4 categories) | in-progress |
| S3F-T24 | Security & Compliance (5 tabs + HIPAA dashboard) | in-progress |
| S3F-T25 | Data Management (5 tabs) | in-progress |
| S3F-T26 | E2E test update for admin sections | pending |
| S3F-T27 | E2E video recording with all admin sections | pending |

---

## Acceptance Criteria

- [ ] All 13 admin sections render with mock data
- [ ] Admin sidebar navigates between all sections
- [ ] Sidebar collapses/expands
- [ ] Tab navigation works within each section
- [ ] Tables have hover states and proper styling
- [ ] MedOS design system applied consistently
- [ ] TypeScript compiles with 0 errors
- [ ] E2E test covers all admin routes
- [ ] E2E video demonstrates full admin hub

---

## Estimated Size

~8,900 lines across 15 files (13 pages + layout + root redirect)

## Dog-Fooding

The admin hub demonstrates MedOS eating its own dog food:
- Dashboard metrics come from MCP tool schemas
- Agent config mirrors actual LangGraph agent parameters
- MCP inventory reflects real 44-tool architecture
- Security compliance mirrors actual HIPAA controls
- Project tracker uses real 141 tasks from execution plan
