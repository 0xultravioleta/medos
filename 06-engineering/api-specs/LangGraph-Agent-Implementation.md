---
type: domain-knowledge
date: "2026-02-27"
tags:
  - engineering
  - ai
  - agents
  - langgraph
  - claude
  - module-b
  - module-c
category: engineering
confidence: high
sources: []
---

# LangGraph Agent Implementation Guide

This document defines the architecture, implementation patterns, and production deployment strategy for all AI agents in the MedOS platform. Each agent is built as a LangGraph `StateGraph` with typed state, conditional routing, human-in-the-loop interrupts, PostgreSQL-backed checkpointing, and a HIPAA compliance wrapper around every LLM call.

Cross-references: [[ADR-003-ai-agent-framework]], [[System-Architecture-Overview]], [[HIPAA-Deep-Dive]], [[Revenue-Cycle-Deep-Dive]], [[Prior-Authorization-Deep-Dive]], [[Ambient-AI-Documentation]]

---

## 1. Why LangGraph for Healthcare AI Agents

### The State Machine Argument

Healthcare workflows are not open-ended chat. They are **regulated, auditable processes** with deterministic paths punctuated by AI decision points. A prior authorization follows a strict sequence: check if PA is required, gather clinical docs, generate a medical justification, submit to a payer, and monitor the response. At each step the system must record who decided what and why.

LangGraph models this directly. Each workflow is a `StateGraph` where nodes are processing functions and edges encode the routing logic. Unlike raw API calls (where the developer must manually manage state, retries, and branching), LangGraph provides:

- **Typed state** (`TypedDict`) that flows through the entire graph, ensuring every node reads and writes to a well-defined schema.
- **Conditional edges** that route execution based on confidence scores, denial types, or payer-specific rules.
- **Built-in persistence** via checkpointers, so a long-running prior auth process (which may take days) survives service restarts.
- **Human-in-the-loop as a first-class primitive** using the `interrupt()` function and `Command(resume=...)` pattern, critical for clinical review.
- **Observability** through LangSmith and Langfuse integration for tracing every LLM call, including cost, latency, and token usage.

### Framework Comparison

| Criteria | Raw API Calls | CrewAI | AutoGen | LangGraph |
|---|---|---|---|---|
| State management | Manual | Implicit | Message-based | Typed StateGraph |
| Persistence | DIY | None built-in | None built-in | PostgreSQL checkpointer |
| Human-in-the-loop | DIY | Limited | Limited | `interrupt()` + `Command` |
| Deterministic routing | DIY | Agent decides | Agent negotiation | Conditional edges |
| Auditability | Manual logging | Minimal | Conversation logs | Full state snapshots |
| Healthcare compliance | No support | No support | No support | Encryption, typed state |
| Production deployment | DIY | Early stage | Research-oriented | LangGraph Platform / self-hosted |

CrewAI and AutoGen are designed for multi-agent conversation, where agents negotiate and collaborate. Healthcare does not need negotiation--it needs **deterministic, auditable pipelines** with AI at specific decision points. LangGraph is the right abstraction.

---

## 2. Core LangGraph Concepts for Our Use Case

### StateGraph and Typed State

Every agent defines a `TypedDict` state schema. State flows through the graph and each node returns a partial update that is merged into the current state.

```python
from typing import Annotated, Optional
from typing_extensions import TypedDict
from operator import add

class ClinicalDocState(TypedDict):
    """State for the Clinical Documentation Agent."""
    audio_url: str
    transcript: Optional[str]
    clinical_entities: Annotated[list[dict], add]
    soap_note: Optional[str]
    icd_codes: list[dict]
    cpt_codes: list[dict]
    confidence: float
    status: str
    review_notes: Optional[str]
    error: Optional[str]
```

The `Annotated[list[dict], add]` pattern uses a **reducer**: when a node returns `{"clinical_entities": [new_entity]}`, it is appended to the existing list rather than replacing it.

### Nodes

Nodes are plain Python functions that accept state and return a partial state update. They can call LLMs, databases, external APIs, or run pure logic.

```python
def transcribe(state: ClinicalDocState) -> dict:
    """Transcribe audio to text using Whisper or Deepgram."""
    transcript = transcription_service.transcribe(state["audio_url"])
    return {"transcript": transcript, "status": "transcribed"}
```

### Edges and Conditional Routing

Static edges define fixed transitions. Conditional edges use a routing function:

```python
from typing import Literal

def route_by_confidence(state: ClinicalDocState) -> Literal["auto_approve", "human_review", "escalate"]:
    """Route based on confidence score."""
    if state["confidence"] >= 0.95:
        return "auto_approve"
    elif state["confidence"] >= 0.80:
        return "human_review"
    else:
        return "escalate"

builder.add_conditional_edges(
    "suggest_codes",
    route_by_confidence,
    {
        "auto_approve": "finalize",
        "human_review": "human_review",
        "escalate": "escalate_to_coder",
    }
)
```

### Checkpointers (PostgreSQL for Production)

Every graph is compiled with a checkpointer. In production we use `PostgresSaver` with AES encryption:

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer

serde = EncryptedSerializer.from_pycryptodome_aes()  # reads LANGGRAPH_AES_KEY env var
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://medos:****@db:5432/langgraph_checkpoints",
    serde=serde
)
checkpointer.setup()

graph = builder.compile(checkpointer=checkpointer)
```

Each invocation requires a `thread_id` that acts as a persistent cursor:

```python
config = {"configurable": {"thread_id": f"pa-{patient_id}-{order_id}"}}
result = graph.invoke(initial_state, config=config)
```

### Human-in-the-Loop with `interrupt()` and `Command`

The `interrupt()` function pauses execution mid-node and surfaces a payload to the caller. The caller resumes by invoking the graph with `Command(resume=value)`:

```python
from langgraph.types import Command, interrupt

def human_review(state: ClinicalDocState) -> Command[Literal["finalize", "revise"]]:
    """Pause for clinical review."""
    decision = interrupt({
        "type": "clinical_review",
        "soap_note": state["soap_note"],
        "icd_codes": state["icd_codes"],
        "cpt_codes": state["cpt_codes"],
        "confidence": state["confidence"],
    })

    if decision["approved"]:
        return Command(
            goto="finalize",
            update={"review_notes": decision.get("notes", ""), "status": "approved"}
        )
    else:
        return Command(
            goto="revise",
            update={"review_notes": decision["notes"], "status": "revision_requested"}
        )
```

The API layer handles the interrupt:

```python
# Initial invocation -- pauses at human_review
result = graph.invoke(initial_state, config=config)
# result["__interrupt__"] contains the review payload

# After clinician reviews in the UI
from langgraph.types import Command
resumed = graph.invoke(
    Command(resume={"approved": True, "notes": "Looks good"}),
    config=config
)
```

---

## 3. Agent 1: Clinical Documentation Agent

This is the most complex agent. It takes an audio recording from a clinical encounter and produces a finalized SOAP note with ICD-10 and CPT codes, ready for billing. See [[Ambient-AI-Documentation]] for the clinical context.

### State Definition

```python
from typing import Annotated, Optional, Literal
from typing_extensions import TypedDict
from operator import add

class ClinicalDocState(TypedDict):
    # Input
    audio_url: str
    encounter_id: str
    provider_id: str
    patient_id: str

    # Processing
    transcript: Optional[str]
    clinical_entities: Annotated[list[dict], add]
    soap_note: Optional[str]
    icd_codes: list[dict]       # [{"code": "J06.9", "description": "...", "confidence": 0.92}]
    cpt_codes: list[dict]       # [{"code": "99213", "description": "...", "confidence": 0.88}]
    confidence: float           # Overall confidence score
    status: str                 # transcribed | entities_extracted | note_generated | codes_suggested | reviewed | finalized

    # Review
    review_notes: Optional[str]
    reviewer_id: Optional[str]

    # Error handling
    error: Optional[str]
    retry_count: int
```

### Graph Definition

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.types import Command, interrupt
from typing import Literal
import anthropic

client = anthropic.Anthropic()

# --- Node implementations ---

def transcribe(state: ClinicalDocState) -> dict:
    """Transcribe clinical audio using Deepgram with medical vocabulary."""
    from app.services.transcription import DeepgramMedicalTranscriber

    transcriber = DeepgramMedicalTranscriber()
    result = transcriber.transcribe(
        audio_url=state["audio_url"],
        model="nova-2-medical",
        language="en",
        diarize=True,
    )
    return {
        "transcript": result.transcript,
        "status": "transcribed",
    }


def extract_entities(state: ClinicalDocState) -> dict:
    """Extract clinical entities (diagnoses, medications, procedures) from transcript."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system="""You are a clinical NLP system. Extract structured clinical entities from
the following medical encounter transcript. Return JSON with the following schema:
{
  "diagnoses": [{"name": str, "icd_hint": str, "confidence": float}],
  "medications": [{"name": str, "dosage": str, "route": str}],
  "procedures": [{"name": str, "cpt_hint": str}],
  "vitals": [{"type": str, "value": str, "unit": str}],
  "allergies": [str],
  "chief_complaint": str,
  "hpi_elements": [str]
}
Do NOT include patient identifiers in your response.""",
        messages=[{"role": "user", "content": state["transcript"]}],
    )

    import json
    entities = json.loads(response.content[0].text)
    return {
        "clinical_entities": [entities],
        "status": "entities_extracted",
    }


def generate_note(state: ClinicalDocState) -> dict:
    """Generate a SOAP note from extracted entities and transcript."""
    entities = state["clinical_entities"][-1] if state["clinical_entities"] else {}

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8192,
        system="""You are an expert medical scribe. Generate a complete SOAP note from the
clinical entities and transcript provided. Follow standard SOAP format:

**Subjective**: Chief complaint, HPI (onset, location, duration, character, aggravating/
alleviating factors, radiation, timing, severity), ROS, PMH, medications, allergies,
social/family history as relevant.

**Objective**: Vitals, physical exam findings, lab/imaging results.

**Assessment**: Numbered problem list with clinical reasoning.

**Plan**: For each problem - workup, treatment, follow-up.

Use professional medical language. Be thorough but concise. Do NOT fabricate findings
not present in the source material. Do NOT include patient name or DOB in the note body.""",
        messages=[{
            "role": "user",
            "content": f"Transcript:\n{state['transcript']}\n\nExtracted Entities:\n{json.dumps(entities, indent=2)}"
        }],
    )

    return {
        "soap_note": response.content[0].text,
        "status": "note_generated",
    }


def suggest_codes(state: ClinicalDocState) -> dict:
    """Suggest ICD-10 and CPT codes based on the SOAP note and clinical entities."""
    entities = state["clinical_entities"][-1] if state["clinical_entities"] else {}

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system="""You are a certified medical coder (CPC). Based on the SOAP note and clinical
entities, suggest appropriate ICD-10-CM and CPT codes.

Rules:
- Code to the highest level of specificity supported by the documentation
- Do NOT upcode -- only code what is clearly documented
- Include laterality and episode of care where applicable
- For E/M codes, calculate the level based on MDM complexity
- Flag any codes where documentation may be insufficient

Return JSON:
{
  "icd10_codes": [{"code": str, "description": str, "confidence": float, "rationale": str}],
  "cpt_codes": [{"code": str, "description": str, "confidence": float, "rationale": str}],
  "overall_confidence": float,
  "documentation_gaps": [str]
}""",
        messages=[{
            "role": "user",
            "content": f"SOAP Note:\n{state['soap_note']}\n\nClinical Entities:\n{json.dumps(entities, indent=2)}"
        }],
    )

    import json
    codes = json.loads(response.content[0].text)

    return {
        "icd_codes": codes["icd10_codes"],
        "cpt_codes": codes["cpt_codes"],
        "confidence": codes["overall_confidence"],
        "status": "codes_suggested",
    }


def human_review(state: ClinicalDocState) -> Command[Literal["finalize", "generate_note"]]:
    """Interrupt for clinician review of the generated note and codes."""
    decision = interrupt({
        "type": "clinical_documentation_review",
        "encounter_id": state["encounter_id"],
        "soap_note": state["soap_note"],
        "icd_codes": state["icd_codes"],
        "cpt_codes": state["cpt_codes"],
        "confidence": state["confidence"],
        "message": "Please review the generated SOAP note and suggested codes.",
    })

    if decision.get("approved"):
        return Command(
            goto="finalize",
            update={
                "review_notes": decision.get("notes", ""),
                "reviewer_id": decision.get("reviewer_id"),
                "status": "approved",
                # Allow reviewer to override codes
                "icd_codes": decision.get("icd_codes", state["icd_codes"]),
                "cpt_codes": decision.get("cpt_codes", state["cpt_codes"]),
            }
        )
    else:
        return Command(
            goto="generate_note",
            update={
                "review_notes": decision.get("notes", ""),
                "reviewer_id": decision.get("reviewer_id"),
                "status": "revision_requested",
            }
        )


def auto_approve(state: ClinicalDocState) -> dict:
    """Auto-approve high-confidence results (>= 0.95)."""
    return {"status": "auto_approved", "review_notes": "Auto-approved: confidence >= 0.95"}


def escalate_to_coder(state: ClinicalDocState) -> dict:
    """Escalate low-confidence results to a human coder."""
    decision = interrupt({
        "type": "coder_escalation",
        "encounter_id": state["encounter_id"],
        "soap_note": state["soap_note"],
        "icd_codes": state["icd_codes"],
        "cpt_codes": state["cpt_codes"],
        "confidence": state["confidence"],
        "message": "Low confidence -- requires manual coding review.",
    })
    return {
        "icd_codes": decision.get("icd_codes", state["icd_codes"]),
        "cpt_codes": decision.get("cpt_codes", state["cpt_codes"]),
        "reviewer_id": decision.get("reviewer_id"),
        "review_notes": decision.get("notes", ""),
        "status": "manually_coded",
    }


def finalize(state: ClinicalDocState) -> dict:
    """Persist the finalized note and codes to the EHR."""
    from app.services.ehr import EHRService

    ehr = EHRService()
    ehr.save_note(
        encounter_id=state["encounter_id"],
        provider_id=state["provider_id"],
        patient_id=state["patient_id"],
        soap_note=state["soap_note"],
        icd_codes=state["icd_codes"],
        cpt_codes=state["cpt_codes"],
        review_notes=state["review_notes"],
        reviewer_id=state.get("reviewer_id"),
    )
    return {"status": "finalized"}


# --- Routing ---

def route_by_confidence(state: ClinicalDocState) -> Literal["auto_approve", "human_review", "escalate_to_coder"]:
    if state["confidence"] >= 0.95:
        return "auto_approve"
    elif state["confidence"] >= 0.80:
        return "human_review"
    else:
        return "escalate_to_coder"


# --- Build the graph ---

builder = StateGraph(ClinicalDocState)

builder.add_node("transcribe", transcribe)
builder.add_node("extract_entities", extract_entities)
builder.add_node("generate_note", generate_note)
builder.add_node("suggest_codes", suggest_codes)
builder.add_node("human_review", human_review)
builder.add_node("auto_approve", auto_approve)
builder.add_node("escalate_to_coder", escalate_to_coder)
builder.add_node("finalize", finalize)

builder.add_edge(START, "transcribe")
builder.add_edge("transcribe", "extract_entities")
builder.add_edge("extract_entities", "generate_note")
builder.add_edge("generate_note", "suggest_codes")
builder.add_conditional_edges("suggest_codes", route_by_confidence, {
    "auto_approve": "auto_approve",
    "human_review": "human_review",
    "escalate_to_coder": "escalate_to_coder",
})
builder.add_edge("auto_approve", "finalize")
builder.add_edge("escalate_to_coder", "finalize")
builder.add_edge("finalize", END)
# human_review routes via Command(goto=...) so no explicit edge needed

# --- Compile ---

serde = EncryptedSerializer.from_pycryptodome_aes()
checkpointer = PostgresSaver.from_conn_string(POSTGRES_URI, serde=serde)
checkpointer.setup()

clinical_doc_graph = builder.compile(checkpointer=checkpointer)
```

### Invocation

```python
config = {"configurable": {"thread_id": f"clinical-doc-{encounter_id}"}}

result = clinical_doc_graph.invoke({
    "audio_url": audio_url,
    "encounter_id": encounter_id,
    "provider_id": provider_id,
    "patient_id": patient_id,
    "clinical_entities": [],
    "icd_codes": [],
    "cpt_codes": [],
    "confidence": 0.0,
    "status": "pending",
    "retry_count": 0,
}, config=config)

# If interrupted for review:
if "__interrupt__" in result:
    review_payload = result["__interrupt__"][0].value
    # Send to frontend for clinician review
    # ...
    # When clinician responds:
    from langgraph.types import Command
    final = clinical_doc_graph.invoke(
        Command(resume={"approved": True, "reviewer_id": "dr-smith"}),
        config=config
    )
```

---

## 4. Agent 2: Prior Authorization Agent

This agent handles the full lifecycle of a prior authorization request, from determination of PA necessity through submission and monitoring. See [[Prior-Authorization-Deep-Dive]] for the clinical and regulatory context.

### State Definition

```python
class PriorAuthState(TypedDict):
    # Input
    order_id: str
    order_type: str                     # imaging, medication, procedure, referral
    order_details: dict                 # CPT, HCPCS, NDC codes, quantities
    patient_id: str
    provider_id: str
    payer_id: str
    payer_name: str

    # Coverage
    coverage_info: Optional[dict]       # Benefits, plan type, PA requirements
    pa_required: Optional[bool]
    pa_determination_reason: str

    # Clinical
    clinical_docs: Annotated[list[dict], add]   # Gathered clinical documentation
    clinical_summary: Optional[str]             # AI-generated clinical justification

    # Submission
    pa_request: Optional[dict]          # Structured PA request payload
    pa_reference_number: Optional[str]
    submission_timestamp: Optional[str]

    # Response
    pa_status: str                      # pending | approved | denied | pend_for_info | partially_approved
    pa_response: Optional[dict]
    denial_reason: Optional[str]

    # Workflow
    status: str
    retry_count: int
    error: Optional[str]
```

### Graph Definition

```python
import json
from datetime import datetime, timezone
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from typing import Literal

def check_pa_required(state: PriorAuthState) -> dict:
    """Check payer rules to determine if PA is required for this order."""
    from app.services.payer_rules import PayerRulesEngine

    engine = PayerRulesEngine()
    result = engine.check_requirement(
        payer_id=state["payer_id"],
        order_type=state["order_type"],
        codes=state["order_details"].get("codes", []),
        patient_id=state["patient_id"],
    )
    return {
        "pa_required": result.required,
        "pa_determination_reason": result.reason,
        "coverage_info": result.coverage_details,
        "status": "pa_determination_complete",
    }


def route_pa_required(state: PriorAuthState) -> Literal["gather_clinical_docs", "no_pa_needed"]:
    return "gather_clinical_docs" if state["pa_required"] else "no_pa_needed"


def no_pa_needed(state: PriorAuthState) -> dict:
    return {"status": "no_pa_required", "pa_status": "not_required"}


def gather_clinical_docs(state: PriorAuthState) -> dict:
    """Gather relevant clinical documentation from EHR for PA justification."""
    from app.services.ehr import EHRService

    ehr = EHRService()
    docs = ehr.gather_pa_documentation(
        patient_id=state["patient_id"],
        order_type=state["order_type"],
        codes=state["order_details"].get("codes", []),
    )
    return {
        "clinical_docs": docs,
        "status": "clinical_docs_gathered",
    }


def generate_justification(state: PriorAuthState) -> dict:
    """Generate clinical justification using Claude."""
    docs_summary = "\n".join([
        f"Document: {doc['type']} ({doc['date']})\n{doc['content']}"
        for doc in state["clinical_docs"]
    ])

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system=f"""You are a prior authorization specialist. Generate a compelling clinical
justification for the following order. The justification must:

1. Clearly state the medical necessity
2. Reference specific clinical findings from the documentation
3. Cite relevant clinical guidelines (ACR Appropriateness Criteria, NCCN, etc.)
4. Address the payer's likely criteria for approval
5. Use the payer's preferred terminology

Payer: {state['payer_name']}
Order Type: {state['order_type']}
Order Details: {json.dumps(state['order_details'])}

Do NOT include patient name, DOB, or SSN in the justification body. Use "the patient"
as the subject.""",
        messages=[{
            "role": "user",
            "content": f"Clinical Documentation:\n{docs_summary}"
        }],
    )

    return {
        "clinical_summary": response.content[0].text,
        "status": "justification_generated",
    }


def submit_to_payer(state: PriorAuthState) -> dict:
    """Submit PA request to payer via X12 278 or payer portal API."""
    from app.services.payer_integration import PayerIntegration

    integration = PayerIntegration(payer_id=state["payer_id"])
    result = integration.submit_pa(
        patient_id=state["patient_id"],
        provider_id=state["provider_id"],
        order_details=state["order_details"],
        clinical_summary=state["clinical_summary"],
        clinical_docs=state["clinical_docs"],
    )
    return {
        "pa_request": result.request_payload,
        "pa_reference_number": result.reference_number,
        "submission_timestamp": datetime.now(timezone.utc).isoformat(),
        "pa_status": "submitted",
        "status": "submitted_to_payer",
    }


def monitor_status(state: PriorAuthState) -> dict:
    """Check current PA status with payer."""
    from app.services.payer_integration import PayerIntegration

    integration = PayerIntegration(payer_id=state["payer_id"])
    status = integration.check_status(state["pa_reference_number"])

    return {
        "pa_status": status.decision,      # approved | denied | pend_for_info | pending
        "pa_response": status.details,
        "denial_reason": status.denial_reason,
        "status": f"payer_response_{status.decision}",
    }


def route_pa_response(state: PriorAuthState) -> Literal["handle_approved", "handle_denied", "handle_pend", "monitor_status"]:
    status_map = {
        "approved": "handle_approved",
        "partially_approved": "handle_approved",
        "denied": "handle_denied",
        "pend_for_info": "handle_pend",
        "pending": "monitor_status",
    }
    return status_map.get(state["pa_status"], "monitor_status")


def handle_approved(state: PriorAuthState) -> dict:
    return {"status": "pa_approved"}


def handle_denied(state: PriorAuthState) -> dict:
    """Notify staff and optionally trigger denial management."""
    decision = interrupt({
        "type": "pa_denied",
        "order_id": state["order_id"],
        "denial_reason": state["denial_reason"],
        "pa_reference_number": state["pa_reference_number"],
        "message": "PA denied. Appeal, modify order, or accept denial?",
        "options": ["appeal", "modify", "accept"],
    })
    return {
        "status": f"denial_{decision['action']}",
        "review_notes": decision.get("notes", ""),
    }


def handle_pend(state: PriorAuthState) -> dict:
    """Handle payer request for additional information."""
    decision = interrupt({
        "type": "additional_info_requested",
        "order_id": state["order_id"],
        "payer_request": state["pa_response"],
        "message": "Payer requests additional clinical information.",
    })
    return {
        "clinical_docs": [decision["additional_doc"]],
        "status": "additional_info_provided",
    }


# --- Build graph ---

pa_builder = StateGraph(PriorAuthState)

pa_builder.add_node("check_pa_required", check_pa_required)
pa_builder.add_node("no_pa_needed", no_pa_needed)
pa_builder.add_node("gather_clinical_docs", gather_clinical_docs)
pa_builder.add_node("generate_justification", generate_justification)
pa_builder.add_node("submit_to_payer", submit_to_payer)
pa_builder.add_node("monitor_status", monitor_status)
pa_builder.add_node("handle_approved", handle_approved)
pa_builder.add_node("handle_denied", handle_denied)
pa_builder.add_node("handle_pend", handle_pend)

pa_builder.add_edge(START, "check_pa_required")
pa_builder.add_conditional_edges("check_pa_required", route_pa_required, {
    "gather_clinical_docs": "gather_clinical_docs",
    "no_pa_needed": "no_pa_needed",
})
pa_builder.add_edge("no_pa_needed", END)
pa_builder.add_edge("gather_clinical_docs", "generate_justification")
pa_builder.add_edge("generate_justification", "submit_to_payer")
pa_builder.add_edge("submit_to_payer", "monitor_status")
pa_builder.add_conditional_edges("monitor_status", route_pa_response, {
    "handle_approved": "handle_approved",
    "handle_denied": "handle_denied",
    "handle_pend": "handle_pend",
    "monitor_status": "monitor_status",
})
pa_builder.add_edge("handle_approved", END)
pa_builder.add_edge("handle_denied", END)
pa_builder.add_edge("handle_pend", "submit_to_payer")

prior_auth_graph = pa_builder.compile(checkpointer=checkpointer)
```

---

## 5. Agent 3: Denial Management Agent

This agent analyzes claim denials, classifies the denial type, gathers supporting evidence, generates appeal letters, and manages the appeal lifecycle. See [[Revenue-Cycle-Deep-Dive]] for denial taxonomy.

### State Definition

```python
class DenialManagementState(TypedDict):
    # Input
    claim_id: str
    denial_code: str                    # CARC/RARC codes
    denial_reason: str                  # Human-readable denial reason
    original_claim: dict                # Original claim details
    patient_id: str
    payer_id: str
    payer_name: str

    # Analysis
    denial_category: str                # clinical | administrative | technical | coverage
    root_cause: str
    appeal_likelihood: float            # Predicted success probability
    similar_denials: list[dict]         # Historical similar denials and outcomes

    # Evidence
    clinical_evidence: Annotated[list[dict], add]
    policy_references: list[dict]       # Payer policy citations

    # Appeal
    appeal_letter: Optional[str]
    appeal_type: str                    # first_level | second_level | external_review
    appeal_deadline: Optional[str]

    # Outcome
    appeal_status: str                  # draft | submitted | under_review | overturned | upheld
    appeal_response: Optional[dict]

    # Workflow
    status: str
    error: Optional[str]
```

### Graph Definition

```python
def analyze_denial(state: DenialManagementState) -> dict:
    """Analyze the denial to determine category and root cause."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system="""You are a healthcare denial management expert. Analyze this claim denial
and determine:
1. Denial category: clinical (medical necessity), administrative (missing info, timely filing),
   technical (coding errors, bundling), or coverage (not covered, out of network)
2. Root cause analysis
3. Predicted appeal success likelihood (0.0-1.0)
4. Recommended appeal strategy

Return JSON:
{
  "denial_category": str,
  "root_cause": str,
  "appeal_likelihood": float,
  "appeal_strategy": str,
  "key_issues": [str]
}""",
        messages=[{
            "role": "user",
            "content": f"""Denial Code: {state['denial_code']}
Denial Reason: {state['denial_reason']}
Claim Details: {json.dumps(state['original_claim'])}
Payer: {state['payer_name']}"""
        }],
    )

    analysis = json.loads(response.content[0].text)
    return {
        "denial_category": analysis["denial_category"],
        "root_cause": analysis["root_cause"],
        "appeal_likelihood": analysis["appeal_likelihood"],
        "status": "analyzed",
    }


def route_by_category(state: DenialManagementState) -> Literal["gather_clinical_evidence", "gather_admin_evidence", "gather_technical_evidence"]:
    routing = {
        "clinical": "gather_clinical_evidence",
        "administrative": "gather_admin_evidence",
        "technical": "gather_technical_evidence",
        "coverage": "gather_clinical_evidence",
    }
    return routing.get(state["denial_category"], "gather_clinical_evidence")


def gather_clinical_evidence(state: DenialManagementState) -> dict:
    """Gather clinical evidence to support medical necessity appeal."""
    from app.services.ehr import EHRService
    from app.services.evidence import ClinicalGuidelineSearch

    ehr = EHRService()
    guidelines = ClinicalGuidelineSearch()

    clinical_records = ehr.get_relevant_records(
        patient_id=state["patient_id"],
        claim=state["original_claim"],
    )
    guideline_refs = guidelines.search(
        diagnosis_codes=state["original_claim"].get("diagnosis_codes", []),
        procedure_codes=state["original_claim"].get("procedure_codes", []),
    )

    return {
        "clinical_evidence": clinical_records,
        "policy_references": guideline_refs,
        "status": "evidence_gathered",
    }


def gather_admin_evidence(state: DenialManagementState) -> dict:
    """Gather administrative evidence (timely filing proof, auth numbers, etc.)."""
    from app.services.claims import ClaimsService

    claims = ClaimsService()
    admin_docs = claims.get_administrative_evidence(claim_id=state["claim_id"])
    return {
        "clinical_evidence": admin_docs,
        "status": "evidence_gathered",
    }


def gather_technical_evidence(state: DenialManagementState) -> dict:
    """Gather evidence for technical denials (correct coding, unbundling rationale)."""
    from app.services.coding import CodingService

    coding = CodingService()
    technical_docs = coding.get_coding_rationale(claim=state["original_claim"])
    return {
        "clinical_evidence": technical_docs,
        "status": "evidence_gathered",
    }


def generate_appeal(state: DenialManagementState) -> dict:
    """Generate appeal letter using Claude with evidence."""
    evidence_text = "\n\n".join([
        f"--- {doc['type']} ({doc.get('date', 'N/A')}) ---\n{doc['content']}"
        for doc in state["clinical_evidence"]
    ])
    policy_text = "\n".join([
        f"- {ref['source']}: {ref['citation']}"
        for ref in state["policy_references"]
    ])

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8192,
        system=f"""You are an expert healthcare appeals writer. Generate a formal appeal letter
for the following denied claim.

APPEAL LETTER REQUIREMENTS:
1. Opening: Reference claim number, date of service, patient (as "the patient"), denial reason
2. Statement of Appeal: Clearly state you are appealing the denial
3. Clinical Argument:
   - Present the clinical history demonstrating medical necessity
   - Reference specific clinical findings from the medical record
   - Cite published clinical guidelines supporting the service
   - Address the specific denial reason point by point
4. Policy Argument:
   - Reference the payer's own coverage policy if available
   - Cite CMS National Coverage Determinations or Local Coverage Determinations
   - Reference relevant state mandates or regulations
5. Conclusion: Request reversal and reprocessing
6. Tone: Professional, factual, assertive but not adversarial

Payer: {state['payer_name']}
Denial Category: {state['denial_category']}
Denial Code: {state['denial_code']}
Denial Reason: {state['denial_reason']}

Do NOT include patient name, DOB, SSN, or full address. Use "the patient" throughout.
Include placeholders [PATIENT NAME] and [DOB] where the final letter needs them.""",
        messages=[{
            "role": "user",
            "content": f"""Claim Details:
{json.dumps(state['original_claim'], indent=2)}

Clinical Evidence:
{evidence_text}

Policy References:
{policy_text}"""
        }],
    )

    return {
        "appeal_letter": response.content[0].text,
        "appeal_type": "first_level",
        "status": "appeal_drafted",
    }


def review_appeal(state: DenialManagementState) -> Command[Literal["submit_appeal", "generate_appeal"]]:
    """Human review of generated appeal letter."""
    decision = interrupt({
        "type": "appeal_review",
        "claim_id": state["claim_id"],
        "appeal_letter": state["appeal_letter"],
        "denial_category": state["denial_category"],
        "appeal_likelihood": state["appeal_likelihood"],
        "message": "Review the generated appeal letter.",
    })

    if decision.get("approved"):
        return Command(
            goto="submit_appeal",
            update={
                "appeal_letter": decision.get("edited_letter", state["appeal_letter"]),
                "status": "appeal_approved",
            }
        )
    else:
        return Command(
            goto="generate_appeal",
            update={
                "status": "appeal_revision_requested",
            }
        )


def submit_appeal(state: DenialManagementState) -> dict:
    """Submit the appeal to the payer."""
    from app.services.payer_integration import PayerIntegration

    integration = PayerIntegration(payer_id=state["payer_id"])
    result = integration.submit_appeal(
        claim_id=state["claim_id"],
        appeal_letter=state["appeal_letter"],
        supporting_docs=state["clinical_evidence"],
        appeal_type=state["appeal_type"],
    )
    return {
        "appeal_status": "submitted",
        "appeal_response": {"submission_id": result.submission_id},
        "appeal_deadline": result.response_deadline,
        "status": "appeal_submitted",
    }


# --- Build graph ---

denial_builder = StateGraph(DenialManagementState)

denial_builder.add_node("analyze_denial", analyze_denial)
denial_builder.add_node("gather_clinical_evidence", gather_clinical_evidence)
denial_builder.add_node("gather_admin_evidence", gather_admin_evidence)
denial_builder.add_node("gather_technical_evidence", gather_technical_evidence)
denial_builder.add_node("generate_appeal", generate_appeal)
denial_builder.add_node("review_appeal", review_appeal)
denial_builder.add_node("submit_appeal", submit_appeal)

denial_builder.add_edge(START, "analyze_denial")
denial_builder.add_conditional_edges("analyze_denial", route_by_category, {
    "gather_clinical_evidence": "gather_clinical_evidence",
    "gather_admin_evidence": "gather_admin_evidence",
    "gather_technical_evidence": "gather_technical_evidence",
})
denial_builder.add_edge("gather_clinical_evidence", "generate_appeal")
denial_builder.add_edge("gather_admin_evidence", "generate_appeal")
denial_builder.add_edge("gather_technical_evidence", "generate_appeal")
denial_builder.add_edge("generate_appeal", "review_appeal")
# review_appeal routes via Command(goto=...)
denial_builder.add_edge("submit_appeal", END)

denial_graph = denial_builder.compile(checkpointer=checkpointer)
```

---

## 6. Agent 4: AI Coding Agent

This agent parses clinical notes, extracts diagnoses and procedures, maps them to ICD-10 and CPT codes using embeddings-based lookup, validates bundling rules, and presents the result for coder review. See [[Revenue-Cycle-Deep-Dive]] for coding compliance requirements.

### State Definition

```python
class AICodingState(TypedDict):
    # Input
    encounter_id: str
    clinical_note: str
    provider_id: str
    note_type: str                      # progress_note | operative_report | discharge_summary

    # Extraction
    extracted_diagnoses: list[dict]      # [{"text": str, "context": str, "negated": bool}]
    extracted_procedures: list[dict]     # [{"text": str, "context": str, "laterality": str}]

    # Coding
    suggested_icd10: list[dict]         # [{"code": str, "desc": str, "confidence": float, "rationale": str}]
    suggested_cpt: list[dict]
    bundling_warnings: list[str]        # NCCI edit violations
    specificity_warnings: list[str]     # Codes that could be more specific
    compliance_flags: list[str]         # Potential upcoding or unbundling issues

    # Review
    confidence_scores: dict             # {"overall": float, "icd": float, "cpt": float}
    coder_review: Optional[dict]        # Coder corrections and final codes
    status: str
    error: Optional[str]
```

### Graph Definition

```python
def parse_note(state: AICodingState) -> dict:
    """Parse clinical note structure and identify codeable sections."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system="""You are a clinical NLP parser. Analyze the clinical note and extract
all codeable clinical statements. For each statement identify:
- The clinical finding or condition
- Whether it is negated ("no evidence of...", "denies...")
- The supporting context
- Laterality if applicable
- Acuity (acute, chronic, recurrent)
- Episode (initial encounter, subsequent, sequela)

Return JSON:
{
  "diagnoses": [{"text": str, "context": str, "negated": bool, "acuity": str, "episode": str}],
  "procedures": [{"text": str, "context": str, "laterality": str, "approach": str}],
  "note_quality": float
}

CRITICAL: Mark negated findings as negated=true. These must NOT be coded.""",
        messages=[{"role": "user", "content": state["clinical_note"]}],
    )

    parsed = json.loads(response.content[0].text)
    # Filter out negated diagnoses
    active_diagnoses = [d for d in parsed["diagnoses"] if not d.get("negated", False)]

    return {
        "extracted_diagnoses": active_diagnoses,
        "extracted_procedures": parsed["procedures"],
        "status": "parsed",
    }


def map_to_icd10(state: AICodingState) -> dict:
    """Map extracted diagnoses to ICD-10-CM codes using pgvector embeddings + LLM validation."""
    from app.services.code_lookup import CodeLookupService

    lookup = CodeLookupService()  # Uses pgvector for semantic search
    suggested_codes = []

    for diagnosis in state["extracted_diagnoses"]:
        # Step 1: Semantic search for candidate codes
        candidates = lookup.search_icd10(
            query=diagnosis["text"],
            context=diagnosis.get("context", ""),
            top_k=5,
        )

        # Step 2: LLM selects best code with rationale
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system="""You are a certified medical coder. Given the clinical finding and
candidate ICD-10-CM codes, select the most specific appropriate code.

Rules:
- Code to highest specificity supported by documentation
- Consider laterality, episode of care, and acuity
- If documentation is insufficient for a specific code, use the unspecified code
- NEVER upcode -- only code what is documented
- Return your confidence (0.0-1.0) that this code is correct

Return JSON: {"code": str, "description": str, "confidence": float, "rationale": str}""",
            messages=[{
                "role": "user",
                "content": f"""Finding: {diagnosis['text']}
Context: {diagnosis.get('context', '')}
Acuity: {diagnosis.get('acuity', 'unknown')}
Episode: {diagnosis.get('episode', 'unknown')}

Candidate codes:
{json.dumps([{"code": c.code, "description": c.description, "score": c.similarity} for c in candidates], indent=2)}"""
            }],
        )

        code_result = json.loads(response.content[0].text)
        suggested_codes.append(code_result)

    return {"suggested_icd10": suggested_codes, "status": "icd_mapped"}


def map_to_cpt(state: AICodingState) -> dict:
    """Map extracted procedures to CPT codes."""
    from app.services.code_lookup import CodeLookupService

    lookup = CodeLookupService()
    suggested_codes = []

    for procedure in state["extracted_procedures"]:
        candidates = lookup.search_cpt(
            query=procedure["text"],
            context=procedure.get("context", ""),
            top_k=5,
        )

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system="""You are a certified medical coder. Select the most appropriate CPT code
for the procedure described. Consider approach, laterality, and extent.

For E/M services, evaluate based on Medical Decision Making (MDM) complexity:
- Number and complexity of problems addressed
- Amount and complexity of data reviewed
- Risk of complications or morbidity

Return JSON: {"code": str, "description": str, "confidence": float, "rationale": str}""",
            messages=[{
                "role": "user",
                "content": f"""Procedure: {procedure['text']}
Context: {procedure.get('context', '')}
Laterality: {procedure.get('laterality', 'N/A')}
Approach: {procedure.get('approach', 'N/A')}

Candidate codes:
{json.dumps([{"code": c.code, "description": c.description, "score": c.similarity} for c in candidates], indent=2)}"""
            }],
        )

        code_result = json.loads(response.content[0].text)
        suggested_codes.append(code_result)

    return {"suggested_cpt": suggested_codes, "status": "cpt_mapped"}


def validate_bundling(state: AICodingState) -> dict:
    """Check NCCI bundling edits and compliance rules."""
    from app.services.compliance import NCCIValidator, UpcodingDetector

    ncci = NCCIValidator()
    upcoding = UpcodingDetector()

    cpt_codes = [c["code"] for c in state["suggested_cpt"]]
    icd_codes = [c["code"] for c in state["suggested_icd10"]]

    # Check NCCI edits
    bundling_warnings = ncci.check_bundling(cpt_codes)

    # Check for potential upcoding
    compliance_flags = upcoding.check(
        cpt_codes=cpt_codes,
        icd_codes=icd_codes,
        note_text=state["clinical_note"],
        note_type=state["note_type"],
    )

    # Check code specificity
    specificity_warnings = []
    for code_info in state["suggested_icd10"]:
        if code_info["code"].endswith(".9") or code_info["code"].endswith("0"):
            specificity_warnings.append(
                f"{code_info['code']}: Consider more specific code if documentation supports it"
            )

    # Calculate confidence scores
    icd_conf = sum(c["confidence"] for c in state["suggested_icd10"]) / max(len(state["suggested_icd10"]), 1)
    cpt_conf = sum(c["confidence"] for c in state["suggested_cpt"]) / max(len(state["suggested_cpt"]), 1)
    overall = (icd_conf + cpt_conf) / 2

    # Reduce confidence if there are warnings
    if bundling_warnings:
        overall *= 0.9
    if compliance_flags:
        overall *= 0.8

    return {
        "bundling_warnings": bundling_warnings,
        "specificity_warnings": specificity_warnings,
        "compliance_flags": compliance_flags,
        "confidence_scores": {"overall": overall, "icd": icd_conf, "cpt": cpt_conf},
        "status": "validated",
    }


def coder_review_node(state: AICodingState) -> dict:
    """Human coder reviews and finalizes codes."""
    decision = interrupt({
        "type": "coder_review",
        "encounter_id": state["encounter_id"],
        "suggested_icd10": state["suggested_icd10"],
        "suggested_cpt": state["suggested_cpt"],
        "bundling_warnings": state["bundling_warnings"],
        "specificity_warnings": state["specificity_warnings"],
        "compliance_flags": state["compliance_flags"],
        "confidence_scores": state["confidence_scores"],
        "clinical_note": state["clinical_note"],
        "message": "Review suggested codes and finalize.",
    })

    return {
        "coder_review": decision,
        "suggested_icd10": decision.get("final_icd10", state["suggested_icd10"]),
        "suggested_cpt": decision.get("final_cpt", state["suggested_cpt"]),
        "status": "finalized",
    }


# --- Build graph ---

coding_builder = StateGraph(AICodingState)

coding_builder.add_node("parse_note", parse_note)
coding_builder.add_node("map_to_icd10", map_to_icd10)
coding_builder.add_node("map_to_cpt", map_to_cpt)
coding_builder.add_node("validate_bundling", validate_bundling)
coding_builder.add_node("coder_review", coder_review_node)

coding_builder.add_edge(START, "parse_note")
coding_builder.add_edge("parse_note", "map_to_icd10")
coding_builder.add_edge("map_to_icd10", "map_to_cpt")
coding_builder.add_edge("map_to_cpt", "validate_bundling")
coding_builder.add_edge("validate_bundling", "coder_review")
coding_builder.add_edge("coder_review", END)

coding_graph = coding_builder.compile(checkpointer=checkpointer)
```

### pgvector Code Lookup Service

```python
# app/services/code_lookup.py
from dataclasses import dataclass
import psycopg2
from anthropic import Anthropic

@dataclass
class CodeCandidate:
    code: str
    description: str
    similarity: float

class CodeLookupService:
    """Semantic code lookup using pgvector embeddings."""

    def __init__(self):
        self.conn = psycopg2.connect(POSTGRES_URI)
        self.client = Anthropic()

    def _get_embedding(self, text: str) -> list[float]:
        """Generate embedding for semantic search."""
        # Using a dedicated embedding model
        from app.services.embeddings import get_embedding
        return get_embedding(text)

    def search_icd10(self, query: str, context: str, top_k: int = 5) -> list[CodeCandidate]:
        combined = f"{query} {context}"
        embedding = self._get_embedding(combined)

        with self.conn.cursor() as cur:
            cur.execute("""
                SELECT code, description, 1 - (embedding <=> %s::vector) as similarity
                FROM icd10_codes
                ORDER BY embedding <=> %s::vector
                LIMIT %s
            """, (embedding, embedding, top_k))

            return [CodeCandidate(code=row[0], description=row[1], similarity=row[2])
                    for row in cur.fetchall()]

    def search_cpt(self, query: str, context: str, top_k: int = 5) -> list[CodeCandidate]:
        combined = f"{query} {context}"
        embedding = self._get_embedding(combined)

        with self.conn.cursor() as cur:
            cur.execute("""
                SELECT code, description, 1 - (embedding <=> %s::vector) as similarity
                FROM cpt_codes
                ORDER BY embedding <=> %s::vector
                LIMIT %s
            """, (embedding, embedding, top_k))

            return [CodeCandidate(code=row[0], description=row[1], similarity=row[2])
                    for row in cur.fetchall()]
```

---

## 7. HIPAA Compliance Layer

Every LLM call in every agent passes through a compliance wrapper. This enforces the minimum necessary principle, audit logging, and prompt injection defense. See [[HIPAA-Deep-Dive]] for the regulatory framework.

### Compliant LLM Wrapper

```python
# app/ai/hipaa_wrapper.py
import hashlib
import json
import re
import time
from dataclasses import dataclass, field
from typing import Optional
from uuid import uuid4

import anthropic
import structlog

logger = structlog.get_logger()


@dataclass
class AuditEntry:
    request_id: str
    timestamp: float
    agent_name: str
    node_name: str
    tenant_id: str
    user_id: str
    prompt_hash: str           # SHA-256 of the full prompt (NOT the prompt itself)
    response_hash: str         # SHA-256 of the full response
    model: str
    input_tokens: int
    output_tokens: int
    latency_ms: float
    phi_detected_in_prompt: bool
    phi_scrubbed: bool
    error: Optional[str] = None


class PHIScrubber:
    """Detect and minimize PHI in LLM prompts."""

    # Patterns for common PHI elements
    PATTERNS = {
        "ssn": re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),
        "phone": re.compile(r"\b(?:\+1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b"),
        "email": re.compile(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b"),
        "mrn": re.compile(r"\bMRN[:\s#]*\d{6,}\b", re.IGNORECASE),
        "dob": re.compile(r"\b(?:DOB|Date of Birth)[:\s]*\d{1,2}[/-]\d{1,2}[/-]\d{2,4}\b", re.IGNORECASE),
        "address": re.compile(r"\b\d{1,5}\s+\w+\s+(?:St|Ave|Blvd|Dr|Rd|Ln|Ct|Way)\b", re.IGNORECASE),
    }

    def scrub(self, text: str) -> tuple[str, bool]:
        """Remove unnecessary PHI from text. Returns (scrubbed_text, was_phi_found)."""
        phi_found = False
        scrubbed = text

        for phi_type, pattern in self.PATTERNS.items():
            if pattern.search(scrubbed):
                phi_found = True
                scrubbed = pattern.sub(f"[REDACTED_{phi_type.upper()}]", scrubbed)

        return scrubbed, phi_found


class PromptInjectionDetector:
    """Detect prompt injection attempts in clinical text."""

    INJECTION_PATTERNS = [
        re.compile(r"ignore\s+(all\s+)?previous\s+instructions", re.IGNORECASE),
        re.compile(r"you\s+are\s+now\s+a", re.IGNORECASE),
        re.compile(r"forget\s+(everything|all|your)\s+", re.IGNORECASE),
        re.compile(r"system\s*:\s*", re.IGNORECASE),
        re.compile(r"<\s*/?system\s*>", re.IGNORECASE),
        re.compile(r"```\s*system", re.IGNORECASE),
        re.compile(r"ADMIN\s*OVERRIDE", re.IGNORECASE),
    ]

    def check(self, text: str) -> bool:
        """Returns True if injection attempt detected."""
        for pattern in self.INJECTION_PATTERNS:
            if pattern.search(text):
                return True
        return False


class HIPAACompliantLLM:
    """Wrapper around Anthropic client that enforces HIPAA compliance."""

    def __init__(self, tenant_id: str, agent_name: str):
        self.client = anthropic.Anthropic()
        self.tenant_id = tenant_id
        self.agent_name = agent_name
        self.scrubber = PHIScrubber()
        self.injection_detector = PromptInjectionDetector()
        self.audit_logger = AuditLogger()

    def create_message(
        self,
        *,
        model: str,
        max_tokens: int,
        system: str,
        messages: list[dict],
        node_name: str,
        user_id: str,
        scrub_phi: bool = True,
    ) -> anthropic.types.Message:
        """Create a message with HIPAA compliance enforced."""
        request_id = str(uuid4())
        start_time = time.monotonic()

        # 1. Check for prompt injection in user-provided content
        for msg in messages:
            content = msg.get("content", "")
            if isinstance(content, str) and self.injection_detector.check(content):
                logger.warning(
                    "prompt_injection_detected",
                    request_id=request_id,
                    agent=self.agent_name,
                    node=node_name,
                )
                raise ValueError(f"Potential prompt injection detected in input (request_id={request_id})")

        # 2. Scrub PHI from prompts (minimum necessary principle)
        phi_detected = False
        if scrub_phi:
            scrubbed_messages = []
            for msg in messages:
                content = msg.get("content", "")
                if isinstance(content, str):
                    scrubbed_content, found = self.scrubber.scrub(content)
                    phi_detected = phi_detected or found
                    scrubbed_messages.append({**msg, "content": scrubbed_content})
                else:
                    scrubbed_messages.append(msg)
            messages = scrubbed_messages

        # 3. Call the LLM
        error_msg = None
        try:
            response = self.client.messages.create(
                model=model,
                max_tokens=max_tokens,
                system=system,
                messages=messages,
            )
        except Exception as e:
            error_msg = str(e)
            raise
        finally:
            # 4. Audit log (always, even on failure)
            elapsed_ms = (time.monotonic() - start_time) * 1000
            prompt_text = json.dumps({"system": system, "messages": messages})
            response_text = response.content[0].text if not error_msg else ""

            entry = AuditEntry(
                request_id=request_id,
                timestamp=time.time(),
                agent_name=self.agent_name,
                node_name=node_name,
                tenant_id=self.tenant_id,
                user_id=user_id,
                prompt_hash=hashlib.sha256(prompt_text.encode()).hexdigest(),
                response_hash=hashlib.sha256(response_text.encode()).hexdigest(),
                model=model,
                input_tokens=response.usage.input_tokens if not error_msg else 0,
                output_tokens=response.usage.output_tokens if not error_msg else 0,
                latency_ms=elapsed_ms,
                phi_detected_in_prompt=phi_detected,
                phi_scrubbed=scrub_phi and phi_detected,
                error=error_msg,
            )
            self.audit_logger.log(entry)

        return response


class AuditLogger:
    """Persist audit entries to PostgreSQL for HIPAA compliance."""

    def log(self, entry: AuditEntry):
        """Write audit entry to the llm_audit_log table."""
        import psycopg2

        with psycopg2.connect(POSTGRES_URI) as conn:
            with conn.cursor() as cur:
                cur.execute("""
                    INSERT INTO llm_audit_log (
                        request_id, timestamp, agent_name, node_name, tenant_id,
                        user_id, prompt_hash, response_hash, model, input_tokens,
                        output_tokens, latency_ms, phi_detected, phi_scrubbed, error
                    ) VALUES (%s, to_timestamp(%s), %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, (
                    entry.request_id, entry.timestamp, entry.agent_name, entry.node_name,
                    entry.tenant_id, entry.user_id, entry.prompt_hash, entry.response_hash,
                    entry.model, entry.input_tokens, entry.output_tokens, entry.latency_ms,
                    entry.phi_detected_in_prompt, entry.phi_scrubbed, entry.error,
                ))
            conn.commit()

        # Also emit structured log for real-time monitoring
        logger.info(
            "llm_call_audit",
            request_id=entry.request_id,
            agent=entry.agent_name,
            node=entry.node_name,
            model=entry.model,
            tokens_in=entry.input_tokens,
            tokens_out=entry.output_tokens,
            latency_ms=round(entry.latency_ms, 1),
            phi_detected=entry.phi_detected_in_prompt,
        )
```

### Anthropic BAA Terms

Under the Anthropic Business Associate Agreement:

- **No training on PHI**: Messages sent via the API are not used for model training.
- **Data retention**: API messages are retained for 30 days for abuse monitoring, then deleted.
- **Encryption**: All data is encrypted in transit (TLS 1.2+) and at rest (AES-256).
- **Access controls**: SOC 2 Type II certified access controls.

Despite these protections, our compliance layer enforces **defense in depth**: scrub PHI before it leaves our infrastructure whenever possible.

---

## 8. Observability and Monitoring

### Langfuse Integration

```python
# app/ai/observability.py
from langfuse import Langfuse
from langfuse.callback import CallbackHandler as LangfuseCallbackHandler
from contextlib import contextmanager

langfuse = Langfuse(
    public_key="YOUR_SECRET_HERE",
    secret_key="YOUR_SECRET_HERE",
    host="https://langfuse.medos.internal",
)


@contextmanager
def traced_agent_run(agent_name: str, tenant_id: str, run_id: str):
    """Context manager for tracing an entire agent run in Langfuse."""
    trace = langfuse.trace(
        name=f"{agent_name}_run",
        id=run_id,
        metadata={
            "tenant_id": tenant_id,
            "agent": agent_name,
        },
    )
    try:
        yield trace
    finally:
        trace.update(status_message="completed")
        langfuse.flush()


def create_langfuse_handler(trace_id: str, agent_name: str) -> LangfuseCallbackHandler:
    """Create a Langfuse callback handler for LangGraph integration."""
    return LangfuseCallbackHandler(
        trace_id=trace_id,
        tags=[agent_name],
    )
```

### Cost Tracking per Agent per Tenant

```python
# app/ai/cost_tracker.py
from dataclasses import dataclass
from datetime import datetime, timezone

# Claude pricing (as of 2026)
MODEL_PRICING = {
    "claude-sonnet-4-20250514": {"input": 3.00, "output": 15.00},    # per 1M tokens
    "claude-haiku-4-5-20251001": {"input": 0.80, "output": 4.00},
}


@dataclass
class CostRecord:
    tenant_id: str
    agent_name: str
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float
    timestamp: datetime


class CostTracker:
    """Track LLM costs per agent per tenant."""

    def calculate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        pricing = MODEL_PRICING.get(model, {"input": 3.00, "output": 15.00})
        input_cost = (input_tokens / 1_000_000) * pricing["input"]
        output_cost = (output_tokens / 1_000_000) * pricing["output"]
        return round(input_cost + output_cost, 6)

    def record(self, tenant_id: str, agent_name: str, model: str,
               input_tokens: int, output_tokens: int):
        cost = self.calculate_cost(model, input_tokens, output_tokens)

        with psycopg2.connect(POSTGRES_URI) as conn:
            with conn.cursor() as cur:
                cur.execute("""
                    INSERT INTO llm_cost_tracking
                    (tenant_id, agent_name, model, input_tokens, output_tokens, cost_usd, timestamp)
                    VALUES (%s, %s, %s, %s, %s, %s, %s)
                """, (tenant_id, agent_name, model, input_tokens, output_tokens,
                      cost, datetime.now(timezone.utc)))
            conn.commit()

    def get_monthly_cost(self, tenant_id: str) -> dict:
        """Get current month cost breakdown by agent."""
        with psycopg2.connect(POSTGRES_URI) as conn:
            with conn.cursor() as cur:
                cur.execute("""
                    SELECT agent_name, SUM(cost_usd), SUM(input_tokens), SUM(output_tokens)
                    FROM llm_cost_tracking
                    WHERE tenant_id = %s
                      AND timestamp >= date_trunc('month', CURRENT_DATE)
                    GROUP BY agent_name
                """, (tenant_id,))
                return {
                    row[0]: {"cost_usd": float(row[1]), "input_tokens": row[2], "output_tokens": row[3]}
                    for row in cur.fetchall()
                }
```

### Quality Metrics Dashboard

Key metrics tracked per agent:

| Metric | Description | Target |
|---|---|---|
| **Confidence Calibration** | Predicted confidence vs actual accuracy | Brier score < 0.1 |
| **Auto-approve Rate** | Percentage of runs that auto-approve (conf >= 0.95) | 60-70% |
| **Human Override Rate** | How often humans change AI suggestions | < 15% |
| **Latency P95** | 95th percentile end-to-end latency | < 30s for coding, < 60s for documentation |
| **Error Rate** | Percentage of runs that fail | < 1% |
| **Cost per Encounter** | Average LLM cost per clinical encounter | < $0.50 |

---

## 9. Testing AI Agents

### Unit Testing Individual Nodes

```python
# tests/agents/test_clinical_doc_nodes.py
import pytest
from unittest.mock import patch, MagicMock
from app.agents.clinical_doc import extract_entities, suggest_codes, route_by_confidence


class TestExtractEntities:
    """Test the entity extraction node in isolation."""

    @patch("app.agents.clinical_doc.client")
    def test_extracts_diagnoses_from_transcript(self, mock_client):
        mock_response = MagicMock()
        mock_response.content = [MagicMock(text='{"diagnoses": [{"name": "acute bronchitis", "icd_hint": "J20.9", "confidence": 0.9}], "medications": [], "procedures": [], "vitals": [], "allergies": [], "chief_complaint": "cough", "hpi_elements": ["cough for 5 days"]}')]
        mock_client.messages.create.return_value = mock_response

        state = {
            "transcript": "Patient presents with cough for 5 days...",
            "clinical_entities": [],
        }
        result = extract_entities(state)

        assert len(result["clinical_entities"]) == 1
        assert result["clinical_entities"][0]["diagnoses"][0]["name"] == "acute bronchitis"
        assert result["status"] == "entities_extracted"


class TestRouteByConfidence:
    def test_auto_approve_high_confidence(self):
        assert route_by_confidence({"confidence": 0.97}) == "auto_approve"

    def test_human_review_medium_confidence(self):
        assert route_by_confidence({"confidence": 0.88}) == "human_review"

    def test_escalate_low_confidence(self):
        assert route_by_confidence({"confidence": 0.72}) == "escalate_to_coder"

    def test_boundary_095(self):
        assert route_by_confidence({"confidence": 0.95}) == "auto_approve"

    def test_boundary_080(self):
        assert route_by_confidence({"confidence": 0.80}) == "human_review"
```

### Integration Testing Full Graph Execution

```python
# tests/agents/test_clinical_doc_integration.py
import pytest
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command
from app.agents.clinical_doc import builder as clinical_doc_builder


@pytest.fixture
def graph():
    """Compile graph with in-memory checkpointer for testing."""
    checkpointer = InMemorySaver()
    return clinical_doc_builder.compile(checkpointer=checkpointer)


class TestClinicalDocGraphIntegration:
    def test_full_flow_auto_approve(self, graph, mock_transcription, mock_anthropic):
        """Test end-to-end flow when confidence is high enough for auto-approval."""
        mock_anthropic.set_confidence(0.97)

        config = {"configurable": {"thread_id": "test-auto-approve"}}
        result = graph.invoke({
            "audio_url": "https://storage.example.com/encounter-123.wav",
            "encounter_id": "enc-123",
            "provider_id": "prov-456",
            "patient_id": "pat-789",
            "clinical_entities": [],
            "icd_codes": [],
            "cpt_codes": [],
            "confidence": 0.0,
            "status": "pending",
            "retry_count": 0,
        }, config=config)

        assert result["status"] == "finalized"
        assert "__interrupt__" not in result

    def test_flow_with_human_review(self, graph, mock_transcription, mock_anthropic):
        """Test flow that pauses for human review and resumes."""
        mock_anthropic.set_confidence(0.88)

        config = {"configurable": {"thread_id": "test-human-review"}}

        # First invocation -- should pause at human_review
        result = graph.invoke({
            "audio_url": "https://storage.example.com/encounter-456.wav",
            "encounter_id": "enc-456",
            "provider_id": "prov-456",
            "patient_id": "pat-789",
            "clinical_entities": [],
            "icd_codes": [],
            "cpt_codes": [],
            "confidence": 0.0,
            "status": "pending",
            "retry_count": 0,
        }, config=config)

        assert "__interrupt__" in result
        interrupt_payload = result["__interrupt__"][0].value
        assert interrupt_payload["type"] == "clinical_documentation_review"

        # Resume with approval
        final = graph.invoke(
            Command(resume={"approved": True, "reviewer_id": "dr-jones"}),
            config=config,
        )

        assert final["status"] == "finalized"
```

### Evaluation Datasets

```python
# tests/agents/eval/clinical_coding_eval.py
"""
Evaluation framework for AI coding accuracy.
Uses synthetic clinical scenarios with known-correct codes.
"""

EVAL_DATASET = [
    {
        "id": "eval-001",
        "clinical_note": """CC: Chest pain
HPI: 55yo M presents with substernal chest pain x 2 hours, radiating to left arm.
Associated with diaphoresis and nausea. No prior cardiac history.
PE: BP 160/95, HR 102, RR 20. Diaphoretic. S1S2 regular, no murmurs.
ECG: ST elevation in leads II, III, aVF.
Assessment: STEMI, inferior wall.
Plan: Activate cath lab, ASA 325mg, heparin bolus, emergent PCI.""",
        "expected_icd10": ["I21.19"],       # STEMI of inferior wall, unspecified
        "expected_cpt": ["92928"],           # PCI with stent
        "acceptable_icd10": ["I21.11", "I21.19"],
    },
    {
        "id": "eval-002",
        "clinical_note": """CC: Follow-up diabetes management
HPI: 62yo F with T2DM on metformin 1000mg BID. A1c today 7.8%, up from 7.2%.
Reports good compliance. Denies hypoglycemia.
PE: BMI 31, monofilament intact bilaterally, no retinopathy on fundoscopic.
Assessment: T2DM with suboptimal control
Plan: Add empagliflozin 10mg daily, recheck A1c in 3 months.""",
        "expected_icd10": ["E11.65"],       # T2DM with hyperglycemia
        "expected_cpt": ["99214"],           # E/M level 4
        "acceptable_icd10": ["E11.65", "E11.9"],
    },
]


def evaluate_coding_agent(graph, dataset=EVAL_DATASET):
    """Run evaluation dataset through coding agent and report accuracy."""
    results = []

    for case in dataset:
        config = {"configurable": {"thread_id": f"eval-{case['id']}"}}
        output = graph.invoke({
            "encounter_id": case["id"],
            "clinical_note": case["clinical_note"],
            "provider_id": "eval-provider",
            "note_type": "progress_note",
            "extracted_diagnoses": [],
            "extracted_procedures": [],
            "suggested_icd10": [],
            "suggested_cpt": [],
            "bundling_warnings": [],
            "specificity_warnings": [],
            "compliance_flags": [],
            "confidence_scores": {},
            "status": "pending",
        }, config=config)

        # Check if interrupt hit (skip for eval mode)
        suggested_icd = [c["code"] for c in output.get("suggested_icd10", [])]
        suggested_cpt = [c["code"] for c in output.get("suggested_cpt", [])]

        icd_correct = any(code in case["acceptable_icd10"] for code in suggested_icd)
        cpt_correct = any(code in case["expected_cpt"] for code in suggested_cpt)

        results.append({
            "case_id": case["id"],
            "icd_correct": icd_correct,
            "cpt_correct": cpt_correct,
            "suggested_icd": suggested_icd,
            "suggested_cpt": suggested_cpt,
            "expected_icd": case["expected_icd10"],
            "expected_cpt": case["expected_cpt"],
        })

    # Aggregate metrics
    icd_accuracy = sum(r["icd_correct"] for r in results) / len(results)
    cpt_accuracy = sum(r["cpt_correct"] for r in results) / len(results)

    return {
        "icd_accuracy": icd_accuracy,
        "cpt_accuracy": cpt_accuracy,
        "total_cases": len(results),
        "details": results,
    }
```

### A/B Testing for Prompt Improvements

```python
# app/ai/ab_testing.py
import random
from dataclasses import dataclass
from typing import Optional


@dataclass
class PromptVariant:
    name: str
    system_prompt: str
    weight: float              # Traffic allocation (0.0 - 1.0)


class PromptABTest:
    """A/B test framework for prompt improvements."""

    def __init__(self, test_name: str, variants: list[PromptVariant]):
        self.test_name = test_name
        self.variants = variants
        assert abs(sum(v.weight for v in variants) - 1.0) < 0.01, "Weights must sum to 1.0"

    def select_variant(self, deterministic_key: Optional[str] = None) -> PromptVariant:
        """Select a variant. Use deterministic_key for consistent assignment."""
        if deterministic_key:
            # Consistent hashing for deterministic assignment
            hash_val = hash(f"{self.test_name}:{deterministic_key}") % 1000
            cumulative = 0
            for variant in self.variants:
                cumulative += variant.weight * 1000
                if hash_val < cumulative:
                    return variant
            return self.variants[-1]
        else:
            r = random.random()
            cumulative = 0
            for variant in self.variants:
                cumulative += variant.weight
                if r < cumulative:
                    return variant
            return self.variants[-1]


# Example usage
coding_prompt_test = PromptABTest(
    test_name="coding_agent_v2_vs_v1",
    variants=[
        PromptVariant(
            name="v1_baseline",
            system_prompt="You are a certified medical coder...",
            weight=0.5,
        ),
        PromptVariant(
            name="v2_chain_of_thought",
            system_prompt="You are a certified medical coder. Think step by step...",
            weight=0.5,
        ),
    ]
)
```

---

## 10. Deployment Architecture

### Infrastructure Overview

```
                    +-------------------+
                    |   API Gateway     |
                    |   (FastAPI)       |
                    +--------+----------+
                             |
                    +--------v----------+
                    |   Redis Queue     |
                    |  (Task Broker)    |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v-----+  +-----v------+
     | Agent       |  | Agent      |  | Agent      |
     | Worker 1    |  | Worker 2   |  | Worker N   |
     | (Fargate)   |  | (Fargate)  |  | (Fargate)  |
     +--------+----+  +------+-----+  +-----+------+
              |              |              |
              +--------------+--------------+
                             |
                    +--------v----------+
                    |   PostgreSQL      |
                    | - Checkpoints     |
                    | - Audit Logs      |
                    | - Code Embeddings |
                    +-------------------+
```

### ECS Fargate Task Definition

```python
# terraform/modules/agent_workers/main.tf (conceptual, shown as Python dict for clarity)

AGENT_WORKER_TASK = {
    "family": "medos-agent-worker",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "1024",          # 1 vCPU
    "memory": "2048",       # 2 GB
    "containerDefinitions": [{
        "name": "agent-worker",
        "image": "ECR_REPO/medos-agent-worker:latest",
        "essential": True,
        "environment": [
            {"name": "AGENT_TYPES", "value": "clinical_doc,prior_auth,denial,coding"},
            {"name": "REDIS_URL", "value": "redis://redis.medos.internal:6379"},
            {"name": "LANGFUSE_HOST", "value": "https://langfuse.medos.internal"},
        ],
        "secrets": [
            {
                "name": "ANTHROPIC_API_KEY",
                "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:medos/anthropic-api-key:::"
            },
            {
                "name": "DATABASE_URL",
                "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:medos/database-url:::"
            },
            {
                "name": "LANGGRAPH_AES_KEY",
                "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:medos/langgraph-aes-key:::"
            },
        ],
        "logConfiguration": {
            "logDriver": "awslogs",
            "options": {
                "awslogs-group": "/ecs/medos-agent-worker",
                "awslogs-region": "us-east-1",
                "awslogs-stream-prefix": "agent",
            },
        },
    }],
}
```

### Worker Process

```python
# app/workers/agent_worker.py
import asyncio
import json
import redis.asyncio as redis
import structlog
from app.agents.clinical_doc import clinical_doc_graph
from app.agents.prior_auth import prior_auth_graph
from app.agents.denial import denial_graph
from app.agents.coding import coding_graph
from langgraph.types import Command

logger = structlog.get_logger()

AGENT_GRAPHS = {
    "clinical_doc": clinical_doc_graph,
    "prior_auth": prior_auth_graph,
    "denial": denial_graph,
    "coding": coding_graph,
}


async def process_task(task_data: dict):
    """Process a single agent task from the queue."""
    agent_type = task_data["agent_type"]
    graph = AGENT_GRAPHS[agent_type]
    config = {"configurable": {"thread_id": task_data["thread_id"]}}

    logger.info("agent_task_started", agent=agent_type, thread_id=task_data["thread_id"])

    try:
        if task_data.get("resume"):
            # Resuming from interrupt
            result = graph.invoke(
                Command(resume=task_data["resume_value"]),
                config=config,
            )
        else:
            # New invocation
            result = graph.invoke(task_data["initial_state"], config=config)

        # Check if interrupted
        if "__interrupt__" in result:
            interrupt_payload = result["__interrupt__"][0].value
            await publish_interrupt(task_data["thread_id"], interrupt_payload)
            logger.info("agent_task_interrupted", agent=agent_type, thread_id=task_data["thread_id"])
        else:
            await publish_completion(task_data["thread_id"], result)
            logger.info("agent_task_completed", agent=agent_type, thread_id=task_data["thread_id"])

    except Exception as e:
        logger.error("agent_task_failed", agent=agent_type, thread_id=task_data["thread_id"], error=str(e))
        await publish_error(task_data["thread_id"], str(e))


async def worker_loop():
    """Main worker loop -- pulls tasks from Redis queue."""
    r = redis.from_url("redis://redis.medos.internal:6379")

    while True:
        # Blocking pop with 30s timeout
        result = await r.brpop("agent_tasks", timeout=30)
        if result:
            _, task_json = result
            task_data = json.loads(task_json)
            await process_task(task_data)


async def publish_interrupt(thread_id: str, payload: dict):
    r = redis.from_url("redis://redis.medos.internal:6379")
    await r.publish(f"agent_events:{thread_id}", json.dumps({
        "type": "interrupt",
        "thread_id": thread_id,
        "payload": payload,
    }))


async def publish_completion(thread_id: str, result: dict):
    r = redis.from_url("redis://redis.medos.internal:6379")
    await r.publish(f"agent_events:{thread_id}", json.dumps({
        "type": "completed",
        "thread_id": thread_id,
        "status": result.get("status"),
    }))


async def publish_error(thread_id: str, error: str):
    r = redis.from_url("redis://redis.medos.internal:6379")
    await r.publish(f"agent_events:{thread_id}", json.dumps({
        "type": "error",
        "thread_id": thread_id,
        "error": error,
    }))


if __name__ == "__main__":
    asyncio.run(worker_loop())
```

### Auto-Scaling Policy

```python
# terraform/modules/agent_workers/autoscaling.tf (conceptual)

SCALING_CONFIG = {
    "metric": "custom/redis_queue_depth",
    "target_value": 10,                    # Target 10 tasks per worker
    "min_capacity": 1,
    "max_capacity": 20,
    "scale_in_cooldown": 300,              # 5 minutes
    "scale_out_cooldown": 60,              # 1 minute
}
```

### Blue-Green Deployment

Agent updates follow a blue-green deployment pattern:

1. Deploy new agent worker image to the "green" ECS service.
2. Route a percentage of new tasks to the green service (canary).
3. Monitor error rates and quality metrics for 30 minutes.
4. If metrics are healthy, shift all traffic to green.
5. Keep blue service running for 1 hour as rollback target.
6. Drain and decommission blue service.

Critical: In-progress agent runs (with active checkpoints) must complete on the service version that started them. The worker process checks a `worker_version` field in the checkpoint metadata and only processes tasks that match its version or are new.

---

## 11. Agent 5: Post-Acute Guardian Agent (Theoria Medical)

This agent monitors post-acute care patients (SNF, ALF) using wearable device data (Oura Ring, Apple Watch, Withings scales, pulse oximeters) and triggers clinical alerts when physiological patterns indicate deterioration. The primary kill shot: catching CHF exacerbation before it results in a $15-25K rehospitalization. See [[ADR-007-wearable-iot-integration]] for device integration architecture and [[ADR-006-context-rehydration]] for the context retrieval pattern.

### Module Mapping

- **Module F** (Patient Engagement): RPM data collection, patient-facing alerts
- **Module B** (Provider Workflow): Clinical alert routing, care plan updates

### Purpose

Continuously monitors wearable device readings from post-acute patients across 50+ SNF/ALF facilities, detects clinically significant trends (weight gain, SpO2 drops, HRV deterioration, sleep quality changes), and generates prioritized alerts to nursing staff and attending physicians via A2A protocol.

### Trigger

Incoming FHIR `Observation` from the Device Bridge MCP Server ([[ADR-007-wearable-iot-integration]]) when a new device reading is received or when a batch of readings crosses a configured threshold.

### State Definition

```python
from typing import Annotated, Optional, Literal
from typing_extensions import TypedDict
from operator import add

class PostAcuteGuardianState(TypedDict):
    # Input
    patient_id: str
    facility_id: str
    device_readings: Annotated[list[dict], add]  # FHIR Observations from Device Bridge
    reading_window_hours: int                     # Lookback window (default: 48h)

    # Context (ADR-006 rehydration)
    clinical_context: Optional[dict]              # Rehydrated patient context
    active_conditions: list[dict]                 # Current diagnoses (CHF, COPD, DM, etc.)
    current_medications: list[dict]               # Active medication list
    baseline_vitals: Optional[dict]               # Patient-specific baselines

    # Risk Assessment
    risk_assessment: Optional[dict]               # AI-generated risk analysis
    risk_score: float                             # 0.0 - 1.0 composite risk
    alert_level: str                              # P1 (immediate) | P2 (urgent) | P3 (routine) | P4 (informational)
    trend_analysis: Optional[dict]                # Multi-day trend data

    # Action
    recommended_action: Optional[str]             # AI-generated recommendation
    recommended_medications: list[dict]            # Medication suggestions (always requires human approval)
    care_plan_updates: list[dict]                 # Suggested care plan modifications
    notification_targets: list[str]               # Provider IDs to notify

    # Workflow
    confidence: float
    status: str                                   # received | context_loaded | assessed | alerted | acknowledged
    a2a_task_id: Optional[str]                    # A2A task for nurse notification
    error: Optional[str]
```

### Graph Definition

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from typing import Literal
import json

def ingest_device_readings(state: PostAcuteGuardianState) -> dict:
    """Ingest and validate incoming device readings from Device Bridge."""
    from app.services.device_bridge import DeviceBridgeService

    bridge = DeviceBridgeService()
    validated_readings = bridge.validate_and_enrich(
        patient_id=state["patient_id"],
        readings=state["device_readings"],
        window_hours=state["reading_window_hours"],
    )
    return {
        "device_readings": validated_readings,
        "status": "received",
    }


def rehydrate_context(state: PostAcuteGuardianState) -> dict:
    """Rehydrate patient clinical context via ADR-006 Context Rehydration Engine."""
    from app.services.context import ContextRehydrationEngine

    engine = ContextRehydrationEngine()
    context = engine.rehydrate(
        patient_id=state["patient_id"],
        include=["conditions", "medications", "care_plans", "recent_encounters", "baseline_vitals"],
    )
    return {
        "clinical_context": context.summary,
        "active_conditions": context.conditions,
        "current_medications": context.medications,
        "baseline_vitals": context.baseline_vitals,
        "status": "context_loaded",
    }


def assess_risk(state: PostAcuteGuardianState) -> dict:
    """AI-powered risk assessment combining device trends with clinical context."""
    readings_summary = json.dumps(state["device_readings"][-20:], indent=2)  # Last 20 readings
    conditions = json.dumps(state["active_conditions"], indent=2)
    meds = json.dumps(state["current_medications"], indent=2)
    baselines = json.dumps(state["baseline_vitals"], indent=2)

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system="""You are a post-acute care clinical decision support system. Analyze device
readings against the patient's clinical context and baseline vitals to assess deterioration risk.

ASSESSMENT CRITERIA:
- Weight gain > 2 lbs in 24h or > 3 lbs in 48h in CHF patients = HIGH RISK
- SpO2 < 92% sustained for > 30 min in COPD/CHF patients = HIGH RISK
- HRV decrease > 20% from baseline = MODERATE RISK
- Sleep quality score drop > 15% over 3 days = MODERATE RISK
- Blood glucose > 300 mg/dL or < 70 mg/dL = HIGH RISK (DM patients)
- Resting HR > 110 bpm sustained = MODERATE RISK
- Temperature > 101.0F = MODERATE RISK (infection risk in SNF)

Return JSON:
{
  "risk_score": float (0.0-1.0),
  "alert_level": "P1" | "P2" | "P3" | "P4",
  "risk_factors": [{"factor": str, "severity": str, "value": str, "baseline": str}],
  "trend_analysis": {"direction": str, "days_declining": int, "key_indicators": [str]},
  "recommended_action": str,
  "recommended_medications": [{"medication": str, "dose": str, "rationale": str}],
  "care_plan_updates": [{"action": str, "urgency": str}],
  "confidence": float,
  "clinical_reasoning": str
}

CRITICAL RULES:
- DO NOT diagnose. You are identifying risk patterns, not making diagnoses.
- DO NOT prescribe. Medication suggestions require physician approval.
- Factor in current medications when assessing readings (e.g., beta-blockers affect HR).
- Consider facility-specific context (SNF patients have different baselines than community-dwelling).""",
        messages=[{
            "role": "user",
            "content": f"""Patient Conditions:\n{conditions}\n\nCurrent Medications:\n{meds}\n\nBaseline Vitals:\n{baselines}\n\nDevice Readings (last 48h):\n{readings_summary}"""
        }],
    )

    assessment = json.loads(response.content[0].text)
    return {
        "risk_assessment": assessment,
        "risk_score": assessment["risk_score"],
        "alert_level": assessment["alert_level"],
        "trend_analysis": assessment.get("trend_analysis"),
        "recommended_action": assessment["recommended_action"],
        "recommended_medications": assessment.get("recommended_medications", []),
        "care_plan_updates": assessment.get("care_plan_updates", []),
        "confidence": assessment["confidence"],
        "status": "assessed",
    }


def route_by_alert_level(state: PostAcuteGuardianState) -> Literal["p1_immediate", "p2_urgent", "p3_routine", "p4_log_only"]:
    """Route based on alert severity level."""
    routing = {
        "P1": "p1_immediate",
        "P2": "p2_urgent",
        "P3": "p3_routine",
        "P4": "p4_log_only",
    }
    return routing.get(state["alert_level"], "p4_log_only")


def p1_immediate(state: PostAcuteGuardianState) -> dict:
    """P1: Immediate action required -- interrupt for physician review."""
    decision = interrupt({
        "type": "post_acute_p1_alert",
        "patient_id": state["patient_id"],
        "facility_id": state["facility_id"],
        "alert_level": "P1",
        "risk_score": state["risk_score"],
        "risk_assessment": state["risk_assessment"],
        "recommended_action": state["recommended_action"],
        "recommended_medications": state["recommended_medications"],
        "message": "IMMEDIATE: Patient deterioration detected. Physician review required.",
    })
    return {
        "status": "acknowledged",
        "notification_targets": [decision.get("acknowledged_by")],
    }


def p2_urgent(state: PostAcuteGuardianState) -> dict:
    """P2: Urgent -- send A2A alert to nursing staff + notify physician."""
    from app.services.a2a import A2AClient

    a2a = A2AClient()
    task_id = a2a.send_message(
        target_agent="patient-communication",
        task_type="clinical_alert",
        payload={
            "patient_id": state["patient_id"],
            "facility_id": state["facility_id"],
            "alert_level": "P2",
            "recommended_action": state["recommended_action"],
            "risk_assessment": state["risk_assessment"],
        },
    )

    # Also interrupt for medication changes
    if state["recommended_medications"]:
        decision = interrupt({
            "type": "post_acute_p2_medication_review",
            "patient_id": state["patient_id"],
            "recommended_medications": state["recommended_medications"],
            "risk_assessment": state["risk_assessment"],
            "message": "Medication change recommended. Physician approval required.",
        })
        return {
            "status": "acknowledged",
            "a2a_task_id": task_id,
            "notification_targets": [decision.get("acknowledged_by")],
        }

    return {"status": "alerted", "a2a_task_id": task_id}


def p3_routine(state: PostAcuteGuardianState) -> dict:
    """P3: Routine -- log alert and add to provider's next-login summary."""
    from app.services.notifications import NotificationService

    notifications = NotificationService()
    notifications.queue_for_next_login(
        facility_id=state["facility_id"],
        alert_type="device_trend",
        patient_id=state["patient_id"],
        summary=state["recommended_action"],
    )
    return {"status": "alerted"}


def p4_log_only(state: PostAcuteGuardianState) -> dict:
    """P4: Informational -- log for trend analysis, no immediate action."""
    return {"status": "logged"}


def write_fhir_risk_assessment(state: PostAcuteGuardianState) -> dict:
    """Persist risk assessment as FHIR RiskAssessment resource."""
    from app.services.fhir import FHIRService

    fhir = FHIRService()
    fhir.create_risk_assessment(
        patient_id=state["patient_id"],
        risk_score=state["risk_score"],
        method="MedOS Post-Acute Guardian Agent v1.0",
        basis_observations=[r.get("id") for r in state["device_readings"]],
        prediction=state["recommended_action"],
    )
    return {"status": "persisted"}


# --- Build graph ---

guardian_builder = StateGraph(PostAcuteGuardianState)

guardian_builder.add_node("ingest_device_readings", ingest_device_readings)
guardian_builder.add_node("rehydrate_context", rehydrate_context)
guardian_builder.add_node("assess_risk", assess_risk)
guardian_builder.add_node("p1_immediate", p1_immediate)
guardian_builder.add_node("p2_urgent", p2_urgent)
guardian_builder.add_node("p3_routine", p3_routine)
guardian_builder.add_node("p4_log_only", p4_log_only)
guardian_builder.add_node("write_fhir_risk_assessment", write_fhir_risk_assessment)

guardian_builder.add_edge(START, "ingest_device_readings")
guardian_builder.add_edge("ingest_device_readings", "rehydrate_context")
guardian_builder.add_edge("rehydrate_context", "assess_risk")
guardian_builder.add_conditional_edges("assess_risk", route_by_alert_level, {
    "p1_immediate": "p1_immediate",
    "p2_urgent": "p2_urgent",
    "p3_routine": "p3_routine",
    "p4_log_only": "p4_log_only",
})
guardian_builder.add_edge("p1_immediate", "write_fhir_risk_assessment")
guardian_builder.add_edge("p2_urgent", "write_fhir_risk_assessment")
guardian_builder.add_edge("p3_routine", "write_fhir_risk_assessment")
guardian_builder.add_edge("p4_log_only", "write_fhir_risk_assessment")
guardian_builder.add_edge("write_fhir_risk_assessment", END)

guardian_graph = guardian_builder.compile(checkpointer=checkpointer)
```

### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| P3-P4 alert generation | >= 0.85 | 0.70-0.85 | < 0.70 |
| P1-P2 alert generation | >= 0.90 | 0.80-0.90 | < 0.80 |
| Medication recommendations | N/A | N/A | ALWAYS human review |
| Care plan modifications | N/A | N/A | ALWAYS human review |

### Human-in-the-Loop Gates

- **All P1 alerts** require physician acknowledgment before resolution
- **All medication changes** require physician or pharmacist approval
- **Care plan updates** require attending physician sign-off
- **Transfer recommendations** (SNF to hospital) require physician order

### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, facility assignment |
| Read | `Observation` | Device readings (weight, SpO2, HR, HRV, sleep) |
| Read | `Device` | Registered wearable device details |
| Read | `DeviceMetric` | Device measurement capabilities |
| Read | `Condition` | Active conditions (CHF, COPD, DM) |
| Read | `MedicationRequest` | Current medications affecting vital baselines |
| Read | `CarePlan` | Active care plan for care plan update context |
| Write | `RiskAssessment` | Generated risk assessment with score |
| Write | `CommunicationRequest` | Alert sent to nursing/physician |
| Write | `Flag` | Patient flag for elevated risk |

### Revenue / Clinical Impact

- **Prevents rehospitalizations**: Each avoided SNF-to-hospital transfer saves $15-25K
- **Protects ACO REACH shared savings**: Rehospitalization penalties directly reduce Empassion Health distributions
- **RPM billing**: Continuous monitoring generates CPT 99457/99458 revenue ($50-80/patient/month)
- **Clinical impact**: Earlier intervention = better patient outcomes, lower mortality in post-acute CHF patients

---

## 12. Agent 6: CCM Revenue Agent (Theoria Medical)

This agent tracks Chronic Care Management (CCM) activities across Theoria's patient population and automatically generates CPT 99490/99491 claims when the 20-minute or 40-minute monthly threshold is crossed. CCM billing is the single largest source of uncaptured revenue in post-acute care -- most practices leave $60-150/patient/month on the table because they cannot track care coordination time accurately. See [[Revenue-Cycle-Deep-Dive]] for CCM billing rules.

### Module Mapping

- **Module C** (Revenue Cycle): Claim generation, CPT code assignment
- **Module F** (Patient Engagement): Activity tracking, care coordination logging

### Purpose

Continuously tracks qualifying CCM activities (phone calls, care plan reviews, medication reconciliation, lab result discussions, care coordination with specialists) per patient per calendar month. When the 20-minute threshold is reached, the agent generates a claim draft and routes it to the billing team. At 40 minutes, it upgrades to the higher-value CPT 99491 code.

### Trigger

EventBridge events from patient chart API actions: `patient.communication.logged`, `careplan.reviewed`, `medication.reconciled`, `lab.discussed`, `referral.coordinated`.

### State Definition

```python
class CCMRevenueState(TypedDict):
    # Input
    patient_id: str
    calendar_month: str                          # "2026-03" format
    provider_id: str

    # Activity Tracking
    activities: Annotated[list[dict], add]        # [{"timestamp": str, "type": str, "duration_minutes": int, "provider_id": str, "description": str}]
    total_minutes_month: int                      # Running total for current month
    previous_activities_count: int                # Activities already logged before this run

    # Billing
    billing_threshold_crossed: bool               # True when >= 20 minutes
    cpt_codes_earned: list[dict]                  # [{"code": "99490", "description": str, "minutes": int}]
    claim_draft: Optional[dict]                   # Structured claim ready for review
    attestation_required: bool                    # True if claim > $500/month

    # Compliance
    consent_verified: bool                        # CCM requires patient consent on file
    care_plan_active: bool                        # Active care plan required for CCM billing
    qualifying_conditions: list[dict]             # 2+ chronic conditions required

    # Workflow
    confidence: float
    status: str                                   # tracking | threshold_crossed | claim_drafted | submitted
    error: Optional[str]
```

### Graph Definition

```python
def aggregate_activities(state: CCMRevenueState) -> dict:
    """Aggregate CCM-qualifying activities for the calendar month."""
    from app.services.ccm import CCMTracker

    tracker = CCMTracker()
    monthly_summary = tracker.get_monthly_summary(
        patient_id=state["patient_id"],
        month=state["calendar_month"],
    )
    return {
        "activities": monthly_summary.activities,
        "total_minutes_month": monthly_summary.total_minutes,
        "previous_activities_count": monthly_summary.previous_count,
        "status": "tracking",
    }


def classify_activity(state: CCMRevenueState) -> dict:
    """AI classification of whether activities qualify for CCM billing."""
    # Only classify new activities (not previously processed)
    new_activities = state["activities"][state["previous_activities_count"]:]
    if not new_activities:
        return {"status": "no_new_activities"}

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system="""You are a CCM billing compliance specialist. Classify each activity as
qualifying or non-qualifying for CPT 99490 (Chronic Care Management).

QUALIFYING ACTIVITIES (per CMS):
- Care plan creation, review, or revision
- Communication with patient/caregiver about care plan
- Medication management and reconciliation
- Coordination with other treating providers
- Lab/test result review and patient communication
- Referral management and follow-up
- Health education related to chronic conditions

NON-QUALIFYING ACTIVITIES:
- Administrative scheduling (appointment reminders without clinical content)
- Billing inquiries
- Activities already billed under another E/M code for same date
- Activities < 1 minute duration

Return JSON:
{
  "classified_activities": [{"index": int, "qualifies": bool, "rationale": str, "adjusted_minutes": int}],
  "total_qualifying_minutes": int,
  "confidence": float
}""",
        messages=[{
            "role": "user",
            "content": f"Activities to classify:\n{json.dumps(new_activities, indent=2)}"
        }],
    )

    classification = json.loads(response.content[0].text)
    return {
        "total_minutes_month": state["total_minutes_month"] + classification["total_qualifying_minutes"],
        "confidence": classification["confidence"],
        "status": "classified",
    }


def check_billing_eligibility(state: CCMRevenueState) -> dict:
    """Verify CCM billing prerequisites: consent, care plan, 2+ chronic conditions."""
    from app.services.fhir import FHIRService

    fhir = FHIRService()

    # Check for active care plan
    care_plans = fhir.search("CarePlan", {
        "patient": state["patient_id"],
        "status": "active",
        "category": "http://hl7.org/fhir/us/core/CodeSystem/careplan-category|assess-plan",
    })

    # Check for 2+ qualifying chronic conditions
    conditions = fhir.search("Condition", {
        "patient": state["patient_id"],
        "clinical-status": "active",
        "category": "problem-list-item",
    })
    chronic_conditions = [c for c in conditions if _is_chronic(c)]

    # Check consent
    consents = fhir.search("Consent", {
        "patient": state["patient_id"],
        "category": "ccm-consent",
        "status": "active",
    })

    return {
        "consent_verified": len(consents) > 0,
        "care_plan_active": len(care_plans) > 0,
        "qualifying_conditions": chronic_conditions[:5],  # Cap at 5 for state size
        "status": "eligibility_checked",
    }


def route_by_threshold(state: CCMRevenueState) -> Literal["generate_claim", "continue_tracking"]:
    """Route based on whether billing threshold is crossed."""
    if not state["consent_verified"] or not state["care_plan_active"]:
        return "continue_tracking"
    if len(state["qualifying_conditions"]) < 2:
        return "continue_tracking"
    if state["total_minutes_month"] >= 20:
        return "generate_claim"
    return "continue_tracking"


def generate_claim(state: CCMRevenueState) -> dict:
    """Generate CCM claim draft with appropriate CPT code."""
    minutes = state["total_minutes_month"]

    cpt_codes = []
    if minutes >= 60:
        cpt_codes.append({"code": "99491", "description": "CCM 60+ minutes", "minutes": 60})
        remaining = minutes - 60
        while remaining >= 30:
            cpt_codes.append({"code": "99437", "description": "CCM each additional 30 min", "minutes": 30})
            remaining -= 30
    elif minutes >= 40:
        cpt_codes.append({"code": "99491", "description": "CCM 40+ minutes clinical staff", "minutes": 40})
    elif minutes >= 20:
        cpt_codes.append({"code": "99490", "description": "CCM 20+ minutes", "minutes": 20})

    claim_draft = {
        "patient_id": state["patient_id"],
        "provider_id": state["provider_id"],
        "service_period": state["calendar_month"],
        "cpt_codes": cpt_codes,
        "qualifying_conditions": [c.get("code", {}).get("coding", [{}])[0].get("code", "") for c in state["qualifying_conditions"]],
        "total_minutes": minutes,
        "activity_count": len(state["activities"]),
    }

    return {
        "billing_threshold_crossed": True,
        "cpt_codes_earned": cpt_codes,
        "claim_draft": claim_draft,
        "attestation_required": sum(c.get("minutes", 0) for c in cpt_codes) > 60,
        "status": "claim_drafted",
    }


def continue_tracking(state: CCMRevenueState) -> dict:
    """Not enough minutes yet -- continue tracking."""
    return {"billing_threshold_crossed": False, "status": "tracking"}


def billing_review(state: CCMRevenueState) -> Command[Literal["submit_claim", "generate_claim"]]:
    """Human review of CCM claim before submission."""
    decision = interrupt({
        "type": "ccm_claim_review",
        "patient_id": state["patient_id"],
        "claim_draft": state["claim_draft"],
        "total_minutes": state["total_minutes_month"],
        "cpt_codes": state["cpt_codes_earned"],
        "activities": state["activities"][-10:],  # Last 10 activities for context
        "message": f"CCM claim ready: {state['total_minutes_month']} minutes, CPT {state['cpt_codes_earned'][0]['code']}",
    })

    if decision.get("approved"):
        return Command(goto="submit_claim", update={"status": "approved"})
    else:
        return Command(goto="generate_claim", update={"status": "revision_requested"})


def submit_claim(state: CCMRevenueState) -> dict:
    """Submit the CCM claim via the Revenue Cycle pipeline."""
    from app.services.claims import ClaimsService

    claims = ClaimsService()
    result = claims.submit_ccm_claim(state["claim_draft"])
    return {"status": "submitted"}


# --- Build graph ---

ccm_builder = StateGraph(CCMRevenueState)

ccm_builder.add_node("aggregate_activities", aggregate_activities)
ccm_builder.add_node("classify_activity", classify_activity)
ccm_builder.add_node("check_billing_eligibility", check_billing_eligibility)
ccm_builder.add_node("generate_claim", generate_claim)
ccm_builder.add_node("continue_tracking", continue_tracking)
ccm_builder.add_node("billing_review", billing_review)
ccm_builder.add_node("submit_claim", submit_claim)

ccm_builder.add_edge(START, "aggregate_activities")
ccm_builder.add_edge("aggregate_activities", "classify_activity")
ccm_builder.add_edge("classify_activity", "check_billing_eligibility")
ccm_builder.add_conditional_edges("check_billing_eligibility", route_by_threshold, {
    "generate_claim": "generate_claim",
    "continue_tracking": "continue_tracking",
})
ccm_builder.add_edge("generate_claim", "billing_review")
# billing_review routes via Command(goto=...)
ccm_builder.add_edge("submit_claim", END)
ccm_builder.add_edge("continue_tracking", END)

ccm_graph = ccm_builder.compile(checkpointer=checkpointer)
```

### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Activity classification | >= 0.90 | 0.80-0.90 | < 0.80 |
| Billing eligibility check | >= 0.99 (rule-based) | N/A | Data quality issues |
| Claim generation | N/A | N/A | ALWAYS human review |

### Human-in-the-Loop Gates

- **All claims** require billing staff review before submission
- **Claims > $500/month** require physician attestation
- **First-time CCM patients** require consent verification review

### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, eligibility |
| Read | `CarePlan` | Active care plan (CCM prerequisite) |
| Read | `Condition` | Chronic conditions (need 2+ for CCM) |
| Read | `Encounter` | Recent encounters for time exclusion |
| Read | `Communication` | Logged care coordination activities |
| Read | `Consent` | CCM consent on file |
| Write | `Claim` | Generated CCM claim |
| Write | `Communication` | Activity log entries |

### Revenue Impact

- **Captured revenue**: $60-150/patient/month in CCM billing (CPT 99490: ~$42, 99491: ~$83)
- **Scale**: With 5,000+ attributed lives across Theoria's network, potential capture of $300K-750K/month
- **Current gap**: Estimated 70%+ of qualifying CCM time goes unbilled due to manual tracking burden

---

## 13. Agent 7: Shift Summary Agent (Theoria Medical)

This agent generates structured handoff briefings when providers transition between shifts at post-acute facilities. Designed for Theoria's distributed model where physicians cover 50+ SNFs/ALFs. Solves information loss at shift boundaries -- the leading cause of adverse events in post-acute care.

### Module Mapping

- **Module B** (Provider Workflow): Shift handoff, patient prioritization, care continuity

### Purpose

When an outgoing provider logs off (or an incoming provider logs on), the agent scans all active patients across assigned facilities, retrieves each patient's recent context via [[ADR-006-context-rehydration]], priority-ranks them by acuity and pending actions, and generates a structured shift briefing. The incoming provider sees a prioritized list with P1 patients requiring immediate attention at the top.

### Trigger

- Provider login event (new shift starting)
- Explicit handoff request from outgoing provider
- Scheduled shift transition (configurable per facility)

### State Definition

```python
class ShiftSummaryState(TypedDict):
    # Input
    outgoing_provider: Optional[str]              # Practitioner ID logging off
    incoming_provider: str                        # Practitioner ID logging on
    shift_id: str                                 # Unique shift identifier
    facility_list: list[str]                      # Facility IDs this provider covers

    # Patient Data
    patient_summaries: Annotated[list[dict], add] # [{patient_id, name, facility, priority, summary, pending_items, vitals}]
    total_patients: int
    p1_count: int                                 # Immediate attention needed
    p2_count: int                                 # Urgent within 1 hour

    # Handoff Content
    pending_items: list[dict]                     # [{patient_id, item, urgency, source}]
    vitals_summary: Optional[dict]                # Aggregated vitals overview
    handoff_narrative: Optional[str]              # AI-generated shift briefing
    critical_alerts: list[dict]                   # Unacknowledged alerts from Guardian Agent

    # Workflow
    confidence: float
    status: str                                   # scanning | summarizing | ready | delivered
    error: Optional[str]
```

### Graph Definition

```python
def scan_active_patients(state: ShiftSummaryState) -> dict:
    """Scan all active patients across assigned facilities."""
    from app.services.fhir import FHIRService

    fhir = FHIRService()
    all_patients = []

    for facility_id in state["facility_list"]:
        patients = fhir.search("Encounter", {
            "location": f"Location/{facility_id}",
            "status": "in-progress",
            "_include": "Encounter:subject",
        })
        all_patients.extend(patients)

    return {
        "total_patients": len(all_patients),
        "patient_summaries": [{"patient_id": p["id"], "facility_id": p.get("facility_id")} for p in all_patients],
        "status": "scanning",
    }


def rehydrate_all_patients(state: ShiftSummaryState) -> dict:
    """Batch rehydrate context for all active patients."""
    from app.services.context import ContextRehydrationEngine

    engine = ContextRehydrationEngine()
    enriched_summaries = []
    pending_items = []
    critical_alerts = []

    for patient_stub in state["patient_summaries"]:
        context = engine.rehydrate(
            patient_id=patient_stub["patient_id"],
            include=["conditions", "recent_vitals", "pending_orders", "alerts", "medications"],
            lightweight=True,
        )

        enriched_summaries.append({
            **patient_stub,
            "conditions": context.conditions_summary,
            "recent_vitals": context.recent_vitals,
            "pending_orders": context.pending_orders,
            "last_seen": context.last_encounter_time,
            "medication_changes_24h": context.recent_med_changes,
        })

        pending_items.extend(context.pending_orders)
        critical_alerts.extend(context.unacknowledged_alerts)

    return {
        "patient_summaries": enriched_summaries,
        "pending_items": pending_items,
        "critical_alerts": critical_alerts,
        "status": "context_loaded",
    }


def prioritize_and_generate_briefing(state: ShiftSummaryState) -> dict:
    """AI-powered priority ranking and shift briefing generation."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8192,
        system="""You are a clinical handoff assistant for post-acute care facilities (SNFs/ALFs).
Generate a structured shift briefing from the patient data provided.

PRIORITIZATION CRITERIA (in order):
P1 (IMMEDIATE): Acute change in condition, unacknowledged critical alerts, unstable vitals
P2 (URGENT): Pending stat orders, medication changes in last 24h, family meeting needed
P3 (ROUTINE): Scheduled assessments, stable patients with active care plans
P4 (INFORMATIONAL): Stable, no pending items, routine monitoring only

OUTPUT FORMAT:
{
  "briefing_sections": [
    {
      "priority": "P1" | "P2" | "P3" | "P4",
      "patients": [
        {
          "patient_id": str,
          "one_liner": str,
          "key_concerns": [str],
          "pending_actions": [str],
          "vitals_trend": str
        }
      ]
    }
  ],
  "shift_narrative": str,
  "p1_count": int,
  "p2_count": int,
  "confidence": float
}

RULES:
- P1 patients MUST be listed first with clear action items
- Use concise clinical language (not full sentences)
- Include vital sign trends, not just current values
- Flag patients with >3 medications changed in 24h
- Do NOT include patient names -- use patient_id only
- Keep the shift narrative under 500 words""",
        messages=[{
            "role": "user",
            "content": f"""Patients ({state['total_patients']} total):\n{json.dumps(state['patient_summaries'], indent=2)}\n\nCritical Alerts:\n{json.dumps(state['critical_alerts'], indent=2)}\n\nPending Items:\n{json.dumps(state['pending_items'], indent=2)}"""
        }],
    )

    briefing = json.loads(response.content[0].text)
    return {
        "handoff_narrative": briefing["shift_narrative"],
        "p1_count": briefing["p1_count"],
        "p2_count": briefing["p2_count"],
        "confidence": briefing["confidence"],
        "patient_summaries": briefing["briefing_sections"],
        "status": "ready",
    }


def deliver_briefing(state: ShiftSummaryState) -> dict:
    """Deliver the shift briefing to the incoming provider."""
    from app.services.notifications import NotificationService

    notifications = NotificationService()
    notifications.deliver_shift_briefing(
        provider_id=state["incoming_provider"],
        shift_id=state["shift_id"],
        briefing={
            "narrative": state["handoff_narrative"],
            "patient_summaries": state["patient_summaries"],
            "p1_count": state["p1_count"],
            "p2_count": state["p2_count"],
            "critical_alerts": state["critical_alerts"],
        },
    )
    return {"status": "delivered"}


# --- Build graph ---

shift_builder = StateGraph(ShiftSummaryState)

shift_builder.add_node("scan_active_patients", scan_active_patients)
shift_builder.add_node("rehydrate_all_patients", rehydrate_all_patients)
shift_builder.add_node("prioritize_and_generate_briefing", prioritize_and_generate_briefing)
shift_builder.add_node("deliver_briefing", deliver_briefing)

shift_builder.add_edge(START, "scan_active_patients")
shift_builder.add_edge("scan_active_patients", "rehydrate_all_patients")
shift_builder.add_edge("rehydrate_all_patients", "prioritize_and_generate_briefing")
shift_builder.add_edge("prioritize_and_generate_briefing", "deliver_briefing")
shift_builder.add_edge("deliver_briefing", END)

shift_graph = shift_builder.compile(checkpointer=checkpointer)
```

### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Priority ranking | >= 0.85 | 0.70-0.85 | < 0.70 |
| Clinical narrative accuracy | >= 0.90 | 0.80-0.90 | < 0.80 |
| P1 patient identification | N/A | N/A | ALWAYS flag for immediate review |

### Human-in-the-Loop Gates

- **P1 patients** always flagged for immediate provider review upon login
- **Conflicting information** between outgoing provider notes and system data triggers review
- **No fully autonomous actions** -- this agent is informational only

### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, facility assignment |
| Read | `Encounter` | Active encounters across facilities |
| Read | `Observation` | Recent vitals, device readings |
| Read | `MedicationRequest` | Current medications, recent changes |
| Read | `DiagnosticReport` | Pending or recent lab/imaging results |
| Read | `ServiceRequest` | Pending orders |
| Read | `Flag` | Active patient flags (fall risk, isolation, etc.) |
| Write | `Communication` | Shift handoff record (audit trail) |

### Clinical Impact

- **Prevents information loss**: Shift handoff errors cause 80% of serious medical errors in SNFs
- **Saves provider time**: Eliminates 30-45 minutes of manual chart review per shift start
- **Scales across 50+ facilities**: Single provider covering multiple SNFs gets unified view

---

## 14. Agent 8: ACO REACH Quality Agent (Theoria Medical)

This agent continuously monitors Theoria's attributed patient population under ACO REACH (via Empassion Health) for care gaps, quality measure compliance, and shared savings optimization. See [[Population-Health-Analytics]] for measure definitions (HEDIS, CMS Stars, MIPS).

### Module Mapping

- **Module E** (Population Health & Analytics): Care gap identification, quality measure calculation, shared savings projection

### Purpose

Scans the attributed patient population daily to identify care gaps (overdue HbA1c, missing blood pressure readings, advance care planning not documented, etc.), calculates the revenue impact of closing each gap, and triggers automated patient outreach via A2A to the Patient Communication Agent. Directly impacts Empassion Health ACO REACH shared savings distributions.

### Trigger

- Scheduled population scan (every 24 hours at 2:00 AM ET)
- Real-time gap detection when new clinical data arrives (FHIR Subscription webhook)
- On-demand quality dashboard refresh

### State Definition

```python
class ACOQualityState(TypedDict):
    # Input
    scan_type: str                                # scheduled | realtime | on_demand
    population_filter: Optional[dict]             # Filter criteria (facility, condition, payer)

    # Population
    attributed_lives: list[dict]                  # [{patient_id, facility_id, risk_score, conditions}]
    total_attributed: int

    # Gap Analysis
    care_gaps: Annotated[list[dict], add]          # [{patient_id, measure_id, gap_type, days_overdue, revenue_impact, priority}]
    gaps_by_measure: dict                          # {measure_id: count}
    total_gaps: int
    high_impact_gaps: int                          # Gaps with revenue_impact > $100

    # Outreach
    outreach_queue: list[dict]                     # [{patient_id, gap_type, channel, message_template}]
    outreach_initiated: int

    # Quality Scores
    quality_scores: Optional[dict]                 # {measure_id: {numerator, denominator, rate, benchmark}}
    shared_savings_projection: Optional[dict]      # {current_quality_score, projected_savings, gap_closure_impact}

    # Workflow
    confidence: float
    status: str                                    # scanning | gaps_identified | outreach_queued | complete
    error: Optional[str]
```

### Graph Definition

```python
def scan_population(state: ACOQualityState) -> dict:
    """Scan attributed patient population for care gaps."""
    from app.services.population import PopulationService

    pop = PopulationService()
    attributed = pop.get_attributed_lives(
        filter_criteria=state.get("population_filter"),
    )
    return {
        "attributed_lives": attributed,
        "total_attributed": len(attributed),
        "status": "scanning",
    }


def identify_care_gaps(state: ACOQualityState) -> dict:
    """Rule-based gap identification against CMS/HEDIS quality measures."""
    from app.services.quality import QualityMeasureEngine

    engine = QualityMeasureEngine()
    all_gaps = []
    gaps_by_measure = {}

    for patient in state["attributed_lives"]:
        patient_gaps = engine.identify_gaps(
            patient_id=patient["patient_id"],
            measures=["HbA1c_control", "BP_control", "ACP_documented", "depression_screening",
                       "fall_risk_assessment", "medication_reconciliation", "annual_wellness"],
        )
        for gap in patient_gaps:
            gap["revenue_impact"] = engine.estimate_revenue_impact(gap)
            all_gaps.append(gap)
            gaps_by_measure[gap["measure_id"]] = gaps_by_measure.get(gap["measure_id"], 0) + 1

    return {
        "care_gaps": all_gaps,
        "gaps_by_measure": gaps_by_measure,
        "total_gaps": len(all_gaps),
        "high_impact_gaps": sum(1 for g in all_gaps if g.get("revenue_impact", 0) > 100),
        "status": "gaps_identified",
    }


def prioritize_and_project(state: ACOQualityState) -> dict:
    """AI-powered gap prioritization and shared savings projection."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system="""You are an ACO REACH quality analytics expert. Analyze the care gaps identified
and prioritize them by combined clinical urgency and revenue impact.

PRIORITIZATION FRAMEWORK:
1. Clinical urgency (overdue by how long, patient risk score)
2. Revenue impact (shared savings impact per gap closure)
3. Outreach feasibility (is the patient reachable, are they engaged)
4. Measure weight (CMS quality measure weights for ACO REACH)

SHARED SAVINGS PROJECTION:
- Calculate current quality score based on measure rates
- Project quality score if top 20% of gaps are closed
- Estimate shared savings impact (ACO REACH uses quality multiplier on shared savings)

Return JSON:
{
  "prioritized_gaps": [{"patient_id": str, "measure_id": str, "priority_rank": int, "outreach_recommended": bool, "channel": str}],
  "quality_scores": {"measure_id": {"numerator": int, "denominator": int, "rate": float, "benchmark": float}},
  "shared_savings_projection": {"current_score": float, "projected_score": float, "estimated_savings_delta": float},
  "confidence": float
}""",
        messages=[{
            "role": "user",
            "content": f"""Population: {state['total_attributed']} attributed lives\n\nCare Gaps ({state['total_gaps']} total):\n{json.dumps(state['care_gaps'][:100], indent=2)}\n\nGaps by Measure:\n{json.dumps(state['gaps_by_measure'], indent=2)}"""
        }],
    )

    projection = json.loads(response.content[0].text)
    return {
        "quality_scores": projection["quality_scores"],
        "shared_savings_projection": projection["shared_savings_projection"],
        "confidence": projection["confidence"],
        "status": "projected",
    }


def queue_outreach(state: ACOQualityState) -> dict:
    """Queue patient outreach via A2A to Patient Communication Agent."""
    from app.services.a2a import A2AClient

    a2a = A2AClient()
    outreach_queue = []
    initiated = 0

    for gap in state["care_gaps"][:50]:  # Top 50 gaps per run
        if gap.get("outreach_recommended", True):
            task_id = a2a.send_message(
                target_agent="patient-communication",
                task_type="care_gap_outreach",
                payload={
                    "patient_id": gap["patient_id"],
                    "gap_type": gap["measure_id"],
                    "message_type": "care_gap_reminder",
                    "urgency": "normal",
                },
            )
            outreach_queue.append({
                "patient_id": gap["patient_id"],
                "gap_type": gap["measure_id"],
                "a2a_task_id": task_id,
            })
            initiated += 1

    return {
        "outreach_queue": outreach_queue,
        "outreach_initiated": initiated,
        "status": "outreach_queued",
    }


def write_measure_reports(state: ACOQualityState) -> dict:
    """Persist quality measure results as FHIR MeasureReport resources."""
    from app.services.fhir import FHIRService

    fhir = FHIRService()
    for measure_id, scores in (state.get("quality_scores") or {}).items():
        fhir.create_measure_report(
            measure_id=measure_id,
            period=state.get("scan_type", "scheduled"),
            numerator=scores.get("numerator", 0),
            denominator=scores.get("denominator", 0),
            rate=scores.get("rate", 0.0),
        )
    return {"status": "complete"}


# --- Build graph ---

aco_builder = StateGraph(ACOQualityState)

aco_builder.add_node("scan_population", scan_population)
aco_builder.add_node("identify_care_gaps", identify_care_gaps)
aco_builder.add_node("prioritize_and_project", prioritize_and_project)
aco_builder.add_node("queue_outreach", queue_outreach)
aco_builder.add_node("write_measure_reports", write_measure_reports)

aco_builder.add_edge(START, "scan_population")
aco_builder.add_edge("scan_population", "identify_care_gaps")
aco_builder.add_edge("identify_care_gaps", "prioritize_and_project")
aco_builder.add_edge("prioritize_and_project", "queue_outreach")
aco_builder.add_edge("queue_outreach", "write_measure_reports")
aco_builder.add_edge("write_measure_reports", END)

aco_quality_graph = aco_builder.compile(checkpointer=checkpointer)
```

### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Gap identification | >= 0.90 (rule-based from HEDIS/CMS) | 0.80-0.90 | < 0.80 |
| Outreach prioritization | >= 0.85 | 0.70-0.85 | < 0.70 |
| Quality score calculation | >= 0.99 (deterministic) | N/A | Data quality issues |
| Clinical recommendations | N/A | N/A | ALWAYS requires provider sign-off |

### Human-in-the-Loop Gates

- **Clinical recommendations** within outreach messages require provider sign-off
- **Quality reports** for external submission (CMS) require practice manager approval
- **Outreach to patients** with active do-not-contact flags requires manual override

### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Attribution, demographics, risk scores |
| Read | `Observation` | Lab results, vitals, screening results |
| Read | `Condition` | Chronic conditions for HCC risk adjustment |
| Read | `Procedure` | Completed screenings and interventions |
| Read | `Immunization` | Vaccination history |
| Write | `MeasureReport` | Calculated quality measures |
| Write | `DetectedIssue` | Identified care gaps |
| Write | `CommunicationRequest` | Outreach request to Patient Communication Agent |

### Revenue Impact

- **Shared savings**: ACO REACH quality multiplier directly impacts shared savings distribution from Empassion Health
- **Gap closure revenue**: Each closed gap contributes to improved quality scores and higher shared savings rate
- **HCC capture**: Proper diagnosis documentation increases risk-adjusted payments

---

## 15. Agent 9: SNF-to-Hospital Semantic Data Bridge (Theoria Medical)

This agent solves Dr. Di Rezze's founding insight: the dangerous data void that exists when patients transition between post-acute care facilities and hospitals. When a patient is discharged from a hospital back to their SNF, this agent ingests the discharge summary, reconciles it against the SNF's existing record (ChartEasy), and generates a discrepancy report highlighting new medications, changed dosages, new diagnoses, and required follow-ups. See [[ADR-001-fhir-native-data-model]] for FHIR storage and [[ADR-008-a2a-agent-communication]] for inter-agent coordination.

### Module Mapping

- **Module H** (Integration): Cross-system data reconciliation, FHIR endpoint interoperability
- **Module A** (Patient Identity): Patient matching across disparate systems

### Purpose

Automatically ingests hospital discharge summaries (structured FHIR or unstructured PDF/text), performs semantic reconciliation against the patient's pre-hospitalization record, and generates a structured discrepancy report for the attending provider. Every medication change, new diagnosis, and follow-up instruction is categorized by severity and presented for review.

### Trigger

- ADT (Admit/Discharge/Transfer) event indicating patient discharge from hospital back to SNF
- Manual upload of discharge summary document
- FHIR Subscription notification from hospital FHIR endpoint

### State Definition

```python
class SNFBridgeState(TypedDict):
    # Input
    patient_id: str
    hospital_source: str                          # Hospital system identifier
    snf_destination: str                          # SNF facility ID
    discharge_event_id: str                       # ADT event reference

    # Discharge Data
    discharge_summary_raw: Optional[str]          # Raw text (from PDF/OCR or FHIR DocumentReference)
    discharge_summary_structured: Optional[dict]  # Parsed structured data
    discharge_medications: list[dict]             # Medications at discharge
    discharge_diagnoses: list[dict]               # Diagnoses at discharge
    discharge_followups: list[dict]               # Follow-up instructions

    # Pre-Hospitalization Record
    pre_hospitalization_record: Optional[dict]    # Snapshot of SNF record before hospital admit
    pre_medications: list[dict]                   # Medications before hospitalization
    pre_conditions: list[dict]                    # Active conditions before hospitalization

    # Reconciliation
    reconciled_medications: list[dict]            # [{name, pre_dose, post_dose, status: new|changed|discontinued|unchanged, severity}]
    reconciled_diagnoses: list[dict]              # [{code, description, status: new|resolved|unchanged, severity}]
    discrepancies: list[dict]                     # [{type, description, severity: critical|moderate|informational, action_required}]

    # Output
    discrepancy_report: Optional[str]             # AI-generated reconciliation narrative
    a2a_alert_sent: bool                          # Whether provider was notified via A2A

    # Workflow
    confidence: float
    status: str                                   # ingesting | reconciling | report_generated | reviewed
    error: Optional[str]
```

### Graph Definition

```python
def ingest_discharge_summary(state: SNFBridgeState) -> dict:
    """Ingest discharge summary from hospital -- handles both FHIR and unstructured formats."""
    from app.services.document_ingestion import DocumentIngestionService

    ingestion = DocumentIngestionService()

    # Try FHIR endpoint first
    structured = ingestion.try_fhir_ingest(
        hospital_id=state["hospital_source"],
        patient_id=state["patient_id"],
        event_id=state["discharge_event_id"],
    )

    if structured:
        return {
            "discharge_summary_structured": structured,
            "discharge_medications": structured.get("medications", []),
            "discharge_diagnoses": structured.get("diagnoses", []),
            "discharge_followups": structured.get("followups", []),
            "status": "ingested_structured",
        }

    # Fall back to unstructured (PDF/scanned document)
    raw_text = ingestion.extract_text(
        source=state["hospital_source"],
        patient_id=state["patient_id"],
        event_id=state["discharge_event_id"],
    )

    return {
        "discharge_summary_raw": raw_text,
        "status": "ingested_unstructured",
    }


def parse_unstructured_discharge(state: SNFBridgeState) -> dict:
    """NLP extraction from unstructured discharge summary."""
    if state.get("discharge_summary_structured"):
        return {"status": "already_structured"}

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8192,
        system="""You are a clinical NLP system specializing in hospital discharge summary parsing.
Extract all structured clinical data from the discharge summary.

EXTRACT:
1. Discharge medications (name, dose, route, frequency, new/changed/continued)
2. Discharge diagnoses (description, ICD-10 code if mentioned, primary vs secondary)
3. Follow-up instructions (appointment type, timeframe, specialist, reason)
4. Vital signs at discharge
5. Procedures performed during hospitalization
6. Activity restrictions
7. Diet orders
8. Wound care instructions (if applicable)

Return JSON:
{
  "medications": [{"name": str, "dose": str, "route": str, "frequency": str, "status": "new"|"changed"|"continued"}],
  "diagnoses": [{"description": str, "icd10": str | null, "type": "primary"|"secondary"}],
  "followups": [{"type": str, "timeframe": str, "specialist": str | null, "reason": str}],
  "procedures": [{"name": str, "date": str | null}],
  "restrictions": [str],
  "diet": str | null,
  "wound_care": str | null,
  "confidence": float
}

RULES:
- Extract EXACTLY what is documented -- do not infer or add information
- If a field is ambiguous, mark confidence lower
- Preserve medication brand/generic names as written""",
        messages=[{
            "role": "user",
            "content": state["discharge_summary_raw"]
        }],
    )

    parsed = json.loads(response.content[0].text)
    return {
        "discharge_summary_structured": parsed,
        "discharge_medications": parsed["medications"],
        "discharge_diagnoses": parsed["diagnoses"],
        "discharge_followups": parsed["followups"],
        "confidence": parsed["confidence"],
        "status": "parsed",
    }


def retrieve_pre_hospitalization_record(state: SNFBridgeState) -> dict:
    """Retrieve the patient's SNF record from before hospitalization."""
    from app.services.fhir import FHIRService

    fhir = FHIRService()
    pre_record = fhir.get_patient_snapshot(
        patient_id=state["patient_id"],
        as_of=state["discharge_event_id"],  # Snapshot before admission
        include=["MedicationRequest", "Condition", "CarePlan", "AllergyIntolerance"],
    )

    return {
        "pre_hospitalization_record": pre_record,
        "pre_medications": pre_record.get("medications", []),
        "pre_conditions": pre_record.get("conditions", []),
        "status": "pre_record_loaded",
    }


def reconcile(state: SNFBridgeState) -> dict:
    """Semantic reconciliation of discharge data against pre-hospitalization record."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8192,
        system="""You are a medication reconciliation and clinical data reconciliation expert.
Compare the hospital discharge data against the SNF pre-hospitalization record.

MEDICATION RECONCILIATION:
For each medication, determine:
- NEW: Not in pre-hospitalization list (highest attention)
- CHANGED: Same medication, different dose/frequency/route
- DISCONTINUED: In pre-hospitalization list but not in discharge list
- UNCHANGED: Same medication, same parameters

DIAGNOSIS RECONCILIATION:
- NEW: Diagnosis not previously documented
- RESOLVED: Previously documented but not in discharge list
- UNCHANGED: Present in both records

SEVERITY CLASSIFICATION:
- CRITICAL: New high-risk medications (anticoagulants, insulin, opioids), discontinued essential medications, new major diagnoses
- MODERATE: Dose changes, new low-risk medications, resolved conditions
- INFORMATIONAL: Unchanged items, minor documentation differences

Return JSON:
{
  "reconciled_medications": [{"name": str, "pre_dose": str|null, "post_dose": str|null, "status": str, "severity": str, "note": str}],
  "reconciled_diagnoses": [{"description": str, "code": str|null, "status": str, "severity": str}],
  "discrepancies": [{"type": "medication"|"diagnosis"|"followup", "description": str, "severity": str, "action_required": str}],
  "confidence": float
}""",
        messages=[{
            "role": "user",
            "content": f"""Pre-Hospitalization Medications:\n{json.dumps(state['pre_medications'], indent=2)}\n\nDischarge Medications:\n{json.dumps(state['discharge_medications'], indent=2)}\n\nPre-Hospitalization Conditions:\n{json.dumps(state['pre_conditions'], indent=2)}\n\nDischarge Diagnoses:\n{json.dumps(state['discharge_diagnoses'], indent=2)}\n\nFollow-up Instructions:\n{json.dumps(state['discharge_followups'], indent=2)}"""
        }],
    )

    reconciliation = json.loads(response.content[0].text)
    return {
        "reconciled_medications": reconciliation["reconciled_medications"],
        "reconciled_diagnoses": reconciliation["reconciled_diagnoses"],
        "discrepancies": reconciliation["discrepancies"],
        "confidence": reconciliation["confidence"],
        "status": "reconciled",
    }


def generate_discrepancy_report(state: SNFBridgeState) -> dict:
    """Generate human-readable discrepancy report for the attending provider."""
    critical_count = sum(1 for d in state["discrepancies"] if d.get("severity") == "critical")
    moderate_count = sum(1 for d in state["discrepancies"] if d.get("severity") == "moderate")

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system="""Generate a concise clinical reconciliation report for the attending physician.

FORMAT:
## Transition of Care Reconciliation Report
### Critical Items (Action Required)
[List critical discrepancies with clear action items]
### Medication Changes
[Table: medication | pre-hospital | post-hospital | status | action]
### New Diagnoses
[List with ICD-10 codes if available]
### Follow-up Requirements
[Ordered by urgency]
### Summary
[2-3 sentence executive summary]

Use clinical language. Be precise. Flag anything that could cause a medication error.""",
        messages=[{
            "role": "user",
            "content": f"""Reconciled Medications:\n{json.dumps(state['reconciled_medications'], indent=2)}\n\nReconciled Diagnoses:\n{json.dumps(state['reconciled_diagnoses'], indent=2)}\n\nDiscrepancies:\n{json.dumps(state['discrepancies'], indent=2)}"""
        }],
    )

    return {
        "discrepancy_report": response.content[0].text,
        "status": "report_generated",
    }


def alert_provider(state: SNFBridgeState) -> dict:
    """Send reconciliation report to attending provider via A2A."""
    from app.services.a2a import A2AClient

    a2a = A2AClient()
    critical_count = sum(1 for d in state["discrepancies"] if d.get("severity") == "critical")

    priority = "urgent" if critical_count > 0 else "normal"

    a2a.send_message(
        target_agent="patient-communication",
        task_type="provider_alert",
        payload={
            "patient_id": state["patient_id"],
            "alert_type": "transition_of_care_reconciliation",
            "priority": priority,
            "critical_items": critical_count,
            "report_summary": state["discrepancy_report"][:500],
        },
    )
    return {"a2a_alert_sent": True, "status": "alerted"}


def physician_review(state: SNFBridgeState) -> dict:
    """ALL medication changes require physician/pharmacist review."""
    decision = interrupt({
        "type": "transition_reconciliation_review",
        "patient_id": state["patient_id"],
        "discrepancy_report": state["discrepancy_report"],
        "reconciled_medications": state["reconciled_medications"],
        "reconciled_diagnoses": state["reconciled_diagnoses"],
        "discrepancies": state["discrepancies"],
        "confidence": state["confidence"],
        "message": "Review transition of care reconciliation. ALL medication changes require approval before updating patient record.",
    })
    return {
        "status": "reviewed",
    }


# --- Build graph ---

bridge_builder = StateGraph(SNFBridgeState)

bridge_builder.add_node("ingest_discharge_summary", ingest_discharge_summary)
bridge_builder.add_node("parse_unstructured_discharge", parse_unstructured_discharge)
bridge_builder.add_node("retrieve_pre_hospitalization_record", retrieve_pre_hospitalization_record)
bridge_builder.add_node("reconcile", reconcile)
bridge_builder.add_node("generate_discrepancy_report", generate_discrepancy_report)
bridge_builder.add_node("alert_provider", alert_provider)
bridge_builder.add_node("physician_review", physician_review)

bridge_builder.add_edge(START, "ingest_discharge_summary")
bridge_builder.add_edge("ingest_discharge_summary", "parse_unstructured_discharge")
bridge_builder.add_edge("parse_unstructured_discharge", "retrieve_pre_hospitalization_record")
bridge_builder.add_edge("retrieve_pre_hospitalization_record", "reconcile")
bridge_builder.add_edge("reconcile", "generate_discrepancy_report")
bridge_builder.add_edge("generate_discrepancy_report", "alert_provider")
bridge_builder.add_edge("alert_provider", "physician_review")
bridge_builder.add_edge("physician_review", END)

bridge_graph = bridge_builder.compile(checkpointer=checkpointer)
```

### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Medication matching | >= 0.90 | 0.80-0.90 | < 0.80 |
| Diagnosis reconciliation | >= 0.85 | 0.75-0.85 | < 0.75 |
| Follow-up extraction | >= 0.80 | 0.70-0.80 | < 0.70 |
| Unstructured document parsing | >= 0.85 | 0.75-0.85 | < 0.75 |

### Human-in-the-Loop Gates

- **ALL medication changes** require pharmacist/physician review before updating the patient record
- **New diagnoses** require attending physician confirmation
- **Discontinued essential medications** trigger immediate physician alert (P1 equivalent)

### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, cross-system identifiers |
| Read | `MedicationRequest` | Pre-hospitalization medication list |
| Read | `Condition` | Pre-hospitalization active conditions |
| Read | `DocumentReference` | Discharge summary document |
| Read | `Encounter` | Hospital admission/discharge events |
| Read | `DiagnosticReport` | Hospital lab/imaging results |
| Write | `MedicationRequest` | Updated medication orders (after physician review) |
| Write | `Condition` | New diagnoses (after physician confirmation) |
| Write | `DocumentReference` | Reconciliation report |
| Write | `Provenance` | Audit trail of reconciliation actions |

### Clinical Impact

- **Solves Di Rezze's founding problem**: The "false sense of security" when patients return from hospital to SNF without proper data reconciliation
- **Medication safety**: Catches drug interactions, duplicate therapies, and omitted essential medications at transitions
- **Reduces readmissions**: Proper transition reconciliation reduces 30-day readmission rates by 20-30%

---

## 16. Agent 10: Generative Care Plan Optimizer (Theoria Medical)

This agent functions as an AI super-consultant, synthesizing longitudinal patient data with the latest clinical guidelines (via RAG from pgvector) to generate proactive care recommendations. Designed to scale top-clinician expertise across Theoria's entire 200+ facility network.

### Module Mapping

- **Module B** (Provider Workflow): Clinical decision support, care plan updates
- **Module E** (Population Health): Evidence-based population interventions

### Purpose

Analyzes a patient's complete longitudinal record (vitals, labs, medications, device readings, encounter notes) against clinical guidelines stored in pgvector, identifies deterioration patterns before they become acute, and generates evidence-based care recommendations with confidence scores and literature citations. Every recommendation includes the evidence level (A/B/C) and requires physician approval.

### Trigger

- Scheduled weekly analysis per patient
- Triggered by significant vital sign change (from Guardian Agent via A2A)
- On-demand physician request ("What should we consider for this patient?")

### State Definition

```python
class CareOptimizationState(TypedDict):
    # Input
    patient_id: str
    trigger_type: str                             # scheduled | vital_change | on_demand
    trigger_context: Optional[dict]               # Context from triggering event

    # Longitudinal Data
    longitudinal_data: Optional[dict]             # Comprehensive patient history
    vitals_trend: list[dict]                      # Last 90 days of vitals
    labs_history: list[dict]                      # Last 12 months of lab results
    medication_history: list[dict]                # Full medication history
    device_readings: list[dict]                   # RPM device data (if available)

    # Evidence
    evidence_sources: list[dict]                  # [{guideline, citation, relevance_score}]
    rag_context: Optional[str]                    # Retrieved guideline text from pgvector

    # Recommendations
    recommendations: list[dict]                   # [{action, evidence_level: A|B|C, confidence, rationale, citations}]
    risk_predictions: list[dict]                  # [{condition, probability, timeframe, preventive_action}]

    # Review
    approval_status: Optional[str]                # pending | approved | partially_approved | rejected
    physician_notes: Optional[str]

    # Workflow
    confidence: float
    status: str                                   # gathering | analyzing | recommendations_ready | reviewed
    error: Optional[str]
```

### Graph Definition

```python
def gather_longitudinal_data(state: CareOptimizationState) -> dict:
    """Gather comprehensive patient history from FHIR store."""
    from app.services.fhir import FHIRService
    from app.services.context import ContextRehydrationEngine

    fhir = FHIRService()
    context = ContextRehydrationEngine()

    # Pull full longitudinal record
    record = context.rehydrate(
        patient_id=state["patient_id"],
        include=["conditions", "medications", "vitals_90d", "labs_12m",
                 "encounters", "care_plans", "device_readings", "allergies"],
        depth="full",
    )

    return {
        "longitudinal_data": record.full_summary,
        "vitals_trend": record.vitals_trend,
        "labs_history": record.labs_history,
        "medication_history": record.medication_history,
        "device_readings": record.device_readings or [],
        "status": "gathered",
    }


def retrieve_clinical_guidelines(state: CareOptimizationState) -> dict:
    """RAG retrieval of relevant clinical guidelines from pgvector."""
    from app.services.embeddings import get_embedding

    # Build query from patient's active conditions
    conditions_text = " ".join([
        c.get("display", "") for c in (state.get("longitudinal_data", {}).get("conditions", []))
    ])

    embedding = get_embedding(f"Clinical guidelines for: {conditions_text}")

    import psycopg2
    with psycopg2.connect(POSTGRES_URI) as conn:
        with conn.cursor() as cur:
            cur.execute("""
                SELECT chunk_text, metadata, 1 - (embedding <=> %s::vector) as similarity
                FROM clinical_guidelines_embeddings
                WHERE 1 - (embedding <=> %s::vector) > 0.70
                ORDER BY embedding <=> %s::vector
                LIMIT 10
            """, (embedding, embedding, embedding))

            guidelines = [
                {
                    "text": row[0],
                    "metadata": row[1],
                    "relevance": float(row[2]),
                }
                for row in cur.fetchall()
            ]

    return {
        "evidence_sources": [{"guideline": g["metadata"].get("source", "Unknown"), "citation": g["metadata"].get("citation", ""), "relevance_score": g["relevance"]} for g in guidelines],
        "rag_context": "\n\n".join([g["text"] for g in guidelines]),
        "status": "guidelines_retrieved",
    }


def generate_recommendations(state: CareOptimizationState) -> dict:
    """Generate evidence-based care recommendations."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8192,
        system="""You are an evidence-based clinical decision support system. Generate proactive
care recommendations by synthesizing the patient's longitudinal data with clinical guidelines.

EVIDENCE LEVELS:
- A: Strong evidence from RCTs or meta-analyses
- B: Moderate evidence from well-designed studies
- C: Expert consensus or case series

RECOMMENDATION CATEGORIES:
1. Medication adjustments (dose changes, additions, deprescribing)
2. Diagnostic workup (labs, imaging, referrals to order)
3. Preventive interventions (screenings, vaccinations, lifestyle modifications)
4. Care plan modifications (frequency of visits, RPM parameter changes, goal updates)
5. Risk predictions (conditions likely to develop/worsen based on trends)

Return JSON:
{
  "recommendations": [
    {
      "action": str,
      "category": "medication"|"diagnostic"|"preventive"|"care_plan"|"risk",
      "evidence_level": "A"|"B"|"C",
      "confidence": float,
      "rationale": str,
      "citations": [str],
      "urgency": "immediate"|"next_visit"|"next_month"|"routine",
      "contraindications_checked": [str]
    }
  ],
  "risk_predictions": [
    {
      "condition": str,
      "probability": float,
      "timeframe": str,
      "preventive_action": str,
      "supporting_data": [str]
    }
  ],
  "overall_confidence": float,
  "clinical_summary": str
}

CRITICAL RULES:
- NEVER recommend without evidence citation
- ALWAYS check for contraindications against current medications and allergies
- Flag drug-drug interactions explicitly
- Evidence Level C recommendations require attending physician (not NP/PA) approval
- Include deprescribing recommendations when appropriate (polypharmacy in elderly SNF patients)
- Factor in patient age, renal function, and hepatic function for medication recommendations""",
        messages=[{
            "role": "user",
            "content": f"""Patient Data:\n{json.dumps(state['longitudinal_data'], indent=2)}\n\nVitals Trend (90 days):\n{json.dumps(state['vitals_trend'][:30], indent=2)}\n\nLabs (12 months):\n{json.dumps(state['labs_history'][:20], indent=2)}\n\nDevice Readings:\n{json.dumps(state['device_readings'][:20], indent=2)}\n\nClinical Guidelines:\n{state.get('rag_context', 'No guidelines retrieved')}"""
        }],
    )

    result = json.loads(response.content[0].text)
    return {
        "recommendations": result["recommendations"],
        "risk_predictions": result.get("risk_predictions", []),
        "confidence": result["overall_confidence"],
        "status": "recommendations_ready",
    }


def physician_review_recommendations(state: CareOptimizationState) -> dict:
    """ALWAYS requires physician review -- this is clinical decision support, not autonomous care."""
    decision = interrupt({
        "type": "care_plan_optimization_review",
        "patient_id": state["patient_id"],
        "recommendations": state["recommendations"],
        "risk_predictions": state["risk_predictions"],
        "evidence_sources": state["evidence_sources"],
        "confidence": state["confidence"],
        "message": "Review AI-generated care recommendations. All medication changes require physician approval.",
    })

    return {
        "approval_status": decision.get("status", "pending"),
        "physician_notes": decision.get("notes", ""),
        "status": "reviewed",
    }


# --- Build graph ---

optimizer_builder = StateGraph(CareOptimizationState)

optimizer_builder.add_node("gather_longitudinal_data", gather_longitudinal_data)
optimizer_builder.add_node("retrieve_clinical_guidelines", retrieve_clinical_guidelines)
optimizer_builder.add_node("generate_recommendations", generate_recommendations)
optimizer_builder.add_node("physician_review_recommendations", physician_review_recommendations)

optimizer_builder.add_edge(START, "gather_longitudinal_data")
optimizer_builder.add_edge("gather_longitudinal_data", "retrieve_clinical_guidelines")
optimizer_builder.add_edge("retrieve_clinical_guidelines", "generate_recommendations")
optimizer_builder.add_edge("generate_recommendations", "physician_review_recommendations")
optimizer_builder.add_edge("physician_review_recommendations", END)

optimizer_graph = optimizer_builder.compile(checkpointer=checkpointer)
```

### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Evidence-matched recommendations (Level A/B) | >= 0.90 | 0.80-0.90 | < 0.80 |
| Novel combinations (Level C) | N/A | N/A | ALWAYS physician review |
| Risk predictions | >= 0.85 | 0.70-0.85 | < 0.70 |
| Medication recommendations | N/A | N/A | ALWAYS physician review |

### Human-in-the-Loop Gates

- **ALL recommendations** require physician review (this is decision SUPPORT, not autonomous care)
- **Medication changes** require attending physician approval (NP/PA cannot approve Level C)
- **Evidence Level C** recommendations require explicit attending physician sign-off
- **Deprescribing** recommendations require pharmacist + physician review

### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Patient` | Demographics, age, clinical context |
| Read | `CarePlan` | Active care plan for update context |
| Read | `Observation` | Vitals, labs, device readings (longitudinal) |
| Read | `MedicationRequest` | Full medication history |
| Read | `Condition` | Active and historical conditions |
| Read | `AllergyIntolerance` | Allergy/interaction checking |
| Read | `DocumentReference` | Clinical notes for context |
| Write | `CarePlan` | Updated care plan (after physician approval) |
| Write | `RequestGroup` | Grouped order recommendations |
| Write | `ClinicalImpression` | AI-generated clinical assessment |

### Clinical / Revenue Impact

- **Scales expertise**: Top-quartile clinical decision-making replicated across 200+ facilities
- **Prevents adverse events**: Proactive intervention before deterioration becomes acute
- **Kill shot example**: "Weight up 4 lbs in 48h + nocturnal O2 desaturation -> 85% predictive of CHF exacerbation within 72h. Recommend: 1) 40mg Lasix IV Push, 2) Stat telemedicine check-in, 3) Daily weight protocol"
- **Revenue**: Better clinical outcomes = higher ACO REACH quality scores = larger shared savings distribution

---

## 17. Agent 11: Dynamic Staffing & Resource Allocation Agent (Theoria Medical)

This agent optimizes provider scheduling and staffing across Theoria's 200+ post-acute facilities. Uses census data, acuity predictions, and cost modeling to generate cost-optimal staffing recommendations. Directly addresses Amulet Capital Partners' margin expansion mandate.

### Module Mapping

- **Module B** (Provider Workflow): Provider scheduling, workload optimization
- **Module F** (Patient Engagement): Census management, capacity planning

### Purpose

Integrates scheduling data, facility census, patient acuity scores, and provider availability to predict staffing needs 72 hours in advance. Generates cost-optimal provider assignments that balance travel time, provider competencies, patient needs, and budget constraints. Reduces reliance on expensive locum tenens providers ($2-5K/day).

### Trigger

- Nightly optimization run (2:00 AM ET, after ACO Quality Agent completes)
- Real-time surge detection (census spike, mass admit/discharge events)
- Manual request from operations team

### State Definition

```python
class StaffingOptimizationState(TypedDict):
    # Input
    optimization_window: str                      # "72h" | "1w" | "2w"
    trigger_type: str                             # nightly | surge | manual

    # Facility Data
    facilities: list[dict]                        # [{facility_id, name, census, capacity, acuity_distribution, current_providers, location}]
    total_facilities: int
    facilities_over_capacity: int
    facilities_under_staffed: int

    # Provider Data
    providers: list[dict]                         # [{provider_id, name, availability, location, competencies, cost_rate, current_assignments}]
    total_providers: int

    # Predictions
    census_predictions: list[dict]                # [{facility_id, date, predicted_census, confidence}]
    surge_risk: list[dict]                        # [{facility_id, risk_level, trigger}]

    # Optimization Results
    schedule_recommendations: list[dict]           # [{provider_id, facility_id, shift, date, rationale, cost}]
    cost_analysis: Optional[dict]                  # {current_cost, optimized_cost, savings, locum_avoided}

    # Workflow
    confidence: float
    status: str                                    # gathering | predicting | optimizing | ready | approved
    error: Optional[str]
```

### Graph Definition

```python
def gather_facility_data(state: StaffingOptimizationState) -> dict:
    """Gather current census, acuity, and staffing data across all facilities."""
    from app.services.facilities import FacilityService

    facilities = FacilityService()
    facility_data = facilities.get_all_with_census()

    over_capacity = sum(1 for f in facility_data if f["census"] > f["capacity"] * 0.95)
    under_staffed = sum(1 for f in facility_data if f["current_providers"] < f["required_providers"])

    return {
        "facilities": facility_data,
        "total_facilities": len(facility_data),
        "facilities_over_capacity": over_capacity,
        "facilities_under_staffed": under_staffed,
        "status": "facility_data_gathered",
    }


def gather_provider_data(state: StaffingOptimizationState) -> dict:
    """Gather provider availability, competencies, and current assignments."""
    from app.services.scheduling import SchedulingService

    scheduling = SchedulingService()
    provider_data = scheduling.get_all_providers_with_availability(
        window=state["optimization_window"],
    )

    return {
        "providers": provider_data,
        "total_providers": len(provider_data),
        "status": "provider_data_gathered",
    }


def predict_census(state: StaffingOptimizationState) -> dict:
    """Predict census trends for the optimization window."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system="""You are a healthcare operations analytics system. Predict census trends for
post-acute care facilities based on current census, historical patterns, and known admissions/discharges.

FACTORS TO CONSIDER:
- Day of week patterns (higher admissions Mon-Wed, higher discharges Fri)
- Seasonal patterns (flu season, holiday staffing)
- Scheduled admissions/discharges
- Historical census volatility per facility
- Current census trajectory (trending up/down/stable)

Return JSON:
{
  "predictions": [{"facility_id": str, "date": str, "predicted_census": int, "confidence": float, "trend": "up"|"down"|"stable"}],
  "surge_risks": [{"facility_id": str, "risk_level": "high"|"moderate"|"low", "trigger": str}],
  "confidence": float
}""",
        messages=[{
            "role": "user",
            "content": f"""Facilities ({state['total_facilities']}):\n{json.dumps(state['facilities'][:30], indent=2)}\n\nOptimization Window: {state['optimization_window']}"""
        }],
    )

    predictions = json.loads(response.content[0].text)
    return {
        "census_predictions": predictions["predictions"],
        "surge_risk": predictions["surge_risks"],
        "confidence": predictions["confidence"],
        "status": "predicted",
    }


def optimize_staffing(state: StaffingOptimizationState) -> dict:
    """Generate cost-optimal staffing recommendations."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8192,
        system="""You are a healthcare workforce optimization system. Generate cost-optimal
provider scheduling recommendations.

OPTIMIZATION OBJECTIVES (in priority order):
1. Patient safety: Adequate staffing for acuity levels (hard constraint)
2. Regulatory compliance: Meet state-mandated staffing ratios (hard constraint)
3. Cost minimization: Prefer employed providers over locum tenens
4. Travel optimization: Minimize provider travel time between facilities
5. Continuity of care: Prefer same provider at same facility when possible
6. Provider satisfaction: Respect provider schedule preferences when possible

CONSTRAINTS:
- No provider can work > 12 hours per shift
- No provider can work > 60 hours per week
- High-acuity facilities require MD coverage (not NP/PA alone)
- Travel time between facilities must allow 30-min buffer

Return JSON:
{
  "schedule_recommendations": [
    {
      "provider_id": str,
      "facility_id": str,
      "shift": "day"|"evening"|"night",
      "date": str,
      "rationale": str,
      "cost": float,
      "replaces_locum": bool
    }
  ],
  "cost_analysis": {
    "current_weekly_cost": float,
    "optimized_weekly_cost": float,
    "weekly_savings": float,
    "locum_shifts_avoided": int,
    "locum_cost_avoided": float
  },
  "unfilled_shifts": [{"facility_id": str, "shift": str, "date": str, "reason": str}],
  "confidence": float
}""",
        messages=[{
            "role": "user",
            "content": f"""Providers ({state['total_providers']}):\n{json.dumps(state['providers'][:50], indent=2)}\n\nFacilities:\n{json.dumps(state['facilities'][:30], indent=2)}\n\nCensus Predictions:\n{json.dumps(state['census_predictions'][:50], indent=2)}\n\nSurge Risks:\n{json.dumps(state['surge_risk'], indent=2)}"""
        }],
    )

    optimization = json.loads(response.content[0].text)
    return {
        "schedule_recommendations": optimization["schedule_recommendations"],
        "cost_analysis": optimization["cost_analysis"],
        "confidence": optimization["confidence"],
        "status": "optimized",
    }


def operations_review(state: StaffingOptimizationState) -> dict:
    """Operations team review of staffing recommendations."""
    changes_count = len(state["schedule_recommendations"])
    savings = state.get("cost_analysis", {}).get("weekly_savings", 0)

    # Large changes require ops director approval
    requires_director = changes_count > 5 or savings > 10000

    decision = interrupt({
        "type": "staffing_optimization_review",
        "schedule_recommendations": state["schedule_recommendations"],
        "cost_analysis": state["cost_analysis"],
        "total_changes": changes_count,
        "requires_director_approval": requires_director,
        "message": f"Staffing optimization: {changes_count} schedule changes, ${savings:.0f}/week savings",
    })

    return {
        "status": "approved" if decision.get("approved") else "rejected",
    }


# --- Build graph ---

staffing_builder = StateGraph(StaffingOptimizationState)

staffing_builder.add_node("gather_facility_data", gather_facility_data)
staffing_builder.add_node("gather_provider_data", gather_provider_data)
staffing_builder.add_node("predict_census", predict_census)
staffing_builder.add_node("optimize_staffing", optimize_staffing)
staffing_builder.add_node("operations_review", operations_review)

staffing_builder.add_edge(START, "gather_facility_data")
staffing_builder.add_edge("gather_facility_data", "gather_provider_data")
staffing_builder.add_edge("gather_provider_data", "predict_census")
staffing_builder.add_edge("predict_census", "optimize_staffing")
staffing_builder.add_edge("optimize_staffing", "operations_review")
staffing_builder.add_edge("operations_review", END)

staffing_graph = staffing_builder.compile(checkpointer=checkpointer)
```

### Confidence Thresholds

| Task | Auto-Execute | Flag for Review | Full Escalation |
|------|-------------|-----------------|-----------------|
| Census prediction (72h) | >= 0.85 | 0.70-0.85 | < 0.70 |
| Cost optimization | >= 0.80 | 0.65-0.80 | < 0.65 |
| Surge detection | >= 0.85 | 0.70-0.85 | < 0.70 |
| Schedule changes | N/A | N/A | ALWAYS operations review |

### Human-in-the-Loop Gates

- **All schedule changes** require operations team review
- **Changes affecting > 5 providers** require ops director approval
- **Cost savings > $10K/week** require ops director + finance review
- **Surge response** (emergency staffing) can bypass normal approval with shift supervisor authorization

### FHIR Resources

| Operation | Resource | Description |
|-----------|----------|-------------|
| Read | `Schedule` | Provider schedules |
| Read | `Practitioner` | Provider demographics, competencies |
| Read | `PractitionerRole` | Provider-facility assignments |
| Read | `Location` | Facility details, capacity, coordinates |
| Read | `Organization` | Facility organization hierarchy |
| Read | `Encounter` | Current census (active encounters) |
| Write | `Schedule` | Updated provider schedules (after approval) |
| Write | `PractitionerRole` | Updated provider-facility assignments |

### Revenue Impact

- **Reduces locum tenens costs**: Each locum shift avoided saves $2-5K/day
- **Addresses Amulet Capital mandate**: PE partner requires margin expansion; staffing is the largest cost center
- **Prevents understaffing penalties**: State regulators fine facilities for staffing ratio violations
- **Projected savings**: 10-15% reduction in total provider staffing costs across Theoria network

---

## Summary

This document covers the complete implementation of eleven LangGraph-based AI agents for MedOS:

**Foundation Agents (1-4):**
1. **Clinical Documentation Agent** -- audio to finalized SOAP note with codes
2. **Prior Authorization Agent** -- end-to-end PA lifecycle management
3. **Denial Management Agent** -- denial analysis, evidence gathering, appeal generation
4. **AI Coding Agent** -- NLP-based diagnosis extraction and code mapping with pgvector

**Theoria Medical Agents (5-11):**
5. **Post-Acute Guardian Agent** -- wearable device monitoring, CHF prevention, P1-P4 alert triage
6. **CCM Revenue Agent** -- chronic care management time tracking, CPT 99490/99491 billing
7. **Shift Summary Agent** -- provider handoff briefings across 50+ facilities
8. **ACO REACH Quality Agent** -- care gap identification, shared savings optimization
9. **SNF-to-Hospital Semantic Data Bridge** -- discharge reconciliation, medication safety at transitions
10. **Generative Care Plan Optimizer** -- AI super-consultant, evidence-based recommendations via RAG
11. **Dynamic Staffing & Resource Allocation Agent** -- workforce optimization, locum cost reduction

Each agent shares common infrastructure: encrypted PostgreSQL checkpointing, HIPAA-compliant LLM wrapper with PHI scrubbing and audit logging, Langfuse observability, Redis-based task queuing, and ECS Fargate worker deployment with auto-scaling.

The Theoria Medical agents (5-11) are specifically designed for post-acute care at scale, leveraging [[ADR-006-context-rehydration]] for cross-facility context, [[ADR-007-wearable-iot-integration]] for device data, and [[ADR-008-a2a-agent-communication]] for inter-agent coordination. They target the unique challenges of managing 200+ SNF/ALF facilities under ACO REACH value-based contracts.

The human-in-the-loop pattern using LangGraph's `interrupt()` and `Command(resume=...)` is central to every agent. Healthcare AI cannot be fully autonomous -- the system is designed so that AI handles the heavy lifting while clinicians retain final authority over clinical decisions.

For framework selection rationale, see [[ADR-003-ai-agent-framework]]. For the broader system context, see [[System-Architecture-Overview]].
