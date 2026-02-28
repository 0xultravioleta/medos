---
type: architecture
date: "2026-02-28"
status: active
tags:
  - architecture
  - ai
  - decision
  - phase-1
---

# MedOS AI Agent Architecture

> Complete specification of the agentic AI system within MedOS. Defines how AI agents operate, communicate, and serve providers, patients, and billing staff -- with bounded autonomy, mandatory human oversight, and full audit trails.

Related: [[HEALTHCARE_OS_MASTERPLAN]] | [[ADR-003-ai-agent-framework]] | [[System-Architecture-Overview]] | [[context-rotting-and-agent-memory]] | [[MOC-Agent-Architecture]]

---

## 1. Agent Framework

MedOS agents are **stateful, multi-step AI workflows** built on a layered framework stack. They are not chatbots. They are deterministic state machines with AI-powered processing nodes, operating under strict healthcare constraints.

### Technology Stack

| Layer | Technology | Role |
|-------|-----------|------|
| **Orchestration** | LangGraph | State machine graphs with checkpointing, interrupts, and replay |
| **LLM Backbone** | Claude (via AWS Bedrock, HIPAA BAA) | Reasoning, generation, classification, extraction |
| **Tool Integration** | MCP (Model Context Protocol) | Standardized interface between agents and external tools/data |
| **Healthcare Data** | FHIR-MCP Server | Read/write FHIR R4 resources through MCP |
| **State Persistence** | PostgreSQL (LangGraph checkpointer) | Agent state survives restarts; enables async human review |
| **Observability** | Langfuse + OpenTelemetry | Trace every LLM call, token usage, latency, confidence |

### Why This Stack

- **LangGraph** was chosen over CrewAI, AutoGen, and custom frameworks because healthcare needs explicit state machines with guaranteed human checkpoints, not autonomous agent swarms (see [[ADR-003-ai-agent-framework]] for full analysis).
- **Claude via AWS Bedrock** provides a HIPAA Business Associate Agreement covering PHI in API calls. No PHI leaves the BAA boundary.
- **MCP** standardizes how agents access tools, making it possible for third-party AI apps to integrate with MedOS through the same protocol (see [[mcp-integration-plan]]).

### Agent Execution Model

Every agent follows the same execution pattern, defined in [[ADR-003-ai-agent-framework]]:

```
Input Validation -> Context Retrieval -> AI Processing -> Confidence Check
    |                                                          |
    v                                                     [>= 0.85] -> Auto-execute -> Audit Log -> Output
  Error                                                   [0.70-0.85] -> Execute + Flag for Review -> Audit Log
                                                          [< 0.70] -> Halt + Escalate to Human -> Audit Log
                                                          [Critical action] -> ALWAYS escalate regardless of confidence
```

---

## 2. The 5 Core Agents

Each agent is a LangGraph `StateGraph` with defined authority, constraints, escalation rules, and audit requirements. All agents share the base pattern from [[ADR-003-ai-agent-framework]] but with domain-specific nodes and thresholds.

---

### 2.1 Clinical Documentation Agent (AI Scribe)

**Module:** B (Provider Workflow Engine)
**Pipeline:** Audio -> Transcript -> Clinical NLU -> SOAP Note -> ICD-10/CPT Codes

#### Purpose

Listens to provider-patient encounters via ambient audio capture and generates structured clinical documentation in SOAP format. This is the flagship agent -- the single capability most likely to drive pilot adoption (see [[Ambient-AI-Documentation]] for full domain analysis).

#### Authority (What It CAN Do Autonomously)

- Transcribe audio to text using Whisper v3
- Generate draft SOAP notes from transcripts
- Suggest ICD-10 diagnosis codes with confidence scores
- Suggest CPT procedure codes with confidence scores
- Pre-populate structured FHIR resources (Condition, Procedure, Observation) from notes
- Flag potential coding errors or missing documentation

#### Constraints (What It CANNOT Do -- Hard Limits)

- **CANNOT diagnose patients** -- it extracts diagnoses stated by the provider, it does not generate new diagnoses
- **CANNOT prescribe medications** -- medication decisions are provider-only
- **CANNOT modify signed/finalized clinical records** -- draft only until provider signs off
- **CANNOT access patient records beyond the current encounter** without explicit provider action
- **CANNOT operate without a licensed provider in the review loop**

#### Escalation Rules

| Condition | Action |
|-----------|--------|
| Confidence < 0.75 on any ICD-10 code | Flag code for manual review; highlight uncertainty in UI |
| Confidence < 0.70 on SOAP note section | Mark section as "needs review"; do not auto-populate FHIR |
| Audio quality below threshold (SNR < 10dB) | Notify provider; request re-recording or manual input |
| Detected contradiction in clinical data | Halt coding; present contradiction to provider |
| Patient mentions self-harm or abuse | Immediately flag to provider via priority notification |

#### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Transcription accuracy | >= 0.95 WER | 0.85-0.95 | < 0.85 |
| SOAP note generation | >= 0.90 | 0.75-0.90 | < 0.75 |
| ICD-10 code suggestion | >= 0.95 | 0.80-0.95 | < 0.80 |
| CPT code suggestion | >= 0.95 | 0.80-0.95 | < 0.80 |

#### Tools / MCP Servers

- **Whisper v3** (self-hosted GPU): Speech-to-text, returns timestamped transcript
- **Claude via Bedrock**: Clinical NLU, note generation, code suggestion
- **FHIR-MCP Server**: Read Patient, Encounter context; Write DocumentReference, Condition, Procedure
- **Terminology MCP Server** (future): ICD-10-CM, CPT, SNOMED CT lookups and validation

#### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, allergies, problem list |
| Read | `Encounter` | Current encounter context, chief complaint |
| Read | `Observation` | Vitals captured during encounter |
| Read | `MedicationStatement` | Current medications for context |
| Write | `DocumentReference` | Generated SOAP note (draft status) |
| Write | `Condition` | Extracted diagnoses (draft, pending provider review) |
| Write | `Procedure` | Extracted procedures (draft, pending provider review) |
| Write | `DiagnosticReport` | Structured findings (draft) |

#### Audit Trail

Every run of the Clinical Documentation Agent produces a FHIR `Provenance` resource containing:

```json
{
  "resourceType": "Provenance",
  "target": [{"reference": "DocumentReference/456"}],
  "recorded": "2026-02-28T14:30:00Z",
  "agent": [
    {
      "type": {"text": "AI Agent"},
      "who": {"display": "MedOS Clinical Documentation Agent v1.2.0"}
    },
    {
      "type": {"text": "Reviewer"},
      "who": {"reference": "Practitioner/dr-smith"}
    }
  ],
  "entity": [
    {
      "role": "source",
      "what": {"display": "Audio recording (encounter/789)"}
    }
  ],
  "extension": [
    {
      "url": "https://medos.health/fhir/ext/ai-metadata",
      "extension": [
        {"url": "model-version", "valueString": "claude-sonnet-4-20250514"},
        {"url": "confidence-score", "valueDecimal": 0.92},
        {"url": "prompt-hash", "valueString": "sha256:a1b2c3..."},
        {"url": "processing-time-ms", "valueInteger": 3400},
        {"url": "review-status", "valueCode": "approved"},
        {"url": "corrections-made", "valueInteger": 2}
      ]
    }
  ]
}
```

#### State Machine

```mermaid
stateDiagram-v2
    [*] --> CaptureAudio
    CaptureAudio --> Transcribe: Audio complete
    Transcribe --> ClinicalNLU: Transcript ready
    ClinicalNLU --> GenerateSOAP: Entities extracted
    GenerateSOAP --> SuggestCodes: SOAP draft ready
    SuggestCodes --> ConfidenceCheck: Codes generated

    ConfidenceCheck --> ProviderReview: Always (clinical = mandatory review)
    ProviderReview --> ApplyCorrections: Provider edits
    ProviderReview --> FinalizeNote: Provider approves
    ApplyCorrections --> FinalizeNote: Corrections saved

    FinalizeNote --> WriteFHIR: Signed off
    WriteFHIR --> AuditLog
    AuditLog --> [*]
```

---

### 2.2 Prior Authorization Agent

**Module:** C (Revenue Cycle) + D (Payer Integration)
**Pipeline:** PA Requirement Detection -> Clinical Evidence Gathering -> Form Generation -> Submission -> Status Tracking

#### Purpose

Automates the prior authorization workflow, which currently costs the U.S. healthcare system $35 billion/year and consumes 14.6 hours/week per practice (see [[Prior-Authorization-Deep-Dive]]). The agent detects when a PA is needed, gathers clinical evidence, generates the submission, and tracks status through approval.

#### Authority (What It CAN Do Autonomously)

- Detect PA requirements by checking payer-specific rules against ordered services
- Gather supporting clinical documentation from the FHIR store
- Generate PA request forms with clinical justification narrative
- Submit PA requests via X12 278 or payer API (after human approval)
- Track PA status and send notifications on updates
- Auto-resubmit with additional documentation when payer requests more information (pended status)

#### Constraints (What It CANNOT Do -- Hard Limits)

- **CANNOT fabricate clinical data** -- all clinical justification must trace to existing FHIR resources
- **CANNOT bypass medical necessity criteria** -- if clinical evidence is insufficient, it must escalate
- **CANNOT submit without human approval** -- PA submissions always require provider or staff sign-off
- **CANNOT modify clinical records** to make them support a PA request
- **CANNOT guarantee approval** -- it optimizes the submission, it does not control payer decisions

#### Escalation Rules

| Condition | Action |
|-----------|--------|
| Insufficient clinical evidence for medical necessity | Alert staff; suggest what documentation is needed |
| Payer requests peer-to-peer review | Notify provider; schedule P2P call; provide talking points |
| PA denied | Hand off to [[#2.3 Denial Management Agent]] |
| Urgent/emergent PA (patient safety) | Priority alert to provider; flag as urgent in submission |
| Unknown payer PA requirements | Flag for manual research; add to knowledge base when resolved |

#### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| PA requirement detection | >= 0.95 | 0.85-0.95 | < 0.85 |
| Clinical evidence sufficiency | >= 0.90 | 0.75-0.90 | < 0.75 |
| Form generation | N/A | N/A | ALWAYS review (0%) |
| Submission | N/A | N/A | ALWAYS review (0%) |

PA submissions **always require human confirmation** regardless of confidence score. This is a hard constraint -- financial and clinical liability demands it.

#### Tools / MCP Servers

- **Claude via Bedrock**: Clinical justification narrative generation, medical necessity analysis
- **FHIR-MCP Server**: Read Patient, Encounter, Condition, Procedure, Observation, MedicationRequest
- **Prior Auth MCP Server** (custom): Payer rule lookups, X12 278 generation, payer API submission
- **Scheduling MCP Server** (custom): P2P review scheduling

#### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, insurance info |
| Read | `Coverage` | Active insurance coverage, payer details |
| Read | `Condition` | Supporting diagnoses |
| Read | `Procedure` | Requested procedure requiring PA |
| Read | `Observation` | Lab results, vitals supporting medical necessity |
| Read | `MedicationRequest` | Medication requiring PA |
| Read | `DocumentReference` | Clinical notes supporting the request |
| Write | `ClaimResponse` (PA) | PA request and response tracking |
| Write | `Task` | PA workflow task status |

#### Audit Trail

Logs: PA requirement trigger, clinical evidence gathered (references only, no PHI duplication), justification narrative generated, submission timestamp, payer response, human reviewer identity, all status transitions.

---

### 2.3 Denial Management Agent

**Module:** C (Revenue Cycle)
**Pipeline:** Denial Receipt -> Root Cause Analysis -> Appeal Strategy -> Appeal Draft -> Resubmission

#### Purpose

Analyzes claim denials, identifies root causes, determines if an appeal is warranted, drafts appeal letters with supporting clinical evidence, and manages the resubmission workflow. Currently, 65% of denials are never appealed, yet 50-70% of appeals are successful -- this represents massive recoverable revenue (see [[Revenue-Cycle-Deep-Dive]]).

#### Authority (What It CAN Do Autonomously)

- Parse X12 835 remittance advice to extract denial reason codes (CARC/RARC)
- Classify denial type: clinical, technical, coding, eligibility, timeliness
- Calculate financial impact and prioritize appeals by dollar value and success probability
- Gather supporting clinical evidence from FHIR store
- Draft appeal letters with payer-specific formatting and clinical arguments
- Suggest corrected codes when denial is coding-related
- Track appeal deadlines and send deadline warnings

#### Constraints (What It CANNOT Do -- Hard Limits)

- **CANNOT alter clinical documentation** to support an appeal
- **CANNOT create false claims** or misrepresent clinical facts
- **CANNOT submit appeals without human approval** -- all appeals require staff or provider sign-off
- **CANNOT fabricate clinical evidence** that does not exist in the FHIR store
- **CANNOT override a legitimate denial** -- if the denial is correct, it must report that to staff

#### Escalation Rules

| Condition | Action |
|-----------|--------|
| Denial appears clinically legitimate | Report to staff: "This denial may be correct. Recommend accepting." |
| High-value denial (> $5,000) | Priority alert to billing manager; recommend provider review |
| Pattern detected (same denial reason across multiple claims) | Generate pattern report; flag systemic issue |
| Appeal deadline < 48 hours | Urgent alert to billing staff |
| Payer requires additional clinical records | Route to provider for clinical input |

#### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Denial reason classification | >= 0.95 | 0.85-0.95 | < 0.85 |
| Appeal viability assessment | >= 0.90 | 0.75-0.90 | < 0.75 |
| Appeal letter draft | N/A | N/A | ALWAYS review (0%) |
| Appeal submission | N/A | N/A | ALWAYS review (0%) |

Like the Prior Auth Agent, appeal submissions **always require human confirmation**.

#### Tools / MCP Servers

- **Claude via Bedrock**: Denial analysis, appeal strategy, letter generation
- **FHIR-MCP Server**: Read Claim, ClaimResponse, ExplanationOfBenefit, Patient, Encounter, Condition
- **Billing/Claims MCP Server** (custom): X12 835 parsing, CARC/RARC code lookups, appeal submission
- **Analytics MCP Server** (custom): Historical denial patterns, success rates by payer/code/reason

#### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Claim` | Original claim details |
| Read | `ClaimResponse` | Payer adjudication result |
| Read | `ExplanationOfBenefit` | Full remittance details |
| Read | `Patient` | Demographics for appeal |
| Read | `Encounter` | Encounter supporting the claim |
| Read | `Condition`, `Procedure` | Clinical evidence for appeal |
| Read | `DocumentReference` | Clinical notes supporting appeal |
| Write | `Claim` | Corrected/resubmitted claim |
| Write | `Communication` | Appeal letter and correspondence |
| Write | `Task` | Appeal workflow tracking |

#### Audit Trail

Logs: Denial reason codes, financial impact, appeal viability score, evidence gathered, appeal strategy selected, letter generated (hash), human reviewer, submission timestamp, appeal outcome.

---

### 2.4 Patient Communication Agent

**Module:** F (Patient Engagement)
**Pipeline:** Event Trigger -> Message Generation -> Channel Selection -> Delivery -> Response Handling

#### Purpose

Manages proactive and reactive patient communications across multiple channels (SMS, email, web portal, voice). Handles appointment reminders, pre-visit instructions, billing inquiries, FAQ responses, and basic triage routing. This agent is patient-facing and operates under strict constraints about what it can and cannot say.

#### Authority (What It CAN Do Autonomously)

- Send appointment reminders and confirmations
- Answer pre-defined FAQ questions (office hours, location, accepted insurance, etc.)
- Provide billing balance information and payment links
- Collect pre-visit intake information (demographics, insurance, symptoms checklist)
- Route urgent messages to appropriate staff
- Send post-visit follow-up instructions (pre-approved by provider)
- Respond to rescheduling requests by offering available slots

#### Constraints (What It CANNOT Do -- Hard Limits)

- **CANNOT diagnose or provide medical advice** -- not a clinical tool
- **CANNOT triage emergencies** -- any mention of chest pain, difficulty breathing, suicidal ideation, or similar emergencies must immediately display "Call 911" and alert staff
- **CANNOT access PHI beyond what is needed** for the specific communication (minimum necessary principle)
- **CANNOT share clinical results** (lab, imaging) -- patients must access these through the patient portal or provider
- **CANNOT make clinical recommendations** -- scheduling, billing, and logistics only
- **CANNOT impersonate a human** -- must clearly identify as an AI assistant

#### Escalation Rules

| Condition | Action |
|-----------|--------|
| Patient describes emergency symptoms | Display emergency instructions; alert provider immediately |
| Patient asks clinical questions | Route to provider messaging; explain "I can help with scheduling and billing, but clinical questions need your care team" |
| Patient expresses frustration or anger | Route to human staff member; do not attempt to de-escalate autonomously |
| Patient requests medical records | Direct to patient portal; do not transmit records via SMS/email |
| Message content unclear | Ask one clarifying question; if still unclear, route to staff |
| Patient mentions billing dispute | Route to billing department; provide reference number |

#### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Appointment reminders | >= 0.99 (template-based) | N/A | N/A |
| FAQ responses | >= 0.85 | 0.70-0.85 | < 0.70 |
| Billing inquiries | >= 0.90 | 0.75-0.90 | < 0.75 |
| Triage routing | N/A | N/A | ALWAYS route to human |

#### Tools / MCP Servers

- **Claude via Bedrock**: Natural language understanding, response generation
- **FHIR-MCP Server**: Read Patient (demographics), Appointment (scheduling)
- **Scheduling MCP Server** (custom): Available slots, appointment booking/rescheduling
- **SMS/Email API**: Twilio (SMS), SES (email) for message delivery
- **Billing/Claims MCP Server** (custom): Balance lookup, payment link generation

#### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Name, contact info, preferred language |
| Read | `Appointment` | Upcoming appointments, scheduling context |
| Read | `Schedule`, `Slot` | Available appointment slots |
| Read | `Account` | Patient balance information |
| Read | `Communication` | Prior message history with this patient |
| Write | `Appointment` | Booked/rescheduled appointments |
| Write | `Communication` | Outbound messages (audit trail) |
| Write | `QuestionnaireResponse` | Pre-visit intake responses |

#### Audit Trail

Logs: Message type, channel (SMS/email/portal), content hash, delivery status, patient response, routing decisions, escalation triggers. All outbound messages stored as FHIR `Communication` resources.

---

### 2.5 Quality Reporting Agent

**Module:** E (Population Health & Analytics)
**Pipeline:** Measure Definition -> Patient Population -> Gap Identification -> Report Generation -> Submission

#### Purpose

Identifies care gaps across patient populations, calculates quality measures (HEDIS, MIPS, CMS Star Ratings), predicts readmission risk, and generates compliance reports. This agent operates on population-level data, not individual encounters.

#### Authority (What It CAN Do Autonomously)

- Calculate quality measure numerators and denominators from FHIR data
- Identify patients with care gaps (overdue screenings, missed follow-ups)
- Generate HEDIS, MIPS, and CMS Star Rating reports
- Calculate HCC risk scores (CMS-HCC v28)
- Predict 30-day readmission risk using LACE+ with SDOH factors
- Generate provider performance benchmarks
- Flag patients who are falling out of care protocols

#### Constraints (What It CANNOT Do -- Hard Limits)

- **CANNOT modify patient records or clinical data** -- read-only access to clinical FHIR resources
- **CANNOT directly contact patients** -- care gap notifications go through the Patient Communication Agent or provider workflows
- **CANNOT alter quality measure definitions** -- measures are defined by CMS/NCQA, not by the agent
- **CANNOT submit quality reports to CMS/payers without human approval**
- **CANNOT make individual clinical recommendations** -- it identifies gaps, clinicians decide actions

#### Escalation Rules

| Condition | Action |
|-----------|--------|
| Critical care gap (e.g., cancer screening 2+ years overdue) | Priority alert to care coordinator |
| High readmission risk patient (LACE+ > 12) | Alert care team; suggest intervention |
| Quality measure below reporting threshold | Alert practice manager with improvement recommendations |
| Data quality issues (missing required fields) | Flag to data team; do not estimate missing values |
| Measure logic conflict with local clinical protocols | Flag to clinical leadership for resolution |

#### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Measure calculation | >= 0.99 (deterministic) | N/A | Data quality issues |
| Care gap identification | >= 0.95 | 0.85-0.95 | < 0.85 |
| Readmission prediction | >= 0.85 | 0.70-0.85 | < 0.70 |
| Report generation | >= 0.95 | N/A | ALWAYS review before external submission |

#### Tools / MCP Servers

- **Claude via Bedrock**: Natural language report generation, gap analysis narrative
- **FHIR-MCP Server**: Read Patient, Condition, Observation, Procedure, MeasureReport, Encounter
- **Analytics MCP Server** (custom): Measure definitions, benchmarking data, population queries
- **Scheduling MCP Server** (custom): Identify patients who need follow-up appointments

#### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, insurance, attribution |
| Read | `Condition` | Chronic conditions, problem list |
| Read | `Observation` | Lab results, vitals, screening results |
| Read | `Procedure` | Completed procedures (screenings, etc.) |
| Read | `Encounter` | Visit history, admission/discharge |
| Read | `MedicationStatement` | Current medications |
| Read | `Immunization` | Vaccination history |
| Write | `MeasureReport` | Calculated quality measures |
| Write | `DetectedIssue` | Identified care gaps |
| Write | `RiskAssessment` | Readmission risk scores |

#### Audit Trail

Logs: Measure calculated, population size, numerator/denominator, data sources queried, calculation timestamp, report version, human reviewer for external submissions.

---

## 3. Agent Communication Protocol

Agents do not operate in isolation. Clinical workflows often span multiple agents (e.g., documentation -> coding -> claim -> denial -> appeal). The communication protocol defines how agents coordinate.

### Event Bus Architecture

All inter-agent communication flows through the event bus (Redis Streams, with path to Kafka/EventBridge at scale). Agents never call each other directly.

```mermaid
graph TB
    subgraph "Agent Layer"
        DOC[Clinical Documentation Agent]
        PA[Prior Auth Agent]
        DEN[Denial Management Agent]
        PAT[Patient Communication Agent]
        QUA[Quality Reporting Agent]
    end

    subgraph "Event Bus (Redis Streams)"
        E1[encounter.documented]
        E2[coding.approved]
        E3[pa.required]
        E4[claim.denied]
        E5[care.gap.detected]
        E6[appointment.reminder.due]
    end

    subgraph "Human Interface"
        PROV[Provider Workspace]
        BILL[Billing Dashboard]
        PTNT[Patient Portal]
    end

    DOC -->|publishes| E1
    E1 -->|triggers| QUA
    E2 -->|triggers| PA
    PA -->|publishes| E3
    E3 -->|notifies| PROV
    E4 -->|triggers| DEN
    E5 -->|triggers| PAT
    E6 -->|triggers| PAT

    DOC -.->|review queue| PROV
    PA -.->|approval queue| BILL
    DEN -.->|appeal queue| BILL
    PAT -.->|messages| PTNT
    QUA -.->|reports| PROV
```

### Event Message Format

```python
class AgentEvent(BaseModel):
    """Standardized event format for inter-agent communication."""
    event_id: str                    # UUID v4
    event_type: str                  # e.g., "encounter.documented"
    source_agent: str                # e.g., "clinical_documentation_agent"
    tenant_id: str                   # Tenant isolation
    timestamp: datetime              # UTC
    correlation_id: str              # Trace entire workflow chain
    payload: dict                    # Event-specific data (FHIR references, not PHI copies)
    priority: Literal["low", "normal", "high", "urgent"]
    requires_ack: bool               # Whether receiving agent must acknowledge
```

### Key Event Flows

| Trigger Event | Source Agent | Target Agent(s) | Action |
|--------------|-------------|-----------------|--------|
| `encounter.documented` | Clinical Documentation | Quality Reporting | Update care gap status, measure denominators |
| `coding.approved` | Clinical Documentation | Prior Auth (if PA required) | Initiate PA workflow |
| `claim.submitted` | (Revenue Cycle module) | Quality Reporting | Track revenue metrics |
| `claim.denied` | (Revenue Cycle module) | Denial Management | Analyze denial, draft appeal |
| `pa.approved` | Prior Auth | Patient Communication | Notify patient that procedure is approved |
| `pa.denied` | Prior Auth | Denial Management | Escalate PA denial for appeal |
| `care.gap.detected` | Quality Reporting | Patient Communication | Send care gap reminder to patient |
| `appeal.outcome` | Denial Management | Quality Reporting | Update financial analytics |

### Agent-to-Human Communication

Agents communicate with humans through **notification queues** with priority levels:

| Priority | SLA | Channel | Example |
|----------|-----|---------|---------|
| **Urgent** | < 5 min | Push notification + SMS + in-app alert | Patient emergency detected; PA deadline today |
| **High** | < 1 hour | Push notification + in-app alert | Appeal draft ready for review; high-value denial |
| **Normal** | < 4 hours | In-app notification | Coding suggestions ready; care gap report updated |
| **Low** | Next login | In-app badge | Quality metrics updated; billing summary available |

### Human-to-Agent Communication

Humans interact with agents through two mechanisms:

1. **Approval Workflows**: Structured UI where humans approve, reject, or correct agent outputs. Every agent with a human-in-the-loop checkpoint presents a review interface specific to its domain (e.g., coding review shows suggested codes with confidence scores and one-click approve/reject).

2. **Natural Language Chat**: For ad-hoc queries ("Why did you suggest this code?" or "What's the denial rate for this payer?"), humans can ask agents questions through a chat interface. The agent has access to its own audit trail and can explain its reasoning.

### Audit of Inter-Agent Communication

All inter-agent events are logged as FHIR `AuditEvent` resources:

```json
{
  "resourceType": "AuditEvent",
  "type": {"code": "agent-communication", "system": "https://medos.health/fhir/audit"},
  "subtype": [{"code": "encounter.documented"}],
  "recorded": "2026-02-28T14:35:00Z",
  "agent": [
    {
      "who": {"display": "Clinical Documentation Agent v1.2.0"},
      "requestor": false
    },
    {
      "who": {"display": "Quality Reporting Agent v1.0.0"},
      "requestor": false
    }
  ],
  "source": {"observer": {"display": "MedOS Event Bus"}},
  "entity": [
    {
      "what": {"reference": "DocumentReference/456"},
      "role": {"code": "4", "display": "Domain Resource"}
    }
  ]
}
```

---

## 4. Bounded Autonomy Framework

The Bounded Autonomy Framework is the governance layer that controls what agents can do. It is the most critical safety mechanism in MedOS and is enforced at the framework level, not the application level.

### Core Principle

> Every agent action has a confidence score. The score determines what happens next. Critical actions always require human confirmation, regardless of confidence.

### Confidence Score Model

Every AI-generated output includes a confidence score between 0.0 and 1.0, calculated from:

- **Model confidence**: Token-level probability from Claude
- **Evidence strength**: How much supporting data exists in the FHIR store
- **Historical accuracy**: How often similar suggestions have been accepted vs corrected
- **Payer-specific patterns**: Historical approval/denial rates for this payer + code combination

```python
class ConfidenceScore(BaseModel):
    """Composite confidence score for agent outputs."""
    overall: float                      # Weighted composite (0.0 - 1.0)
    model_confidence: float             # LLM token probability
    evidence_strength: float            # Supporting data availability
    historical_accuracy: float          # Past acceptance rate for similar outputs
    payer_pattern: float | None         # Payer-specific (billing agents only)

    weights: dict = {
        "model_confidence": 0.40,
        "evidence_strength": 0.30,
        "historical_accuracy": 0.20,
        "payer_pattern": 0.10,
    }
```

### Action Classification

| Category | Confidence Routing | Examples |
|----------|-------------------|----------|
| **Informational** | Standard thresholds | Care gap reports, analytics, measure calculations |
| **Administrative** | Standard thresholds | Appointment reminders, FAQ responses, status updates |
| **Financial** | Always human review | Claim submissions, PA submissions, appeal submissions |
| **Clinical** | Always human review | Code suggestions, note finalization, medication-related |

### Configurable Per Tenant

Confidence thresholds are configurable per tenant via the admin console. A large practice with experienced billing staff might set lower thresholds (more automation). A new practice might set higher thresholds (more review).

```python
class TenantAgentConfig(BaseModel):
    """Per-tenant agent configuration."""
    tenant_id: str
    agent_type: str
    auto_execute_threshold: float      # Default: 0.85
    flag_for_review_threshold: float   # Default: 0.70
    always_review_actions: list[str]   # Actions that bypass confidence (always review)
    enabled: bool                      # Can disable specific agents per tenant
    max_daily_auto_executions: int     # Safety cap on autonomous actions
```

### Safety Caps

Even with high confidence, agents have daily limits on autonomous actions:

| Agent | Max Daily Auto-Executions | Rationale |
|-------|--------------------------|-----------|
| Clinical Documentation | Unlimited (always reviewed) | Provider reviews every note |
| Prior Auth | 0 (always reviewed) | Financial and clinical liability |
| Denial Management | 0 (always reviewed) | Financial liability |
| Patient Communication | 500 per tenant | Prevent spam; rate limit outbound messages |
| Quality Reporting | 50 reports | Prevent excessive compute on large populations |

---

## 5. Memory Architecture

Agents need context to operate effectively, but context degrades over time (see [[context-rotting-and-agent-memory]]). MedOS implements a tiered memory architecture to balance speed, relevance, and accuracy.

### Three-Tier Memory Model

```
                    Speed       Capacity    Persistence    Source of Truth
                    -----       --------    -----------    ---------------
Short-term (Redis)  < 1ms       Small       Session TTL    No (cache)
Mid-term (pgvector) < 10ms      Medium      Persistent     No (index)
Long-term (FHIR)    < 100ms     Large       Permanent      YES (golden source)
```

### Short-Term Memory: Redis

- **What**: Current encounter context, active conversation state, session variables
- **TTL**: 2 hours (configurable per agent type)
- **Use case**: The Clinical Documentation Agent holds the current transcript, extracted entities, and draft note in Redis during an active encounter
- **Eviction**: LRU with TTL expiry

### Mid-Term Memory: pgvector

- **What**: Embedded patient history summaries, recent lab result summaries, payer rule embeddings
- **Persistence**: Permanent, but re-indexed periodically
- **Use case**: The Denial Management Agent retrieves similar past denials and their outcomes via vector similarity search to inform appeal strategy
- **Model**: `text-embedding-3-small` (OpenAI) or equivalent via Bedrock

### Long-Term Memory: PostgreSQL FHIR Store

- **What**: Complete clinical records, claims history, audit trails -- stored as FHIR JSONB resources
- **Persistence**: Permanent (6+ year retention for HIPAA)
- **Source of truth**: This is the golden source. All other memory tiers are derived from it.
- **Use case**: Every agent query ultimately validates against the FHIR store. An agent never "remembers" something that contradicts the FHIR store.

### Context Rot Detection

As described in [[context-rotting-and-agent-memory]], agent context degrades during long sessions:

```python
class ContextHealthMonitor:
    """Monitors agent context quality during sessions."""

    REFRESH_THRESHOLD = 0.75  # Cosine similarity threshold

    async def check_context_health(self, agent_state: AgentState) -> float:
        """Compare current context against golden source."""
        current_embedding = await self.embed(agent_state.context_summary)
        fresh_embedding = await self.embed(
            await self.fetch_fresh_context(agent_state.patient_id)
        )
        similarity = cosine_similarity(current_embedding, fresh_embedding)

        if similarity < self.REFRESH_THRESHOLD:
            await self.refresh_context(agent_state)

        return similarity
```

### Golden Source Principle

> The FHIR store is ALWAYS the source of truth. Everything in Redis and pgvector is cache. If an agent's memory contradicts the FHIR store, the FHIR store wins.

Refresh process when context rot is detected:
1. System detects cosine similarity drop below 0.75
2. Queries fresh data from the FHIR store
3. Re-embeds into pgvector
4. Prunes stale context from Redis
5. Agent continues with refreshed context
6. If context is unsalvageable: hand off to human with a clean context ("safe mode")

---

## 6. Observability & Monitoring

### LLM Observability (Langfuse)

Every LLM call is traced through Langfuse with:
- Input/output token counts
- Latency (TTFB and total)
- Model version
- Confidence score
- Cost per call
- Prompt template version (hash)

### Agent-Level Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `agent.confidence.mean` | Average confidence score per agent type | < 0.80 (investigate model drift) |
| `agent.escalation.rate` | % of actions escalated to humans | > 40% (model may be underperforming) |
| `agent.correction.rate` | % of auto-approved actions later corrected | > 5% (lower auto-approve threshold) |
| `agent.latency.p95` | 95th percentile agent execution time | > 30s (optimize pipeline) |
| `agent.error.rate` | % of agent runs that fail | > 2% (investigate errors) |

### Drift Detection

Per [[ml-drift-monitoring]], agents are monitored for:
- **Data drift**: Input distribution changes (KS test, PSI)
- **Concept drift**: Model accuracy decay over time
- **Output drift**: Hallucination rate, confidence score trends

Drift detection triggers retraining pipeline in Phase 3+ (see [[ml-drift-monitoring]] for full protocol).

---

## 7. Security & Compliance

### HIPAA Controls for Agents

All agent operations comply with HIPAA requirements as detailed in [[HIPAA-Deep-Dive]]:

| Control | Implementation |
|---------|---------------|
| **Minimum Necessary** | Agents only receive FHIR resources needed for their specific task |
| **Audit Trail** | Every agent action logged as FHIR Provenance/AuditEvent |
| **Access Control** | Agents inherit the permissions of the invoking user (RBAC + ABAC) |
| **Encryption** | All LLM calls over TLS 1.3; data at rest encrypted per-tenant KMS |
| **BAA Coverage** | Claude via AWS Bedrock (HIPAA BAA signed) |
| **Data Retention** | Agent audit logs retained 6+ years |
| **Breach Protocol** | Agent anomaly detection triggers security review |

### Agent Identity & Authentication

Each agent instance runs with a service identity, not a user identity. The agent inherits the requesting user's permissions but logs actions under both identities:

```python
class AgentIdentity(BaseModel):
    agent_type: str          # "clinical_documentation_agent"
    agent_version: str       # "1.2.0"
    instance_id: str         # UUID for this specific run
    invoking_user_id: str    # The human who triggered this agent
    tenant_id: str           # Tenant context
    scopes: list[str]        # FHIR scopes granted to this agent
```

### PHI Handling Rules for Agents

1. **Never log PHI** in agent traces, metrics, or error messages
2. **Never include PHI** in LLM metadata or Langfuse traces -- use resource references only
3. **Never cache PHI** in Redis beyond session TTL
4. **Never transmit PHI** in inter-agent events -- pass FHIR references, not data copies
5. **Always apply minimum necessary** -- strip fields the agent does not need

---

## 8. Implementation Roadmap

### Phase 1 (Months 0-6): Foundation

- [ ] Base agent framework (LangGraph + PostgreSQL checkpointer)
- [ ] HIPAA compliance layer (minimum necessary, audit logging)
- [ ] Clinical Documentation Agent (MVP: audio -> SOAP note)
- [ ] Confidence routing engine with configurable thresholds
- [ ] Provider review UI for documentation agent
- [ ] FHIR-MCP Server integration

### Phase 2 (Months 6-12): Revenue Agents

- [ ] Prior Authorization Agent
- [ ] Denial Management Agent
- [ ] Patient Communication Agent (appointment reminders, basic FAQ)
- [ ] Inter-agent event bus (Redis Streams)
- [ ] Billing staff review UI

### Phase 3 (Months 12-18): Intelligence

- [ ] Quality Reporting Agent
- [ ] ML drift monitoring and retraining pipeline
- [ ] Cross-agent workflow orchestration
- [ ] Custom MCP servers (Scheduling, Billing, Analytics)
- [ ] Agent analytics dashboard

### Phase 4 (Months 18-24): Platform

- [ ] Third-party agent marketplace (MCP-based)
- [ ] Tenant-customizable agent workflows
- [ ] Multi-model routing (specialized models per task)
- [ ] Advanced memory architecture (context rot detection)

---

## References

- [[HEALTHCARE_OS_MASTERPLAN]] -- Full vision and module definitions
- [[ADR-003-ai-agent-framework]] -- Framework selection decision
- [[ADR-001-fhir-native-data-model]] -- FHIR data model consumed by agents
- [[ADR-004-fastapi-backend-architecture]] -- Backend serving agent endpoints
- [[System-Architecture-Overview]] -- How agents fit in the overall system
- [[context-rotting-and-agent-memory]] -- Memory architecture research
- [[ml-drift-monitoring]] -- Drift detection for agent models
- [[Ambient-AI-Documentation]] -- Clinical documentation domain knowledge
- [[Prior-Authorization-Deep-Dive]] -- Prior auth domain knowledge
- [[Revenue-Cycle-Deep-Dive]] -- Revenue cycle domain knowledge
- [[HIPAA-Deep-Dive]] -- HIPAA compliance requirements
- [[mcp-integration-plan]] -- MCP integration strategy
- [[MOC-Agent-Architecture]] -- Navigation index for agent docs
