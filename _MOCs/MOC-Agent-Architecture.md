---
type: moc
date: "2026-02-28"
status: active
tags:
  - ai
  - architecture
  - moc
---

# Map of Content: AI Agent Architecture

> Navigation index for MedOS AI Agent documentation. The agent architecture is the core execution engine of the Healthcare OS, implementing bounded autonomy, human oversight, and full HIPAA compliance for AI-driven clinical and revenue cycle workflows.

Related: [[HEALTHCARE_OS_MASTERPLAN]] | [[System-Architecture-Overview]] | [[MOC-Architecture]]

---

## Overview

[[agent-architecture|Complete AI Agent Architecture Specification]] -- The definitive specification of MedOS's five core agents, the bounded autonomy framework, memory architecture, and inter-agent communication protocol.

**Key principles:**
- Agents are stateful LangGraph workflows, not chatbots
- All agents operate under confidence thresholds with mandatory human checkpoints
- Clinical and financial decisions always require human review
- Complete audit trail via FHIR Provenance and AuditEvent
- HIPAA BAA coverage via AWS Bedrock + Claude

---

## The 5 Core Agents

### 1. Clinical Documentation Agent (AI Scribe)

[[agent-architecture#2.1 Clinical Documentation Agent (AI Scribe)|Detailed Specification]]

**Module:** B (Provider Workflow Engine)
**Pipeline:** Audio → Transcript → Clinical NLU → SOAP Note → ICD-10/CPT Codes
**Key capability:** Listens to provider-patient encounters and generates structured clinical documentation with suggested diagnoses and procedures.

**Constraints:**
- Cannot diagnose (extracts only)
- Cannot prescribe
- Cannot finalize notes without provider approval
- Cannot access records beyond current encounter

---

### 2. Prior Authorization Agent

[[agent-architecture#2.2 Prior Authorization Agent|Detailed Specification]]

**Module:** C (Revenue Cycle) + D (Payer Integration)
**Pipeline:** PA Requirement Detection → Clinical Evidence Gathering → Form Generation → Submission → Status Tracking
**Key capability:** Automates prior authorization workflows, detecting requirements, gathering evidence, and managing submissions (saves 14.6 hours/week per practice).

**Constraints:**
- Cannot fabricate clinical data
- Cannot bypass medical necessity criteria
- Cannot submit without human approval
- Cannot guarantee approval

---

### 3. Denial Management Agent

[[agent-architecture#2.3 Denial Management Agent|Detailed Specification]]

**Module:** C (Revenue Cycle)
**Pipeline:** Denial Receipt → Root Cause Analysis → Appeal Strategy → Appeal Draft → Resubmission
**Key capability:** Analyzes claim denials, identifies root causes, calculates financial impact, and manages appeals (50-70% of appeals succeed, yet 65% of denials are never appealed).

**Constraints:**
- Cannot alter clinical documentation
- Cannot create false claims
- Cannot submit appeals without human approval
- Cannot fabricate evidence

---

### 4. Patient Communication Agent

[[agent-architecture#2.4 Patient Communication Agent|Detailed Specification]]

**Module:** F (Patient Engagement)
**Pipeline:** Event Trigger → Message Generation → Channel Selection (SMS/Email/Portal) → Delivery → Response Handling
**Key capability:** Manages proactive and reactive patient communications (appointment reminders, pre-visit instructions, billing inquiries, basic FAQ responses).

**Constraints:**
- Cannot diagnose or provide medical advice
- Cannot triage emergencies (must display "Call 911" and alert staff)
- Cannot share clinical results
- Cannot impersonate a human

---

### 5. Quality Reporting Agent

[[agent-architecture#2.5 Quality Reporting Agent|Detailed Specification]]

**Module:** E (Population Health & Analytics)
**Pipeline:** Measure Definition → Patient Population → Gap Identification → Report Generation → Submission
**Key capability:** Calculates quality measures (HEDIS, MIPS, CMS Star Ratings), identifies care gaps, predicts readmission risk.

**Constraints:**
- Cannot modify patient records (read-only)
- Cannot directly contact patients
- Cannot alter quality measure definitions
- Cannot submit reports without human approval

---

## Framework & Decision Records

### Core Architecture Decision

[[ADR-003-ai-agent-framework|ADR-003: AI Agent Framework Selection]]

Decision: **LangGraph + Claude + MCP** (vs. CrewAI, AutoGen, custom)

Rationale:
- LangGraph provides explicit state machines with guaranteed human checkpoints
- Claude via AWS Bedrock provides HIPAA BAA coverage for PHI
- Model Context Protocol (MCP) standardizes tool integration and enables third-party AI apps

---

### Related Architecture Decisions

- [[ADR-001-fhir-native-data-model|ADR-001: FHIR-Native Data Model]] -- The canonical data structure consumed by all agents
- [[ADR-004-fastapi-backend-architecture|ADR-004: FastAPI Backend Architecture]] -- Backend serving agent endpoints

---

## Domain Knowledge & Research

### Memory Architecture

[[context-rotting-and-agent-memory|Context Rotting and Agent Memory]]

Understanding how agent context degrades during long sessions and how the three-tier memory model (Redis → pgvector → FHIR store) maintains freshness.

**Key concepts:**
- Short-term memory (Redis): Session state, TTL 2 hours
- Mid-term memory (pgvector): Embedded patient summaries, vector similarity search
- Long-term memory (FHIR store): Golden source of truth, permanent retention

---

### ML Drift & Monitoring

[[ml-drift-monitoring|ML Drift Monitoring]]

How to detect when agent models degrade over time and trigger retraining pipelines.

**Drift types monitored:**
- Data drift (input distribution changes)
- Concept drift (model accuracy decay)
- Output drift (hallucination rate, confidence trends)

**Metrics:**
- `agent.confidence.mean` (alert if < 0.80)
- `agent.escalation.rate` (alert if > 40%)
- `agent.correction.rate` (alert if > 5%)

---

### HIPAA Compliance for AI

[[claude-for-healthcare-hipaa|Claude for Healthcare: HIPAA Deep Dive]]

How agents operate within HIPAA constraints using Claude via AWS Bedrock with Business Associate Agreement coverage.

**Key controls:**
- Minimum necessary principle: agents receive only required FHIR resources
- Full audit trail via Provenance and AuditEvent
- Per-tenant encryption with KMS
- 6+ year retention for agent audit logs
- Breach detection and escalation

---

## Engineering Integration

### MCP Integration Plan

[[mcp-integration-plan|Model Context Protocol Integration Strategy]]

How agents access tools, data, and external services through standardized MCP servers.

**MCP servers in scope:**
- FHIR-MCP Server (read/write clinical data)
- Scheduling MCP Server (appointment management)
- Billing/Claims MCP Server (X12 278, 835 handling, payer APIs)
- Prior Auth MCP Server (payer rule lookups)
- Analytics MCP Server (measure definitions, benchmarking)
- Terminology MCP Server (ICD-10-CM, CPT, SNOMED CT)

---

### Bedrock & Claude Setup

[[bedrock-claude-setup|AWS Bedrock + Claude Configuration for Healthcare]]

Infrastructure setup for:
- HIPAA BAA compliance verification
- Subnet isolation (agents in private subnets only)
- KMS encryption for each tenant
- CloudWatch logging and monitoring
- Cost optimization (on-demand vs. provisioned throughput)

---

## Key Concepts

### Bounded Autonomy Framework

From [[agent-architecture#4. Bounded Autonomy Framework|Bounded Autonomy Framework]] in agent-architecture:

Every agent action routes based on confidence score:

```
[>= 0.85] Auto-execute → Audit Log → Output
[0.70-0.85] Execute + Flag for Review → Human checkpoint
[< 0.70] Halt + Escalate to Human → Always human decides
[Critical action] ALWAYS escalate regardless of confidence
```

### Agent Communication Protocol

From [[agent-architecture#3. Agent Communication Protocol|Agent Communication Protocol]] in agent-architecture:

Agents communicate through an event bus (Redis Streams → Kafka at scale):

- `encounter.documented` (Clinical Doc → Quality Reporting)
- `coding.approved` (Clinical Doc → Prior Auth)
- `pa.required` (Prior Auth → Payer Integration)
- `claim.denied` (Revenue Cycle → Denial Management)
- `care.gap.detected` (Quality Reporting → Patient Communication)

---

## Observability & Monitoring

### LLM Tracing

Via Langfuse, every LLM call is traced with:
- Input/output token counts
- Latency (TTFB, total duration)
- Model version and cost
- Confidence score components
- Prompt template version (hash)

### Agent-Level Metrics

| Metric | Alert Threshold | Action |
|--------|-----------------|--------|
| `agent.confidence.mean` | < 0.80 | Investigate model drift |
| `agent.escalation.rate` | > 40% | Model underperforming |
| `agent.correction.rate` | > 5% | Lower auto-approve threshold |
| `agent.latency.p95` | > 30s | Optimize pipeline |
| `agent.error.rate` | > 2% | Investigate errors |

---

## Implementation Roadmap

### Phase 1 (Months 0-6): Foundation
- Base agent framework (LangGraph + PostgreSQL checkpointer)
- HIPAA compliance layer
- Clinical Documentation Agent (audio → SOAP)
- Confidence routing engine
- Provider review UI

### Phase 2 (Months 6-12): Revenue Agents
- Prior Authorization Agent
- Denial Management Agent
- Patient Communication Agent
- Inter-agent event bus
- Billing staff review UI

### Phase 3 (Months 12-18): Intelligence
- Quality Reporting Agent
- ML drift monitoring
- Cross-agent workflow orchestration
- Custom MCP servers
- Agent analytics dashboard

### Phase 4 (Months 18-24): Platform
- Third-party agent marketplace
- Tenant-customizable workflows
- Multi-model routing
- Advanced memory architecture

---

## Future Agents (Planned)

Placeholder for agents in planning phase:

- **Peer-to-Peer Review Agent** -- Schedules and facilitates P2P reviews with payers
- **Care Coordinator Agent** -- Risk stratification and care plan generation
- **Revenue Forecasting Agent** -- Predictive analytics for claim success rates
- **Provider Efficiency Coach** -- Benchmarks provider workflows against peers
- **Compliance Agent** -- Tracks regulatory updates and alerts on policy changes

---

## Domain Knowledge (Informs Agent Behavior)

These domain documents define the clinical and business workflows that agents automate:

- [[Ambient-AI-Documentation]] -- What the Clinical Documentation Agent replicates
- [[Prior-Authorization-Deep-Dive]] -- What the Prior Auth Agent automates
- [[Revenue-Cycle-Deep-Dive]] -- Revenue cycle that Denial Management Agent supports
- [[Clinical-Workflows-Overview]] -- Clinical workflows agents operate within
- [[Population-Health-Analytics]] -- Quality measures the Quality Reporting Agent calculates
- [[Patient-Engagement-Patterns]] -- Communication patterns the Patient Comms Agent follows

---

## Compliance & Security

- [[HIPAA-Deep-Dive]] -- HIPAA requirements that constrain all agent behavior
- [[claude-for-healthcare-hipaa]] -- Claude-specific HIPAA considerations
- [[SOC2-HITRUST-Roadmap]] -- Compliance certifications roadmap
- [[agent-architecture#7. Security & Compliance]] -- Agent-specific security controls

---

## Related Navigational Hubs

- [[MOC-Architecture]] -- Complete architecture of all MedOS systems
- [[MOC-Projects]] -- Active projects and sprints
- [[MOC-Domain-Knowledge]] -- Clinical workflows, standards, regulatory knowledge
- [[HEALTHCARE_OS_MASTERPLAN]] -- 90-day roadmap and vision

---

## Dataview: Auto-Listed AI-Tagged Notes

```dataview
TABLE type, status, date
FROM #ai
SORT date DESC
LIMIT 50
```

---

## Recent Updates

- **2026-02-28**: Expanded MOC-Agent-Architecture with comprehensive sections
- **2026-02-28**: Completed agent-architecture.md specification (all 5 agents)
- **2026-02-27**: ADR-003 finalized (LangGraph selection)

---

## Quick Links for Common Tasks

**Getting started with agents:**
1. Read [[HEALTHCARE_OS_MASTERPLAN#The Core 5 Agents|5 Agents overview]] for business context
2. Review [[ADR-003-ai-agent-framework|ADR-003]] to understand why we chose LangGraph
3. Deep dive: [[agent-architecture|Complete agent specification]]

**Working on a specific agent:**
1. Find the agent section in [[agent-architecture]]
2. Review its constraints, escalation rules, and FHIR resources
3. Check [[mcp-integration-plan]] for MCP server dependencies
4. Review [[context-rotting-and-agent-memory]] if dealing with context management

**Monitoring & troubleshooting:**
1. Check [[ml-drift-monitoring]] for drift detection
2. Review Langfuse metrics in the observability dashboard
3. Check [[claude-for-healthcare-hipaa]] for compliance concerns
4. Review audit trails in the FHIR AuditEvent query
