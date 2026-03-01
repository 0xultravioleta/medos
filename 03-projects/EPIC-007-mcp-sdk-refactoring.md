---
type: epic
date: "2026-02-28"
status: done
priority: 2
tags:
  - project
  - epic
  - phase-1
  - ai
  - mcp
  - backend
owner: ""
target-date: "2026-04-11"
---

# EPIC-007: MCP SDK Refactoring + Sprint 2 Servers & Agents

> **Timeline:** Weeks 5-6 (Sprint S2)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-003-fhir-data-layer]], [[EPIC-004-ai-clinical-documentation]]
> **Blocks:** [[EPIC-005-revenue-cycle-mvp]], [[EPIC-006-pilot-readiness]]

## Objective

Refactor the custom MCP implementation to use the official MCP Python SDK (FastMCP) via a `HIPAAFastMCP` subclass, migrate existing servers to `@hipaa_tool` decorators, build two new MCP servers (Billing + Scheduling), implement two new LangGraph agents (Prior Auth + Denial Management), and deliver the approval workflow API. This epic combines SDK modernization with Sprint 2 feature delivery.

See [[ADR-005-mcp-sdk-integration]] for the architectural decision behind the hybrid approach.

---

## Timeline (Gantt)

```
Week 5 (Sprint 2, Part 1)
|----- T1: HIPAAFastMCP subclass + decorators ----|
|---- T2: Migrate FHIR server to @hipaa_tool -----|
|---- T3: Migrate Scribe server to @hipaa_tool ---|
|--------- T4: Mount FastMCP + cleanup ------------|
|------------- T5: Billing MCP server -------------|

Week 6 (Sprint 2, Part 2)
|---- T6: Scheduling MCP server -------------------|
|---- T7: Prior Auth LangGraph agent --------------|
|--------- T8: Denial Management agent ------------|
|------------- T9: Approval workflow API ----------|
|------------- T10: Schema updates + migrations ---|
|------------------ T11: Frontend /docs pages -----|
|------------------ T12: Vault documentation ------|
|------------------ T13: Tests + verification -----|
```

---

## Tasks

### T1: Create HIPAAFastMCP Subclass + @hipaa_tool Decorator
**Complexity:** L
**Estimate:** 6h
**Dependencies:** None
**References:** [[ADR-005-mcp-sdk-integration]]

**Description:**
Create `HIPAAFastMCP(FastMCP)` that overrides `call_tool()` to inject the HIPAA security pipeline (auth validation, PHI policy check, rate limiting, audit logging). Create `@hipaa_tool` decorator for tool registration with healthcare metadata.

**Acceptance Criteria:**
- [x] `HIPAAFastMCP` subclass passes all existing MCP tests
- [x] `@hipaa_tool` decorator registers tools with PHI access level and required scopes
- [x] Security pipeline executes on every tool call
- [x] Audit events generated for all tool invocations

---

### T2: Migrate FHIR Server to @hipaa_tool Decorators
**Complexity:** M
**Estimate:** 3h
**Dependencies:** T1
**References:** [[mcp-integration-plan]]

**Description:**
Migrate all 12 FHIR MCP tools from tuple-based registration to `@hipaa_tool` decorators. Verify that all tools maintain identical behavior.

**Acceptance Criteria:**
- [x] All 12 FHIR tools registered via `@hipaa_tool`
- [x] Existing FHIR MCP tests pass without modification
- [x] Tool input schemas auto-generated from type hints

---

### T3: Migrate Scribe Server to @hipaa_tool Decorators
**Complexity:** M
**Estimate:** 2h
**Dependencies:** T1
**References:** [[agent-architecture]]

**Description:**
Migrate all 6 Scribe MCP tools from tuple-based registration to `@hipaa_tool` decorators.

**Acceptance Criteria:**
- [x] All 6 Scribe tools registered via `@hipaa_tool`
- [x] Existing Scribe MCP tests pass without modification

---

### T4: Mount FastMCP + Cleanup transport.py + Simplify mcp_sse.py
**Complexity:** M
**Estimate:** 3h
**Dependencies:** T2, T3

**Description:**
Mount HIPAAFastMCP instances on the FastAPI app using `mcp.mount()`. Delete custom `transport.py` (~176 lines). Simplify `mcp_sse.py` to delegate to FastMCP's built-in SSE transport.

**Acceptance Criteria:**
- [x] `transport.py` deleted
- [x] FastMCP mounted on FastAPI app
- [x] SSE endpoint operational via FastMCP transport
- [x] All integration tests pass

---

### T5: Create Billing MCP Server (8 tools)
**Complexity:** L
**Estimate:** 6h
**Dependencies:** T1
**References:** [[mcp-integration-plan]], [[Revenue-Cycle-Deep-Dive]], [[X12-EDI-Deep-Dive]]

**Description:**
Build the Billing/Claims MCP server with 8 tools: `billing_check_eligibility`, `billing_submit_claim`, `billing_claim_status`, `billing_parse_remittance`, `billing_denial_lookup`, `billing_submit_appeal`, `billing_patient_balance`, `billing_payer_rules`.

**Acceptance Criteria:**
- [x] 8 tools registered via `@hipaa_tool`
- [x] Financial tools (`submit_claim`, `submit_appeal`) marked `requires_human_approval: true`
- [x] X12 270/271, 837P, 835 handling via structured data (agents see structured, not raw EDI)
- [x] CARC/RARC code lookup returns human-readable descriptions
- [x] Tests for all 8 tools

---

### T6: Create Scheduling MCP Server (6 tools)
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T1
**References:** [[mcp-integration-plan]], [[Clinical-Workflows-Overview]]

**Description:**
Build the Scheduling MCP server with 6 tools: `scheduling_available_slots`, `scheduling_book`, `scheduling_reschedule`, `scheduling_cancel`, `scheduling_waitlist_add`, `scheduling_no_show_predict`.

**Acceptance Criteria:**
- [x] 6 tools registered via `@hipaa_tool`
- [x] Returns FHIR `Slot` and `Appointment` resources
- [x] Provider availability rules enforced
- [x] Tests for all 6 tools

---

### T7: Create Prior Auth LangGraph Agent
**Complexity:** L
**Estimate:** 6h
**Dependencies:** T5
**References:** [[agent-architecture]], [[Prior-Authorization-Deep-Dive]]

**Description:**
Build the Prior Authorization agent as a LangGraph `StateGraph` with states: CheckPARequirement -> GatherEvidence -> GenerateJustification -> CreatePAForm -> SubmitForApproval. All submissions require human approval.

**Acceptance Criteria:**
- [x] Agent correctly identifies PA-required procedures
- [x] Clinical evidence gathered from FHIR store
- [x] Medical necessity narrative generated by Claude
- [x] X12 278 request form generated
- [x] Submission gated by human approval
- [x] All outputs include confidence scores

---

### T8: Create Denial Management LangGraph Agent
**Complexity:** L
**Estimate:** 6h
**Dependencies:** T5
**References:** [[agent-architecture]], [[Revenue-Cycle-Deep-Dive]]

**Description:**
Build the Denial Management agent as a LangGraph `StateGraph` with states: AnalyzeDenial -> AssessAppealViability -> GatherEvidence -> DraftAppealLetter -> SubmitForApproval. All appeal submissions require human approval.

**Acceptance Criteria:**
- [x] Agent parses denial reason codes (CARC/RARC)
- [x] Appeal viability assessed with confidence score
- [x] Clinical evidence gathered for appeal
- [x] Appeal letter drafted by Claude
- [x] Submission gated by human approval
- [x] Legitimate denials correctly identified and reported

---

### T9: Create Approval Workflow Router + API
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T7, T8

**Description:**
Build the approval workflow API that allows staff to approve, reject, or modify agent-generated submissions (PA requests, appeal letters, claims). Includes queue management and SLA tracking.

**Acceptance Criteria:**
- [x] `POST /api/v1/approvals` - list pending approvals
- [x] `POST /api/v1/approvals/{id}/approve` - approve submission
- [x] `POST /api/v1/approvals/{id}/reject` - reject with reason
- [x] `POST /api/v1/approvals/{id}/modify` - approve with modifications
- [x] SLA alerts for aging approvals

---

### T10: Schema Updates + Database Migrations
**Complexity:** S
**Estimate:** 2h
**Dependencies:** T5, T6

**Description:**
Add database tables/columns needed by Billing and Scheduling servers. Run Alembic migrations.

**Acceptance Criteria:**
- [x] Billing-related tables created
- [x] Scheduling-related tables created
- [x] Migrations reversible
- [x] No data loss on existing tables

---

### T11: Build Frontend /docs Pages
**Complexity:** M
**Estimate:** 4h
**Dependencies:** T5, T6, T7, T8

**Description:**
Build documentation pages in the Next.js frontend showing MCP tool catalog, agent capabilities, and architecture diagrams (Mermaid rendered client-side).

**Acceptance Criteria:**
- [x] `/docs` page with MCP tool catalog (32 tools)
- [x] Agent capability cards with status indicators
- [x] Mermaid diagrams rendered for architecture

---

### T12: Update Obsidian Vault Documentation
**Complexity:** M
**Estimate:** 3h
**Dependencies:** T1-T8

**Description:**
Create ADR-005, EPIC-007, update system architecture docs, add Mermaid diagrams to existing docs, update CLAUDE.md and execution plan.

**Acceptance Criteria:**
- [x] ADR-005 created with decision rationale
- [x] EPIC-007 created with full task breakdown
- [x] Mermaid diagrams added to agent-architecture.md and System-Architecture-Overview.md
- [x] CLAUDE.md updated with Sprint 2 file map
- [x] All wikilinks valid

---

### T13: Write All Tests + Final Verification
**Complexity:** M
**Estimate:** 4h
**Dependencies:** ALL

**Description:**
Write tests for all new code. Run full test suite. Verify ~120+ tests passing, ruff clean.

**Acceptance Criteria:**
- [x] Tests for HIPAAFastMCP subclass
- [x] Tests for all 14 new tools (8 Billing + 6 Scheduling)
- [x] Tests for Prior Auth agent state machine
- [x] Tests for Denial Management agent state machine
- [x] Tests for approval workflow API
- [x] Full suite: ~120+ tests passing
- [x] `ruff check` clean

---

## Dependencies Map

```
T1 (HIPAAFastMCP) ──> T2 (FHIR migration)
                   └──> T3 (Scribe migration)
                   └──> T5 (Billing server)
                   └──> T6 (Scheduling server)
T2 + T3 ──> T4 (Mount + cleanup)
T5 ──> T7 (Prior Auth agent)
T5 ──> T8 (Denial Mgmt agent)
T5 + T6 ──> T10 (Schema updates)
T7 + T8 ──> T9 (Approval workflow)
T5-T8 ──> T11 (Frontend docs)
T1-T8 ──> T12 (Vault docs)
ALL ──> T13 (Tests)
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| Billing MCP Server (T5) | [[EPIC-005-revenue-cycle-mvp]] |
| Scheduling MCP Server (T6) | [[EPIC-005-revenue-cycle-mvp]], [[EPIC-006-pilot-readiness]] |
| Prior Auth Agent (T7) | [[EPIC-005-revenue-cycle-mvp]] |
| Denial Management Agent (T8) | [[EPIC-005-revenue-cycle-mvp]] |
| Approval Workflow (T9) | [[EPIC-006-pilot-readiness]] |
| HIPAAFastMCP (T1) | All future MCP servers |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| FastMCP SDK breaking changes | Low | Medium | Pin SDK version, test before upgrading |
| Subclass override fragility | Medium | Medium | Keep override thin, test SDK internal calls |
| Billing server X12 complexity | Medium | High | Use structured data layer; agents never see raw EDI |
| Agent confidence calibration | Medium | Medium | Start with conservative thresholds, tune with real data |
| Approval workflow UX complexity | Low | Medium | Simple approve/reject/modify pattern, iterate based on feedback |

---

## Definition of Done

- [x] HIPAAFastMCP subclass working with FastMCP SDK
- [x] 32 MCP tools registered (12 FHIR + 6 Scribe + 8 Billing + 6 Scheduling)
- [x] 3 LangGraph agents operational (Clinical Scribe + Prior Auth + Denial Management)
- [x] Approval workflow API operational with queue management
- [x] Frontend `/docs` page with tool catalog and architecture diagrams
- [x] All tests passing (~120+ total)
- [x] `ruff check` clean
- [x] Vault documentation updated (ADR-005, EPIC-007, diagrams)
- [x] `transport.py` deleted, custom JSON-RPC code removed
