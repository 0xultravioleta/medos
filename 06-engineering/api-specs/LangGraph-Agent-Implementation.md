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

## Summary

This document covers the complete implementation of four LangGraph-based AI agents for MedOS:

1. **Clinical Documentation Agent** -- audio to finalized SOAP note with codes
2. **Prior Authorization Agent** -- end-to-end PA lifecycle management
3. **Denial Management Agent** -- denial analysis, evidence gathering, appeal generation
4. **AI Coding Agent** -- NLP-based diagnosis extraction and code mapping with pgvector

Each agent shares common infrastructure: encrypted PostgreSQL checkpointing, HIPAA-compliant LLM wrapper with PHI scrubbing and audit logging, Langfuse observability, Redis-based task queuing, and ECS Fargate worker deployment with auto-scaling.

The human-in-the-loop pattern using LangGraph's `interrupt()` and `Command(resume=...)` is central to every agent. Healthcare AI cannot be fully autonomous -- the system is designed so that AI handles the heavy lifting while clinicians retain final authority over clinical decisions.

For framework selection rationale, see [[ADR-003-ai-agent-framework]]. For the broader system context, see [[System-Architecture-Overview]].
