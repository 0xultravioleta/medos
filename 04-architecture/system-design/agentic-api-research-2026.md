---
type: domain
date: 2026-02-28
tags: [architecture, ai, mcp, a2a, fhir, agents, research]
status: complete
---

# Agentic API + MCP Layer Research Synthesis (2026)

> Comprehensive research findings for building an agent-native healthcare OS. This document synthesizes the current state of agentic protocols (MCP, A2A), agent operating systems (OpenClaw, IronClaw), healthcare-specific extensions (HMCP, FHIR MCP), security patterns, and competitive landscape -- all applied to MedOS's architectural pivot from UI-first to agent-first.

**Related:** [[agent-architecture]] | [[mcp-integration-plan]] | [[ADR-003-ai-agent-framework]] | [[System-Architecture-Overview]] | [[HIPAA-Deep-Dive]] | [[FHIR-R4-Deep-Dive]] | [[claude-for-healthcare-hipaa]]

---

## 1. Executive Summary

MedOS is undergoing an architectural pivot. The original design centered on human-facing UIs -- a Next.js frontend for providers, dashboards for billing staff, patient portals. That architecture is valid but insufficient. The industry is shifting toward agent-first systems where AI agents are the primary consumers of APIs, and human interfaces are rendered on top of agent outputs rather than built as standalone CRUD applications.

The strategic insight: **nobody in healthcare has MCP + A2A natively**. Epic, Cerner, athenahealth -- all are adding AI features incrementally on top of legacy architectures. None have been designed from the ground up as agent-native platforms. MedOS has the opportunity to be the first healthcare OS where agents are first-class citizens, not bolt-ons.

**Target market:** Mid-size specialty practices (5-30 providers) in Florida, starting with orthopedics or dermatology. These practices are underserved by enterprise EHRs (too expensive, too complex) and current small-practice solutions (limited AI, no agent capabilities). MedOS fills the gap with an AI-native platform that reduces administrative burden through autonomous agents operating under strict healthcare constraints.

**Capital:** $100M family office (Miami, FL). Pitch to Dr. Di Reze (Technical Operations). The technical differentiation must be demonstrable: agents that work, protocols that interoperate, compliance that is built-in rather than bolted-on.

**Key thesis:** The convergence of MCP (standardized tool access), A2A (agent discovery and collaboration), FHIR R5 Subscriptions (event-driven triggers), Claude for Healthcare (HIPAA-ready LLM), and LangGraph (deterministic state machines) creates a technology stack that makes agent-native healthcare OS feasible for a two-person team augmented by AI tools.

---

## 2. Agent Operating Systems Landscape

### 2.1 OpenClaw: Validating the "Agent OS" Thesis

OpenClaw, created by PSPDFKit founder Peter Steinberger, is an open-source personal AI agent that went viral in late January 2026. As of February 2026, it has accumulated over 68,000 GitHub stars (reports of 216K may include forks and derivative projects). On February 14, 2026, Steinberger announced he would be joining OpenAI, with the project moving to an open-source foundation.

OpenClaw validates a critical thesis: **users want agents that operate across their digital lives, not isolated chatbots**. The platform functions as an agentic interface for autonomous workflows, accessed via messaging services (Signal, Telegram, Discord, WhatsApp), with local data storage enabling persistent and adaptive behavior.

Key characteristics relevant to MedOS:

| Feature | OpenClaw | MedOS Parallel |
|---------|----------|---------------|
| Persistent memory | Local storage, session history | PostgreSQL + pgvector, FHIR-native state |
| Tool execution | 100+ AgentSkills, shell commands | MCP servers, FHIR operations, billing tools |
| Multi-channel | Signal, Telegram, Discord | Provider portal, patient app, EHR integration |
| Orchestration | Simple skill chaining | LangGraph state machines with human checkpoints |
| Compliance | None (consumer product) | HIPAA, SOC 2, HITRUST (healthcare mandate) |

**Strategic takeaway:** OpenClaw proves the market wants agent OS platforms. MedOS applies the same concept to healthcare with the compliance, determinism, and audit trails the domain requires.

### 2.2 IronClaw: The Security Blueprint

IronClaw is a security-focused Rust reimplementation of OpenClaw, built by NEAR AI and launched at NEARCON on February 24, 2026. While OpenClaw prioritizes ease of use and viral adoption, IronClaw prioritizes defense-in-depth -- making it the more relevant reference architecture for healthcare.

IronClaw's security innovations directly applicable to MedOS:

**WASM Sandboxing.** Every untrusted tool runs in an isolated WebAssembly container with capability-based permissions. Tools must explicitly opt into HTTP access, secret access, and tool invocation. This is the most significant security innovation in the agent OS space and something OpenClaw does not have.

**Host-Boundary Credential Injection.** Credentials are never passed into the WASM sandbox. The execution pipeline is:

```
WASM tool -> Allowlist check -> Leak scan -> Credential injection -> Execute -> Leak scan -> Response
```

A malicious tool must defeat every layer to exfiltrate credentials. This pattern maps directly to MedOS's requirement that PHI never enters the LLM context window unnecessarily.

**Network Isolation.** IronClaw ships with configurable network profiles: loopback-only, LAN, or remote. Network exposure is an explicit opt-in, not a default. For MedOS, this translates to agents that can only reach approved endpoints (FHIR servers, payer APIs, terminology services) and nothing else.

**Applicability to MedOS:** IronClaw's patterns -- WASM isolation, credential injection at host boundary, leak detection, and capability-based permissions -- should be adopted for MedOS's agent runtime. While MedOS uses Python (FastAPI) rather than Rust, the architectural patterns are language-agnostic. See Section 8 for detailed security architecture.

### 2.3 Healthcare as Greenfield for Agentic Systems

Healthcare is simultaneously the industry most in need of agent automation and the most resistant to it. The result is a massive greenfield opportunity:

- **70% of hospitals** report integration as the top barrier to adopting automation tools
- **62.6% of Epic hospitals** had adopted ambient AI documentation by June 2025, proving demand exists
- **72% of AI-using practices** rely on ambient scribes as their primary AI tool, but this is only one narrow use case
- Prior authorization, claims management, denial appeals, quality reporting, and care coordination remain largely manual

The practices that adopt agent-native platforms first will have compounding advantages: lower administrative costs, faster reimbursement cycles, better quality scores, and happier providers.

---

## 3. MCP (Model Context Protocol) for Healthcare

### 3.1 Protocol Foundation

MCP is a JSON-RPC 2.0 based open protocol, originally created by Anthropic and now community-governed, that standardizes how AI models interact with external tools and data sources. It has been described as "FHIR for AI" -- just as FHIR standardized how healthcare systems exchange data, MCP standardizes how AI interacts with those systems.

Core capabilities:

| Capability | Description | Healthcare Application |
|-----------|-------------|----------------------|
| **Tools** | Callable functions with typed parameters | FHIR CRUD, claim submission, eligibility check |
| **Resources** | Read-only structured data access | Patient records, formularies, fee schedules |
| **Prompts** | Context-specific prompt templates | Clinical note generation, coding guidelines |
| **Sampling** | Server-initiated LLM requests | Agent-driven analysis within MCP server context |

Transport options:
- **stdio** for local integrations (agent and MCP server on the same machine)
- **SSE (Server-Sent Events)** for remote access over HTTP, which is the primary mode for MedOS where agents run in ECS containers connecting to centralized MCP servers
- **Streamable HTTP** (newer transport option for request-response patterns)

### 3.2 FHIR MCP Servers: Open-Source Ecosystem

Multiple open-source FHIR MCP servers have emerged, validating the pattern:

| Project | Maintainer | Key Features |
|---------|-----------|-------------|
| **AWS HealthLake MCP Server** | AWS Labs | Native integration with HealthLake FHIR store; supports Patient, Condition, Observation, MedicationRequest queries |
| **Flexpa mcp-fhir** | Flexpa | Generic FHIR R4 MCP server; connects to any FHIR-compliant API |
| **WSO2 fhir-mcp-server** | WSO2 | Exposes any FHIR server or API as an MCP server; configurable resource types |
| **MCP-FHIR Framework** | Academic (arXiv 2506.13800) | Research framework for clinical decision support and EHR insights through LLMs and MCP |

**MedOS approach:** Rather than using a generic FHIR MCP server, MedOS will build a custom FHIR-MCP server that enforces tenant isolation (schema-per-tenant from [[ADR-002-multi-tenancy-schema-per-tenant]]), HIPAA audit logging (FHIR AuditEvent for every access), and minimum-necessary PHI filtering (agents only receive the FHIR resources they need for their current task). Generic servers expose too much surface area for healthcare compliance.

### 3.3 HMCP: Healthcare Model Context Protocol

Innovaccer introduced HMCP (Healthcare Model Context Protocol) as a specialized extension of MCP specifically crafted for healthcare AI agents. HMCP is available as an open-source specification and SDK on GitHub (innovaccer/Healthcare-MCP).

HMCP adds healthcare-specific layers on top of base MCP:

- **SMART on FHIR authorization**: OAuth 2.0 + OpenID Connect following the SMART on FHIR framework, providing standardized, secure access to clinical data
- **Patient context isolation**: Ensures healthcare AI agents maintain proper patient context isolation and security, preventing cross-patient data leakage
- **Data minimization**: Only necessary and permitted data is shared with AI agents, aligning with HIPAA's minimum necessary standard
- **Audit trails**: Built-in support for access control, encrypted data handling, and audit logging at the protocol level
- **Rate limiting and risk assessment**: Protocol-level controls to prevent agent overuse or suspicious patterns

**MedOS approach:** Adopt HMCP's specification as the foundation for MedOS's MCP servers, contributing back any orthopedics/dermatology-specific extensions. This aligns MedOS with an emerging industry standard rather than building proprietary compliance layers.

### 3.4 MedOS MCP Server Architecture

Six MCP servers planned for MedOS, each with bounded responsibilities:

| MCP Server | Tools Exposed | Resources | Primary Agent Consumer |
|-----------|---------------|-----------|----------------------|
| **FHIR Server** | Patient CRUD, Encounter management, Observation read/write, Condition management | Patient bundles, clinical summaries | All agents |
| **Scribe Server** | Start/stop recording, generate SOAP note, suggest ICD-10/CPT codes, submit for review | Audio buffer, transcript, note templates | Clinical Scribe Agent |
| **Billing Server** | Eligibility check (X12 270/271), claim submission (X12 837P), ERA processing (X12 835) | Fee schedules, payer rules, denial history | Prior Auth, Denial Mgmt Agents |
| **Scheduling Server** | Slot availability, appointment create/modify/cancel, waitlist management | Provider schedules, room availability | Patient Comms Agent |
| **Analytics Server** | Quality measure calculation, population health queries, trend analysis | HEDIS/MIPS dashboards, HCC scores | Quality Reporting Agent |
| **Terminology Server** | ICD-10 lookup, CPT search, SNOMED mapping, drug interaction check | Code sets, crosswalks, formularies | Clinical Scribe, Billing Agents |

Each server implements HMCP guardrails: tenant isolation, audit logging, PHI filtering, and SMART on FHIR scopes.

---

## 4. A2A (Agent-to-Agent) Protocol

### 4.1 Protocol Overview

Google unveiled the Agent2Agent (A2A) Protocol on April 9, 2025, as an open standard for agent interoperability. It is now maintained as an open-source project under the Linux Foundation with Apache 2.0 license, backed by 50+ technology partners.

A2A addresses a different problem than MCP:
- **MCP** = how agents access tools and data (agent-to-system)
- **A2A** = how agents discover and collaborate with each other (agent-to-agent)

These protocols are complementary, not competing. MedOS needs both.

### 4.2 Core A2A Concepts

**Agent Cards.** Agents advertise their capabilities using an Agent Card, a JSON document served at `/.well-known/agent.json`. A client agent uses Agent Cards to identify the best agent for a task and initiate communication.

Example MedOS Agent Card for the Clinical Scribe:

```json
{
  "name": "MedOS Clinical Scribe",
  "description": "Converts provider-patient encounters into structured SOAP notes with ICD-10 and CPT coding suggestions",
  "url": "https://api.medos.health/agents/scribe",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "ambient-documentation",
      "name": "Ambient Clinical Documentation",
      "description": "Real-time encounter capture and SOAP note generation",
      "inputModes": ["audio/webm", "text/plain"],
      "outputModes": ["application/fhir+json", "text/markdown"]
    }
  ],
  "authentication": {
    "schemes": ["oauth2"],
    "credentials": "/.well-known/oauth-authorization-server"
  }
}
```

**Task Lifecycle.** Communication between agents is oriented toward task completion. Tasks have a defined lifecycle and can be completed immediately (synchronous) or span extended periods (asynchronous with status updates). This maps well to healthcare workflows where a prior authorization request might take days and involve multiple agent interactions.

**Protocol Layers.** A2A has a layered specification:
- **Layer 2:** Fundamental capabilities and behaviors that agents must support
- **Layer 3:** Concrete mappings to protocol bindings (JSON-RPC, gRPC, HTTP/REST)

Version 0.2 adds support for stateless interactions and standardized authentication based on OpenAPI-like schemas.

### 4.3 A2A in MedOS: Agent Discovery and Delegation

A2A enables three critical patterns for MedOS:

**Internal agent delegation.** The Prior Auth Agent discovers it needs clinical documentation. Instead of hard-coding a function call, it uses A2A to discover and delegate to the Clinical Scribe Agent, which may have been updated independently.

**External agent integration.** A clearinghouse (Availity) publishes an A2A Agent Card for their claims processing agent. MedOS's Billing Agent discovers it and delegates claim submission, receiving asynchronous status updates as the claim moves through adjudication.

**Marketplace enablement.** Third-party developers build specialized agents (e.g., a dermatology-specific coding agent, an orthopedic outcome tracker). They register Agent Cards in MedOS's marketplace. Practices discover and enable agents through A2A, not custom integration.

### 4.4 NIST Agent Identity Standards (February 2026)

On February 17, 2026, NIST's Center for AI Standards and Innovation (CAISI) announced the **AI Agent Standards Initiative** -- the first U.S. government framework specifically targeting autonomous AI agents. This is directly relevant to MedOS's compliance story.

The initiative addresses:

| Area | NIST Focus | MedOS Application |
|------|-----------|-------------------|
| **Identification** | Distinguishing AI agents from human users; managing metadata to control agent actions | Agent identity tokens with agent_type, tenant_id, delegation chain |
| **Authorization** | OAuth 2.0 extensions, policy-based access control for agent rights | SMART on FHIR scopes per agent, purpose_of_use claims |
| **Interoperability** | Secure human-agent and multi-agent interactions | A2A protocol adoption with NIST-aligned identity layer |
| **Accountability** | Audit trails for autonomous actions | FHIR AuditEvent + Provenance for every agent action |

NIST released a concept paper, "Accelerating the Adoption of Software and Artificial Intelligence Agent Identity and Authorization," proposing demonstrations of how identity practices apply to AI agents in enterprise settings. Public comments on the AI Agent Identity paper are due April 2, 2026.

**Strategic importance:** MedOS can align its agent identity architecture with NIST standards from day one, making regulatory conversations significantly easier. Being NIST-aligned is a differentiator when pitching to healthcare CISOs and compliance officers.

---

## 5. Claude Agent SDK Analysis

### 5.1 SDK Capabilities

Anthropic's Claude Agent SDK (available in Python, with community ports to Go and Rust) provides production-ready primitives for building agentic applications.

Key features relevant to MedOS:

**@tool Decorator.** Custom tools are implemented as in-process MCP servers running directly within the Python application, eliminating the need for separate processes. This is ideal for tools that need low-latency access to application state.

```python
from claude_agent_sdk import tool

@tool("check_eligibility", "Verify patient insurance eligibility", {
    "patient_id": str,
    "payer_id": str,
    "service_date": str
})
async def check_eligibility(patient_id: str, payer_id: str, service_date: str):
    # Implementation with FHIR + X12 270/271
    ...
```

**PreToolUse / PostToolUse Hooks.** Callback functions that fire during agent execution at specific points:
- **PreToolUse**: Runs before a tool is called. Can block dangerous operations, inject credentials at the host boundary (IronClaw pattern), validate PHI access authorization, and log intent.
- **PostToolUse**: Runs after a tool returns. Can sanitize PHI from responses, validate output format, trigger audit events, and detect anomalies.

These hooks are the mechanism for implementing IronClaw-style credential injection and PHI filtering without modifying individual tool implementations.

**Subagent Pattern.** The SDK supports spawning subagents for lower-stakes tasks. A supervising agent can delegate to a subagent with more restricted permissions. This maps to MedOS's pattern where the Clinical Scribe Agent (high-stakes, requires provider review) might spawn a terminology lookup subagent (low-stakes, fully automated).

### 5.2 SDK vs. LangGraph: Decision

The Claude Agent SDK and LangGraph serve different purposes:

| Dimension | Claude Agent SDK | LangGraph |
|-----------|-----------------|-----------|
| **Orchestration model** | Sequential tool calls with hooks | Explicit state machine graphs with named nodes and edges |
| **State persistence** | Session-based | PostgreSQL checkpointer with full replay |
| **Human-in-the-loop** | Hooks can pause/block | Native interrupt nodes with state serialization |
| **LLM portability** | Claude-only | LLM-agnostic (can swap Claude, GPT, Gemini at any node) |
| **Determinism** | Emergent from hooks | Explicit from graph topology |
| **Healthcare fit** | Good for simple tool-calling workflows | Superior for multi-day workflows (prior auth) needing guaranteed checkpoints |

**Decision:** Keep LangGraph as the orchestration layer (per [[ADR-003-ai-agent-framework]]). Adopt Claude Agent SDK patterns selectively:
1. Use `@tool` decorator style for MCP tool definitions within LangGraph nodes
2. Implement PreToolUse/PostToolUse hook patterns in LangGraph's node pre/post processors for credential injection and PHI filtering
3. Use subagent pattern for low-stakes delegations within a LangGraph graph (e.g., terminology lookups)

This gives MedOS the determinism and durability of LangGraph with the ergonomics and security patterns of the Claude Agent SDK.

---

## 6. LangGraph for Healthcare Agents

### 6.1 Why State Machines, Not Chatbots

LangGraph implements agents as deterministic state machines. As of February 2026, the latest version is 1.0.6 with improved durable execution support. The core insight, articulated in recent analyses: "If you understand finite state machines, you understand LangGraph." The shared state model means nothing is hidden, nothing implicit, and everything is traceable.

This is non-negotiable for healthcare:
- **Regulatory requirement**: HIPAA requires knowing exactly what data was accessed, by whom, when, and why. Chatbot-style agents with emergent behavior cannot satisfy this.
- **Clinical safety**: A prior authorization agent that "sometimes" follows the correct workflow is unacceptable. The workflow must be guaranteed.
- **Legal liability**: When an agent action leads to a denied claim or delayed treatment, the full execution trace must be available for review.

### 6.2 PostgreSQL Checkpointer for Durable State

LangGraph's PostgreSQL checkpointer (`langgraph-checkpoint-postgres==3.0.3`) persists agent state to the database at every node transition. This enables:

- **Crash recovery**: Agents resume from the last saved state after failures or restarts
- **Async workflows**: A prior authorization workflow that spans 3-5 business days maintains its state across the entire lifecycle
- **Human review**: When an agent reaches a confidence threshold below 0.85, the workflow pauses. A human reviewer sees the full state, makes a decision, and the workflow resumes from exactly where it stopped
- **Audit replay**: Any historical workflow can be replayed step-by-step for compliance review, malpractice investigation, or quality improvement
- **Session memory**: Agents remember context across multiple interactions with the same patient or case

This aligns with [[ADR-001-fhir-native-data-model]] -- agent state is stored alongside FHIR resources in the same PostgreSQL instance, enabling transactional consistency between agent actions and clinical data updates.

### 6.3 The 5 Core Agents

As defined in [[agent-architecture]], MedOS deploys five core agents, each implemented as a LangGraph `StateGraph`:

| Agent | Primary Function | Autonomy Level | Human Checkpoint |
|-------|-----------------|----------------|-----------------|
| **Clinical Scribe** | Ambient encounter capture, SOAP note generation, ICD-10/CPT coding suggestions | Medium | Provider must review and sign every note |
| **Prior Authorization** | Determine PA requirements, compile clinical evidence, submit requests, track status | Medium-High | Auto-submit for routine cases; human review for complex/high-value |
| **Denial Management** | Analyze denial reasons, draft appeal letters, track appeal outcomes | Medium | All appeals reviewed before submission |
| **Patient Communications** | Appointment reminders, pre-visit instructions, post-visit follow-up, care gap outreach | High | Templated messages auto-send; custom messages require review |
| **Quality Reporting** | HEDIS/MIPS measure calculation, care gap detection, HCC risk adjustment | High | Reports auto-generate; action recommendations require review |

Each agent's LangGraph graph includes mandatory nodes for:
1. **Input validation** (schema check, tenant verification)
2. **Authorization check** (SMART on FHIR scopes, purpose_of_use)
3. **PHI minimization** (load only necessary FHIR resources)
4. **AI processing** (Claude via Bedrock with confidence scoring)
5. **Confidence gate** (threshold-based routing to auto-execute, flag, or escalate)
6. **Audit logging** (FHIR AuditEvent creation)
7. **Output sanitization** (strip PHI from agent responses sent to UI)

### 6.4 LLM-Agnostic Node Design

A key advantage of LangGraph over Claude Agent SDK for orchestration: individual nodes can use different LLMs. While Claude (via AWS Bedrock) is the primary LLM, specific nodes might benefit from specialized models:

- **Terminology extraction node**: A fine-tuned model for ICD-10/CPT coding might outperform general-purpose Claude
- **OCR processing node**: For reading scanned insurance documents
- **Translation node**: For patient-facing communications in Spanish

The LangGraph graph topology remains constant; only the LLM call within a node changes. This protects against vendor lock-in and enables best-of-breed model selection at the node level.

---

## 7. FHIR R5 Subscriptions for Reactive Agents

### 7.1 Event-Driven Agent Triggers

Agents should not poll for work. They should react to events. FHIR R5 completely reworked the subscription framework, introducing topic-based subscriptions that standardize event-driven notifications aligned with modern publish/subscribe models.

Key R5 subscription improvements over R4:
- **SubscriptionTopic**: Defines what events can be subscribed to (e.g., "new encounter created", "lab result received")
- **Subscription**: A client's registration to receive notifications for a specific topic with filter criteria
- **SubscriptionStatus**: Heartbeat and handshake notifications for connection reliability
- **Batching**: Multiple notifications can be bundled into a single delivery

While FHIR R5 is the standard, R6 (expected late 2026) will bring even deeper event capabilities. The R5 Subscriptions Backport Implementation Guide enables R4 servers to support R5-style subscriptions, which is the path MedOS should take given R4 is the current industry standard.

### 7.2 MedOS Event Bus Architecture

Implementation: **Redis Streams** as the internal event bus, with FHIR Subscriptions as the external interface.

```
FHIR Server (write) -> PostgreSQL trigger -> Redis Streams -> Agent Router -> LangGraph Agent
```

Defined event types and their agent routing:

| Event | Trigger Condition | Agent Routed To | Action |
|-------|-------------------|----------------|--------|
| `encounter.completed` | Provider closes encounter | Clinical Scribe | Generate SOAP note draft |
| `coding.approved` | Provider signs coded note | Prior Auth | Check if services require PA |
| `claim.denied` | ERA (835) processed with denial | Denial Management | Analyze denial, draft appeal |
| `pa.approved` | Payer approves prior auth | Billing | Submit claim (837P) |
| `care.gap.detected` | Quality measure identifies gap | Patient Comms | Schedule outreach |
| `lab.result.received` | Lab sends ORU/FHIR Observation | Clinical Scribe + Patient Comms | Flag abnormals, notify patient |
| `appointment.no_show` | Check-in time passes without arrival | Patient Comms | Send follow-up, offer reschedule |
| `eligibility.changed` | Scheduled eligibility check fails | Billing | Alert front desk, flag appointments |

Each event carries a minimal payload (event type, resource ID, tenant ID) -- never PHI in the event bus itself. The routed agent fetches necessary FHIR resources through the FHIR MCP server with proper authorization.

### 7.3 External Event Sources

Beyond internal FHIR operations, MedOS agents should react to external events:

- **Payer portal updates**: PA decision arrived, claim status changed (via clearinghouse webhook or polling)
- **Lab interfaces**: Results delivered via HL7v2 or FHIR (converted to internal events)
- **Patient actions**: Portal message received, appointment request submitted, form completed
- **Regulatory updates**: New coverage determination published, quality measure updated

The Redis Streams architecture provides a unified internal bus regardless of external event source, simplifying agent routing logic.

---

## 8. Security Architecture (IronClaw Patterns Applied to Healthcare)

### 8.1 Host-Boundary Credential Injection

Adapted from IronClaw's pipeline for MedOS:

```
Agent Request -> Scope Validation -> PHI Filter -> Credential Injection -> MCP Tool Execute -> PHI Scan -> Audit Log -> Response
```

**Critical principle:** Secrets (API keys, database credentials, payer portal passwords) and unnecessary PHI never enter the LLM context window. The agent reasons about what action to take. The runtime injects credentials and executes the action outside the LLM's visibility.

Implementation in LangGraph:

```python
# PreToolUse hook (runs before every MCP tool call)
async def pre_tool_hook(state: AgentState, tool_call: ToolCall) -> ToolCall:
    # 1. Validate agent has SMART on FHIR scope for this tool
    validate_scopes(state.agent_identity, tool_call.tool_name)

    # 2. Filter PHI to minimum necessary
    tool_call.params = phi_filter(tool_call.params, state.agent_type)

    # 3. Inject credentials from AWS Secrets Manager (NOT from state)
    tool_call.credentials = await secrets_manager.get(
        f"medos/{state.tenant_id}/{tool_call.tool_name}"
    )

    # 4. Log intent (before execution)
    await audit_log(AuditEvent(
        type="tool_invocation",
        agent=state.agent_id,
        tool=tool_call.tool_name,
        patient_id=state.patient_id,  # if applicable
        purpose_of_use=state.purpose_of_use
    ))

    return tool_call
```

### 8.2 Minimum Necessary PHI Filter

Each agent type has a defined PHI access profile (aligned with HIPAA's minimum necessary standard from [[HIPAA-Deep-Dive]]):

| Agent | Allowed FHIR Resources | Denied FHIR Resources |
|-------|----------------------|---------------------|
| Clinical Scribe | Patient, Encounter, Condition, Observation, MedicationRequest, AllergyIntolerance, Procedure | Coverage, Claim, PaymentReconciliation, Account |
| Prior Auth | Patient (demographics only), Encounter, Condition, Procedure, Coverage, ServiceRequest | Full clinical history, psychiatric notes, substance abuse records |
| Denial Management | Claim, ClaimResponse, ExplanationOfBenefit, Coverage, Condition (relevant DX only) | Unrelated conditions, social history, family history |
| Patient Comms | Patient (name, contact, preferred language), Appointment, CarePlan | Clinical notes, financial data, insurance details |
| Quality Reporting | Aggregated/de-identified data, Measure, MeasureReport | Individual patient identifiers (unless calculating individual gaps) |

The PHI filter runs at the MCP server layer, not in the agent code. This ensures that even a compromised or misbehaving agent cannot access data outside its profile.

### 8.3 Multi-Stage Safety Layer

Inspired by IronClaw's defense-in-depth, adapted for healthcare:

| Stage | Action | Example |
|-------|--------|---------|
| **Block** | Prevent execution entirely | Agent attempts to access psychiatric notes without 42 CFR Part 2 authorization |
| **Warn** | Allow but flag for review | Agent confidence below 0.85 on ICD-10 code selection |
| **Review** | Queue for human approval | Agent drafts appeal letter for high-value claim (>$10,000) |
| **Sanitize** | Strip sensitive data from output | Remove SSN, DOB from agent response sent to UI layer |

### 8.4 Agent Identity Architecture

Each agent receives a JWT identity token containing:

```json
{
  "agent_id": "medos-scribe-001",
  "agent_type": "clinical_scribe",
  "tenant_id": "practice-456",
  "delegation_chain": ["provider-dr-smith", "medos-orchestrator"],
  "fhir_scopes": [
    "patient/Patient.read",
    "patient/Encounter.read",
    "patient/Condition.read",
    "patient/Observation.write"
  ],
  "purpose_of_use": "TREAT",
  "max_autonomy_level": "medium",
  "session_expiry": "2026-02-28T18:00:00Z"
}
```

This aligns with NIST's February 2026 AI Agent Standards Initiative requirements for agent identification, authorization, and accountability. The `delegation_chain` field tracks the human who authorized the agent's actions, the `purpose_of_use` maps to HIPAA's permitted uses (TREAT, PAYMENT, OPERATIONS), and FHIR scopes enforce resource-level access control.

### 8.5 Audit Trail: FHIR AuditEvent + Provenance

Every agent action generates two FHIR resources:

1. **AuditEvent**: Records what happened (who, what, when, where, outcome)
2. **Provenance**: Records the data lineage (what data was used, what was produced, what agent version)

These resources are stored in the same PostgreSQL database as clinical data (per [[ADR-001-fhir-native-data-model]]) and are immutable -- append-only, never updated or deleted.

---

## 9. Competitive Analysis

### 9.1 Enterprise EHRs

| Competitor | AI Strategy | MCP/A2A | Agent Architecture | Target Market | Weakness |
|-----------|------------|---------|-------------------|---------------|----------|
| **Epic** | Factory toolkit for custom AI agents; Art, Penny, Emmie built-in tools; 62.6% ambient AI adoption | No native MCP or A2A; SMART on FHIR only | Agent features embedded in monolithic EHR; not standalone | Large health systems (500+ beds) | Too expensive and complex for mid-size practices; 18-24 month implementations |
| **Oracle Cerner** | Clinical AI Agent across 30+ specialties; voice-first design on OCI; 30% documentation time reduction | No MCP or A2A | Embedded in OCI-native EHR rebuild | Large health systems | Major platform migration underway; customer uncertainty |
| **athenahealth** | FHIR Event Notifications (R5-style); ambient AI via partners | FHIR Subscriptions (no MCP) | Partner-dependent AI strategy | Mid-market (target overlap with MedOS) | Legacy architecture; AI is bolt-on, not native |

### 9.2 Point Solutions

| Competitor | Focus | Strength | Weakness |
|-----------|-------|----------|----------|
| **Abridge** | Ambient clinical documentation | Best-in-class ambient AI; deployed at major health systems | Documentation only; not a platform; no billing, scheduling, or quality |
| **Nabla** | Ambient AI documentation | Strong European presence; multi-language | Single use case; no US regulatory depth |
| **Regard** | AI-powered diagnosis suggestions | Clinical reasoning AI | Diagnosis only; no workflow automation |
| **Olive AI** | Revenue cycle automation | RPA-based automation at scale | Shut down core operations; pivoted; trust issues in market |

### 9.3 MedOS Differentiation

MedOS's competitive moat rests on three pillars:

**1. Agent-native architecture.** MedOS is designed from the ground up for AI agents. Agents are not features added to an existing EHR -- they are the core operating model. This means agent-optimized data models (FHIR JSONB with pgvector), agent-optimized APIs (MCP servers), and agent-optimized workflows (LangGraph state machines).

**2. Protocol-native interoperability.** MedOS is the first healthcare platform to natively support MCP + A2A. This enables a healthcare AI marketplace where third-party agents can discover MedOS capabilities (via A2A Agent Cards), connect to MedOS tools (via MCP servers), and operate within MedOS's compliance framework (via HMCP guardrails). No competitor offers this.

**3. Specialty focus with expansion path.** By targeting mid-size specialty practices (5-30 providers) in orthopedics and dermatology, MedOS avoids direct competition with Epic/Cerner (enterprise) and DrChrono (small/solo practice). The specialty-specific agent training (ICD-10 codes, workflow patterns, payer rules) creates switching costs that general-purpose platforms cannot match.

---

## 10. Implementation Strategy

### 10.1 Sprint Mapping

The agentic API layer integrates with the existing MedOS sprint plan from [[PHASE-1-EXECUTION-PLAN]]:

| Sprint | Weeks | Agentic API Focus | Dependencies |
|--------|-------|-------------------|--------------|
| **Sprint 1** | W3-4 | MCP Foundation + FHIR MCP Server + Clinical Scribe Agent (basic) | Sprint 0 complete: AWS, Auth, DB, FastAPI, FHIR Patient CRUD |
| **Sprint 2** | W5-6 | Revenue cycle agents (Prior Auth, Denial Mgmt) + Billing MCP + Scheduling MCP | Scribe agent functional; X12 integration started |
| **Sprint 3** | W7-8 | Analytics MCP + Quality Reporting Agent + Terminology MCP + A2A Agent Cards | Core agents operational; quality measures defined |
| **Sprint 4** | W9-10 | Production hardening: load testing, security pen test, HMCP compliance audit, marketplace prep | All agents in staging; security review complete |

### 10.2 Sprint 1 Detail: MCP Foundation

Week 3-4 deliverables:

1. **MCP Server Framework**
   - FastAPI-based MCP server runtime with JSON-RPC 2.0 handler
   - SSE transport for remote agent connections
   - HMCP-compliant auth layer (SMART on FHIR scopes)
   - Tenant isolation middleware (from [[ADR-002-multi-tenancy-schema-per-tenant]])

2. **FHIR MCP Server**
   - Tools: `patient.read`, `patient.search`, `encounter.create`, `encounter.read`, `observation.write`, `condition.read`
   - Resources: Patient summary, encounter history, problem list
   - PHI filter: Agent-type-based access profiles
   - Audit: FHIR AuditEvent for every tool invocation

3. **Clinical Scribe Agent (MVP)**
   - LangGraph StateGraph: audio_input -> transcription (Whisper) -> note_generation (Claude) -> coding_suggestion (Claude) -> confidence_check -> provider_review
   - Scribe MCP Server: `recording.start`, `recording.stop`, `note.generate`, `code.suggest`
   - Confidence threshold: >= 0.85 for auto-suggestion, < 0.85 for flagged review
   - Output: Draft SOAP note as FHIR DocumentReference with suggested Condition (ICD-10) and Procedure (CPT) resources

4. **Event Bus (Foundation)**
   - Redis Streams setup with consumer groups per agent type
   - `encounter.completed` event wired to Scribe Agent
   - Event schema with tenant isolation (tenant_id in stream key)

### 10.3 Sprint 2 Detail: Revenue Cycle Agents

Week 5-6 deliverables:

1. **Billing MCP Server**
   - Tools: `eligibility.check` (X12 270/271), `claim.submit` (X12 837P), `era.process` (X12 835), `pa.check_required`
   - Resources: Fee schedules, payer rules, denial reason codes
   - Integration: Availity clearinghouse API (see [[Revenue-Cycle-Deep-Dive]])

2. **Scheduling MCP Server**
   - Tools: `slot.search`, `appointment.create`, `appointment.cancel`, `waitlist.add`
   - Resources: Provider availability, room calendar, appointment types

3. **Prior Auth Agent**
   - LangGraph StateGraph: pa_check -> evidence_compile -> request_submit -> status_track -> outcome_process
   - Durable state: PostgreSQL checkpointer for multi-day PA workflows
   - Integration with Da Vinci PAS (FHIR-based PA) where payers support it, X12 278 otherwise (see [[Prior-Authorization-Deep-Dive]])

4. **Denial Management Agent**
   - LangGraph StateGraph: denial_receive -> reason_analyze -> appeal_draft -> review_queue -> appeal_submit -> outcome_track
   - Knowledge base: Historical denial patterns per payer, successful appeal templates
   - Human checkpoint: All appeal letters require staff review before submission

### 10.4 Sprint 3 Detail: Analytics and Intelligence

Week 7-8 deliverables:

1. **Analytics MCP Server**
   - Tools: `measure.calculate`, `population.query`, `trend.analyze`, `hcc.score`
   - Resources: Quality dashboards, population health summaries, financial analytics

2. **Terminology MCP Server**
   - Tools: `icd10.search`, `cpt.lookup`, `snomed.map`, `ndc.search`, `interaction.check`
   - Resources: Code sets, crosswalk tables, clinical guidelines

3. **Quality Reporting Agent**
   - HEDIS/MIPS measure calculation with care gap detection
   - HCC risk adjustment scoring for value-based contracts
   - Automated report generation for CMS submission

4. **A2A Agent Cards**
   - Publish Agent Cards for all 5 agents at `/.well-known/agent.json`
   - A2A discovery endpoint for marketplace preview
   - Task lifecycle management for cross-agent delegation

### 10.5 Sprint 4 Detail: Production Hardening

Week 9-10 deliverables:

1. **Security hardening**
   - Penetration testing of all MCP servers
   - Agent isolation verification (confirm no cross-tenant data leakage)
   - Credential injection audit (confirm no secrets in LLM context)
   - PHI filter testing (confirm minimum necessary enforcement)

2. **Load testing**
   - Concurrent agent execution under realistic practice load (30 providers, 200 encounters/day)
   - Redis Streams throughput and consumer group lag monitoring
   - LangGraph checkpointer performance under concurrent state writes

3. **HMCP compliance audit**
   - Verify all MCP servers implement HMCP specification
   - Audit trail completeness check (every tool call has AuditEvent)
   - SMART on FHIR scope enforcement validation

4. **Marketplace preparation**
   - Third-party developer documentation for MCP server and A2A integration
   - Sandbox environment for partner testing
   - Agent Card registry and discovery service

---

## 11. Risk Analysis

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **HIPAA violation from agent action** | Low | Critical | Multi-stage safety layer (Block/Warn/Review/Sanitize); PHI never in LLM context without authorization; FHIR AuditEvent for every action; BAA with Anthropic covers Claude API calls |
| **Agent hallucination causing clinical harm** | Medium | Critical | Confidence scoring on every AI output; < 0.85 threshold triggers human review; all clinical notes require provider signature; agents suggest, never prescribe |
| **Vendor lock-in on Claude/Anthropic** | Low | High | LangGraph's LLM-agnostic design allows node-level model swapping; MCP is an open standard; FHIR data model is vendor-neutral |
| **MCP/A2A protocol immaturity** | Medium | Medium | Both protocols have strong backing (Anthropic, Google, Linux Foundation); HMCP adds healthcare-specific guardrails; MedOS can contribute to spec evolution |
| **Low adoption by target practices** | Medium | High | Mid-size practices are underserved and motivated; ROI from reduced admin burden is quantifiable; specialty focus creates relevance; phased rollout starting with ambient scribe (proven demand, 72% adoption) |
| **NIST standards not finalized** | Medium | Low | MedOS aligns with NIST direction but does not depend on finalized standards; can adapt as standards mature; early alignment is a competitive advantage |
| **Two-person team capacity** | High | Medium | AI-augmented development (Claude Code + Cursor = 5-10x multiplier); modular architecture allows incremental delivery; phased sprint plan prioritizes high-value features first |
| **Clearinghouse integration complexity** | Medium | Medium | Start with Availity (largest US clearinghouse); use standardized X12 EDI formats; Billing MCP server abstracts clearinghouse-specific details |

---

## 12. References

### Internal Vault References

- [[ADR-001-fhir-native-data-model]] -- FHIR-native JSONB storage in PostgreSQL
- [[ADR-002-multi-tenancy-schema-per-tenant]] -- Schema-per-tenant with RLS + KMS
- [[ADR-003-ai-agent-framework]] -- LangGraph + Claude for AI agent orchestration
- [[ADR-004-fastapi-backend-architecture]] -- FastAPI backend structure and conventions
- [[System-Architecture-Overview]] -- Complete system architecture
- [[agent-architecture]] -- Detailed agent specifications and bounded autonomy model
- [[mcp-integration-plan]] -- MCP server design and integration strategy
- [[HIPAA-Deep-Dive]] -- HIPAA compliance requirements and Day 1 checklist
- [[FHIR-R4-Deep-Dive]] -- FHIR R4 resources, search, profiles
- [[X12-EDI-Deep-Dive]] -- X12 270, 837P, 835 with examples
- [[Revenue-Cycle-Deep-Dive]] -- Complete RCM lifecycle
- [[Prior-Authorization-Deep-Dive]] -- PA automation and Da Vinci PAS
- [[Ambient-AI-Documentation]] -- Ambient AI documentation pipeline
- [[claude-for-healthcare-hipaa]] -- Claude for Healthcare HIPAA analysis
- [[context-rotting-and-agent-memory]] -- Context rot detection and agent memory architecture
- [[PHASE-1-EXECUTION-PLAN]] -- 117 tasks across 7 sprints

### External References

- [Anthropic - Advancing Claude in Healthcare and Life Sciences (JPM26, January 2026)](https://www.anthropic.com/news/healthcare-life-sciences)
- [MCP Specification - Model Context Protocol](https://modelcontextprotocol.io/)
- [A2A Protocol Specification - Agent2Agent (Linux Foundation)](https://a2a-protocol.org/latest/specification/)
- [NIST AI Agent Standards Initiative (February 17, 2026)](https://www.nist.gov/news-events/news/2026/02/announcing-ai-agent-standards-initiative-interoperable-and-secure)
- [NIST Concept Paper: AI Agent Identity and Authorization](https://www.nccoe.nist.gov/sites/default/files/2026-02/accelerating-the-adoption-of-software-and-ai-agent-identity-and-authorization-concept-paper.pdf)
- [HMCP: Healthcare Model Context Protocol (Innovaccer)](https://innovaccer.com/blogs/introducing-hmcp-a-universal-open-standard-for-ai-in-healthcare)
- [HMCP Specification and SDK (GitHub)](https://github.com/innovaccer/Healthcare-MCP)
- [AWS HealthLake MCP Server (AWS Labs)](https://aws.amazon.com/blogs/industries/building-healthcare-ai-agents-with-open-source-aws-healthlake-mcp-server/)
- [WSO2 FHIR MCP Server (GitHub)](https://github.com/wso2/fhir-mcp-server)
- [Flexpa mcp-fhir (GitHub)](https://github.com/flexpa/mcp-fhir)
- [MCP-FHIR Framework for Clinical Decision Support (arXiv 2506.13800)](https://arxiv.org/html/2506.13800v1)
- [FHIR R5 Subscriptions Specification](https://hl7.org/fhir/R5/subscriptions.html)
- [FHIR R5 Subscriptions Backport Implementation Guide](https://hl7.org/fhir/uv/subscriptions-backport/2024Jan/)
- [OpenClaw - Open Source AI Agent](https://openclaw.ai/)
- [IronClaw - Secure Agent Runtime (NEAR AI)](https://github.com/nearai/ironclaw)
- [Claude Agent SDK - Python (Anthropic)](https://github.com/anthropics/claude-agent-sdk-python)
- [Claude Agent SDK Hooks Documentation](https://platform.claude.com/docs/en/agent-sdk/hooks)
- [LangGraph - Build Resilient Language Agents as Graphs](https://github.com/langchain-ai/langgraph)
- [LangGraph Interrupts for Human-in-the-Loop](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [athenahealth FHIR Event Notifications](https://www.athenahealth.com/resources/blog/fhir-subscriptions-event-driven-apis)
- [Google A2A Protocol Announcement (April 2025)](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- [A2A Protocol Upgrades (Google Cloud Blog)](https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade)

---

*This document synthesizes research conducted February 28, 2026 for the MedOS Healthcare OS project. It should be treated as a living reference and updated as protocols mature, competitors evolve, and implementation experience accumulates.*
