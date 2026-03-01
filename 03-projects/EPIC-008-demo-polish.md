---
type: epic
date: "2026-03-01"
status: done
priority: 1
tags:
  - epic
  - sprint-3
  - demo
  - frontend
  - backend
  - phase-1
owner: ""
target-date: "2026-04-25"
---

# EPIC-008: Demo Polish -- Sprint 3 UI Integration & Workflow Orchestration

> **Timeline:** Weeks 7-8 (Sprint S3)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-007-mcp-sdk-refactoring]] (done), [[ADR-005-mcp-sdk-integration]]
> **Blocks:** [[EPIC-005-revenue-cycle-mvp]], [[EPIC-006-pilot-readiness]]

## Objective

Deliver a demo-ready platform by connecting the Next.js frontend to real backend APIs, building the Approvals UI (the demo centerpiece), implementing WebSocket-based real-time agent events, creating an agent runner API, orchestrating the patient intake workflow end-to-end, and enriching seed data for compelling demonstrations. This epic bridges the gap between Sprint 2's backend capabilities (32 MCP tools, 3 agents, approval workflow) and a polished, interactive frontend experience.

Sprint 2 deliverables: 174 tests, 32 MCP tools, 3 LangGraph agents, approval workflow API, frontend `/docs` page. See [[EPIC-007-mcp-sdk-refactoring]] for full Sprint 2 scope.

---

## Timeline (Gantt)

```
Week 7 (Sprint 3, Part 1)
|----- T1: Approvals page (frontend) -----------|
|---- T2: Connect frontend to real APIs ---------|
|---- T3: WebSocket real-time agent events ------|
|--------- T4: Agent runner API endpoint --------|

Week 8 (Sprint 3, Part 2)
|---- T5: Patient intake workflow (backend) -----|
|---- T6: Enhanced demo seed data ---------------|
|------------- T7: Dashboard agent stats --------|
|------------- T8: Vault enrichment -------------|
```

---

## Tasks

### T1: Build Approvals Page (Frontend)
**Complexity:** L
**Estimate:** 8h
**Dependencies:** [[EPIC-007-mcp-sdk-refactoring]] T9 (Approval workflow API)
**References:** [[agent-architecture]], [[ADR-005-mcp-sdk-integration]]

**Description:**
Build the Approvals page in the Next.js frontend -- the demo star. This page shows pending agent outputs (SOAP notes, prior auth requests, appeal letters) waiting for human review. Each approval card shows the agent type, confidence score, a summary of the output, and approve/reject/modify controls. Real-time updates via WebSocket (T3).

**Acceptance Criteria:**
- [x] `/approvals` route with responsive layout
- [x] Approval cards display: agent type, timestamp, confidence score, patient reference, summary
- [x] Approve button sends `POST /api/v1/approvals/{id}/approve`
- [x] Reject button opens reason modal, sends `POST /api/v1/approvals/{id}/reject`
- [x] Modify button opens inline editor, sends `POST /api/v1/approvals/{id}/modify`
- [x] Real-time badge count updates via WebSocket
- [x] Filter by agent type (Clinical Scribe, Prior Auth, Denial Management)
- [x] Confidence score displayed with color coding (green >= 0.85, yellow 0.70-0.85, red < 0.70)
- [x] Server Component for PHI data; Client Component only for interactive controls

---

### T2: Connect Frontend Pages to Real Backend APIs
**Complexity:** L
**Estimate:** 6h
**Dependencies:** T1, [[EPIC-007-mcp-sdk-refactoring]]
**References:** [[System-Architecture-Overview]]

**Description:**
Replace all mock data in existing frontend pages (Patients, Encounters, Dashboard, AI Scribe) with real API calls to the FastAPI backend. Implement proper error handling, loading states, and tenant context headers.

**Acceptance Criteria:**
- [x] Patients page: `GET /fhir/r4/Patient` with search, pagination
- [x] Encounters page: `GET /fhir/r4/Encounter` with patient linking
- [x] Dashboard: `GET /api/v1/analytics/dashboard/overview` for stats
- [x] AI Scribe: `POST /api/v1/agent-tasks` to trigger scribe agent
- [x] All API calls include `Authorization` and `X-Tenant-ID` headers
- [x] Loading skeletons while data fetches
- [x] Error boundaries with user-friendly messages (no PHI in errors)
- [x] Stale-while-revalidate caching with SWR or React Query

---

### T3: WebSocket Real-Time Agent Events (Backend)
**Complexity:** M
**Estimate:** 4h
**Dependencies:** None (builds on existing FastAPI)
**References:** [[agent-architecture]], [[mcp-integration-plan]]

**Description:**
Implement WebSocket endpoint for real-time agent event streaming. When an agent processes a task (transcription, SOAP note generation, coding, approval creation), events are broadcast to connected frontend clients. This replaces polling and enables the "live agent activity" experience in the demo.

**Acceptance Criteria:**
- [x] `WS /ws/agent-events` endpoint on FastAPI
- [x] Events streamed: `agent.started`, `agent.step_completed`, `agent.output_ready`, `agent.approval_created`
- [x] Events include: agent_type, step_name, progress_pct, timestamp, task_id
- [x] Tenant-isolated: clients only receive events for their tenant
- [x] Auto-reconnect logic in frontend WebSocket client
- [x] Connection authenticated via JWT token in query params or first message
- [x] Events persisted in Redis Streams for replay on reconnect (last 100 events)

---

### T4: Agent Runner API Endpoint (Backend)
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T3
**References:** [[agent-architecture]], [[mcp-integration-plan]]

**Description:**
Create a unified API endpoint to trigger any LangGraph agent by name with input parameters. This replaces agent-specific trigger routes and provides a standard interface for the frontend to invoke agents. Supports async execution with status polling and WebSocket updates.

**Acceptance Criteria:**
- [x] `POST /api/v1/agents/run` accepts `{ agent_type, input_params, callback_url? }`
- [x] Returns `{ task_id, status: "queued" }` immediately
- [x] `GET /api/v1/agents/tasks/{task_id}` returns current status and output
- [x] Supported agents: `clinical_scribe`, `prior_auth`, `denial_management`
- [x] Agent execution emits WebSocket events (T3)
- [x] Rate limiting: max 10 concurrent agent tasks per tenant
- [x] Input validation per agent type (e.g., scribe requires encounter_id)

---

### T5: Patient Intake Workflow (Backend)
**Complexity:** L
**Estimate:** 6h
**Dependencies:** T4, [[EPIC-007-mcp-sdk-refactoring]] T5 (Billing MCP), T6 (Scheduling MCP)
**References:** [[Clinical-Workflows-Overview]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Implement the end-to-end patient intake workflow that demonstrates MedOS's value proposition in a single flow: Patient check-in -> Eligibility verification -> Schedule confirmation -> Encounter creation -> AI documentation -> Coding suggestion -> Approval queue. This is the workflow the demo walks through.

**Acceptance Criteria:**
- [x] `POST /api/v1/workflows/patient-intake` triggers the full workflow
- [x] Step 1: Verify patient demographics (FHIR Patient lookup)
- [x] Step 2: Run eligibility check (Billing MCP `billing_check_eligibility`)
- [x] Step 3: Confirm/create appointment (Scheduling MCP `scheduling_book`)
- [x] Step 4: Create Encounter (FHIR MCP `fhir_create`)
- [x] Step 5: Trigger Clinical Scribe agent (Agent Runner T4)
- [x] Each step emits WebSocket events (T3)
- [x] Workflow state persisted for resume on failure
- [x] Demo mode: can run with seed data without external clearinghouse

---

### T6: Enhanced Demo Seed Data
**Complexity:** M
**Estimate:** 3h
**Dependencies:** T5
**References:** [[Clinical-Workflows-Overview]], [[FHIR-R4-Deep-Dive]]

**Description:**
Create rich, realistic seed data that makes the demo compelling. Multiple patients with different scenarios (orthopedic injury, dermatology follow-up, chronic disease management), pre-populated encounters, and staged approval items at different confidence levels.

**Acceptance Criteria:**
- [x] 5+ realistic patients with FHIR-compliant demographics (no real PHI)
- [x] 3+ encounter scenarios: new patient ortho consult, dermatology follow-up, chronic disease check-up
- [x] Pre-generated SOAP notes at various review stages (draft, pending review, approved)
- [x] Pre-populated approval queue with items at different confidence levels (0.65, 0.78, 0.92)
- [x] Eligibility data for demo insurance plans
- [x] Seed script: `python -m medos.seed_demo` populates all data
- [x] Idempotent: running seed twice does not create duplicates

---

### T7: Dashboard Agent Stats Widget
**Complexity:** S
**Estimate:** 3h
**Dependencies:** T2, T4
**References:** [[System-Architecture-Overview]]

**Description:**
Add an agent activity widget to the main dashboard showing real-time agent stats: tasks processed today, average confidence, approval queue depth, time saved estimate. This widget makes the dashboard feel "alive" and demonstrates the AI backbone.

**Acceptance Criteria:**
- [x] Widget on `/dashboard` showing: tasks today, avg confidence, pending approvals, estimated time saved
- [x] Data from `GET /api/v1/analytics/agent-stats`
- [x] Auto-refresh every 30 seconds (or via WebSocket)
- [x] Sparkline chart for tasks processed over last 7 days
- [x] Click-through to `/approvals` from pending approvals count

---

### T8: Vault Enrichment
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T1-T7

**Description:**
Update the Obsidian vault documentation to reflect Sprint 3 progress: update execution plan, enrich architecture docs with new diagrams (approval workflow, WebSocket flow, full data flow), enrich sparse domain docs with MedOS implementation examples, add cross-references with wikilinks.

**Acceptance Criteria:**
- [x] PHASE-1-EXECUTION-PLAN.md updated with Sprint 3 section
- [x] Mermaid diagrams added to [[agent-architecture]] and [[System-Architecture-Overview]]
- [x] Domain docs enriched with practical MedOS examples
- [x] Cross-references added between ADRs, EPICs, and architecture docs
- [x] CLAUDE.md updated with Sprint 3 files
- [x] All new/modified files have valid YAML frontmatter

---

## Dependencies Map

```
T3 (WebSocket) ──> T4 (Agent Runner)
T4 ──> T5 (Patient Intake Workflow)
T5 ──> T6 (Seed Data)
EPIC-007 T9 ──> T1 (Approvals Page)
T1 ──> T2 (Frontend API Connection)
T2 + T4 ──> T7 (Dashboard Stats)
T1-T7 ──> T8 (Vault Enrichment)
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| Approvals UI (T1) | [[EPIC-006-pilot-readiness]] |
| Patient Intake Workflow (T5) | [[EPIC-006-pilot-readiness]], Demo script |
| Agent Runner API (T4) | All future agent-triggered workflows |
| WebSocket Events (T3) | [[EPIC-006-pilot-readiness]] (real-time UX) |
| Seed Data (T6) | [[EPIC-006-pilot-readiness]] (demo environments) |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| WebSocket connection instability | Medium | Medium | Auto-reconnect with exponential backoff, fallback to polling |
| Frontend-backend API contract mismatch | Medium | Low | TypeScript types generated from OpenAPI schema, contract tests |
| Seed data not realistic enough for demo | Low | Medium | Validate with clinical advisor, use real CPT/ICD-10 codes |
| Agent runner performance under load | Low | High | Rate limiting per tenant, async execution, queue management |
| Approval workflow UX complexity | Medium | Medium | Start with simple approve/reject, iterate based on feedback |

---

## Definition of Done

- [x] Approvals page functional with real API data
- [x] All existing frontend pages connected to real APIs (no mock data)
- [x] WebSocket agent events streaming in real-time
- [x] Agent runner API operational for all 3 agents
- [x] Patient intake workflow end-to-end functional
- [x] Demo seed data creates a compelling walkthrough scenario
- [x] Dashboard shows live agent stats
- [x] Vault documentation updated with Sprint 3 content and diagrams
- [x] All tests passing (target: 200+)
- [x] `ruff check` clean

---

## References

- [[EPIC-007-mcp-sdk-refactoring]] -- Sprint 2 scope (prerequisite)
- [[ADR-005-mcp-sdk-integration]] -- HIPAAFastMCP design decision
- [[agent-architecture]] -- Agent framework and bounded autonomy
- [[System-Architecture-Overview]] -- Full system architecture
- [[mcp-integration-plan]] -- MCP tool catalog and gateway
- [[Clinical-Workflows-Overview]] -- Patient journey map
- [[Revenue-Cycle-Deep-Dive]] -- Revenue cycle business processes
- [[FHIR-R4-Deep-Dive]] -- FHIR resource specifications
