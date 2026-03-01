---
type: epic
date: "2026-02-28"
status: in-progress
sprint: "3"
tags:
  - epic
  - frontend
  - admin
  - monitoring
  - phase-1
  - sprint-3
priority: 2
---

# EPIC-014: Admin System Monitoring (Phase 1)

> **Sprint:** 3 (Frontend Pages + Admin Phase 1)
> **Status:** In Progress
> **Dependencies:** [[EPIC-012-device-integration]], [[EPIC-013-context-rehydration]], [[ADR-006-patient-context-rehydration]], [[ADR-007-wearable-iot-integration]]

---

## Objective

Build the first phase of platform administration as **sub-pages under Settings** (`/settings/*`), providing visibility into device management, context freshness, and system health. Phase 2 (future) will extract these into an independent `admin.medos.com` subdomain with role-based access.

## Architecture Decision

**Phase 1 (Sprint 3):** Admin features live under `/settings/devices`, `/settings/context`, `/settings/system` — accessible to any authenticated provider in the practice. Uses mock data inline (no backend API calls yet).

**Phase 2 (Future):** Dedicated admin portal at `admin.medos.com` with:
- Super-admin role (practice manager + system admin)
- Real-time API integration (WebSocket for live metrics)
- Multi-tenant admin views
- Audit trail for admin actions

---

## Tasks

| ID | Task | Status | Description |
|----|------|--------|-------------|
| S3-T01 | Device Management Page | in-progress | `/settings/devices` — 3 tabs: Registered Devices, Readings, Alerts. Mock data matching device_server.py |
| S3-T02 | Context Freshness Dashboard | in-progress | `/settings/context` — 4 tabs: Patient Contexts, System Contexts, Dependency Graph, Rehydration Log |
| S3-T03 | System Health Dashboard | in-progress | `/settings/system` — 4 tabs: Overview, MCP Inventory (44 tools), Agent Performance, Cache & Events |
| S3-T04 | Settings Landing Update | pending | Add Device/Context/System link cards to `/settings` page |
| S3-T05 | E2E Demo Update | pending | Extend ACT 11 in full-demo.spec.ts to cover 3 new pages |
| S3-T06 | E2E Bug Fix | done | Scope claims analytics selector to `main` content area (line 326) |
| S3-T07 | Backend Commit | done | Commit 12 Sprint 2.5 backend files (device + context modules) |

---

## Page Specifications

### /settings/devices — Device Management

**Pattern:** Follows `settings/practice/page.tsx` — "use client", tabbed interface, Back to Settings link, mock data inline.

**Tab 1 — Registered Devices:**
- Cards for 3 mock devices: Oura Ring Gen 4, Apple Watch Ultra 3, Dexcom G8 CGM
- Status badges (active/paused), last sync, Pause/Resume, Deregister buttons

**Tab 2 — Readings:**
- Table: Patient, Device, Metric, Value, Unit, LOINC, Timestamp
- Color coding: normal (green), warning (yellow), critical (red) per alert thresholds
- LOINC codes from [[ADR-007-wearable-iot-integration]]

**Tab 3 — Alerts:**
- Threshold breach alerts with severity (warning/critical)
- Acknowledge, Dismiss actions

### /settings/context — Context Freshness Dashboard

**Tab 1 — Patient Contexts:**
- Freshness score grid (6 patients x 7 context types)
- Color gradient: >= 0.90 green, >= 0.75 amber, < 0.75 red (STALE)

**Tab 2 — System Contexts:**
- Cards for 6 system context types: Scheduling, Payer Rules, Agent Config, Clinical Protocols, Formulary, Compliance
- Freshness score, TTL, urgency badge

**Tab 3 — Dependency Graph:**
- Static visualization of `DEPENDENCY_GRAPH` from [[ADR-006-patient-context-rehydration|context_rehydration.py]]
- 17 ChangeTypes → affected ContextTypes with urgency badges

**Tab 4 — Rehydration Log:**
- Timeline of recent rehydration events (mock, last 2 hours)
- Filters: change_type, status

### /settings/system — System Health Dashboard

**Tab 1 — Overview:**
- KPI cards: 44 MCP Tools, 3 Active Agents, 87% Cache Hit Rate, 0.82 System Freshness
- 6 MCP server health status with dots

**Tab 2 — MCP Inventory:**
- 44 tools grouped by 6 servers, with PHI level and approval badges
- Collapsible server sections

**Tab 3 — Agent Performance:**
- 3 agent cards: Clinical Scribe, Prior Auth, Denial Management
- Metrics: tasks completed, avg confidence, human review rate, avg duration
- Confidence distribution bar

**Tab 4 — Cache & Events:**
- Cache stats (hot/warm/cold), recent events, rehydration latency percentiles

---

## Acceptance Criteria

- [ ] All 3 pages render with 0 TypeScript errors
- [ ] Each page follows the `settings/practice/page.tsx` pattern
- [ ] Mock data is consistent with backend modules
- [ ] Color coding follows MedOS design system
- [ ] E2E demo visits all 3 pages and their tabs
- [ ] Settings landing page links to all 3 new pages
- [ ] `npm run build` succeeds with 0 errors

---

## References

- [[EPIC-012-device-integration]] — Device MCP Server backend
- [[EPIC-013-context-rehydration]] — Context Rehydration Engine backend
- [[ADR-006-patient-context-rehydration]] — Context rehydration architecture
- [[ADR-007-wearable-iot-integration]] — Wearable/IoT integration architecture
- [[ADR-008-a2a-agent-communication]] — A2A protocol (agents visible in System Health)
