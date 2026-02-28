---
type: epic
date: "2026-02-27"
status: planning
priority: 4
tags:
  - project
  - epic
  - phase-1
  - ai
  - clinical
  - module-b
  - ambient
owner: ""
target-date: "2026-04-10"
---

# EPIC-004: AI Clinical Documentation (The Hero Feature)

> **Timeline:** Week 3-6 (2026-03-13 to 2026-04-10)
> **Phase:** 1 - Foundation
> **Dependencies:** [[EPIC-001-aws-infrastructure-foundation]] (ECS, GPU compute), [[EPIC-002-auth-identity-system]] (auth, audit), [[EPIC-003-fhir-data-layer]] (FHIR CRUD, events, pgvector)
> **Blocks:** [[EPIC-005-revenue-cycle-mvp]] (AI coding suggestions feed billing), [[EPIC-006-pilot-readiness]]

## Objective

Build the ambient AI clinical documentation system -- the hero feature that differentiates MedOS. A provider walks into a patient room, starts recording, conducts a natural conversation, and within 30 seconds of ending the encounter receives a structured SOAP note with ICD-10/CPT suggestions. This single feature replaces 2+ hours of daily documentation work per provider.

---

## Timeline (Gantt)

```
Week 3 (Mar 13 - Mar 20)
|----- T1: Audio Capture Pipeline ------------------|
|----- T2: Speech-to-Text (Whisper v3) -------------|

Week 4 (Mar 21 - Mar 27)
|----------- T3: Clinical NLU Pipeline -------------|
|----------- T4: SOAP Note Generation (Claude) -----|

Week 5 (Mar 28 - Apr 3)
|----- T5: Provider Review Interface (Next.js) -----|
|----- T6: AI Coding Suggestions (ICD-10 + CPT) ----|

Week 6 (Apr 4 - Apr 10)
|----- T7: FHIR Resource Generation -----------------|
|----- T8: Confidence Scoring + HITL -----------------|
|----------- T9: Provenance + Audit Trail -----------|
|----------- T10: Performance Optimization ----------|
```

---

## Tasks

### T1: Audio Capture Pipeline
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** [[EPIC-001-aws-infrastructure-foundation]] T7 (ECS), T5 (S3)
**References:** [[Ambient-AI-Documentation]], [[System-Architecture-Overview]]

**Description:**
Build the browser-based audio capture pipeline that records the provider-patient conversation and streams or uploads it to the backend for transcription.

**Subtasks:**
- [ ] Implement browser audio capture:
  - MediaRecorder API with `audio/webm;codecs=opus` (best compression)
  - Fallback to `audio/wav` for unsupported browsers
  - Sample rate: 16kHz (optimal for speech recognition)
  - Mono channel (stereo unnecessary for single-room recording)
  - Visual recording indicator with waveform display
- [ ] Implement audio streaming (WebSocket):
  - Stream audio chunks every 5 seconds to backend
  - WebSocket connection: `wss://api.medos.health/ws/audio`
  - Authentication via JWT token in connection handshake
  - Reconnection logic with buffering on network interruption
- [ ] Implement batch upload fallback:
  - If WebSocket unavailable, record full audio locally
  - Upload via `POST /api/v1/encounters/{id}/audio`
  - Chunked upload for files > 10MB
  - Resume upload on network failure
- [ ] Audio pre-processing:
  - Noise reduction (WebAudio API or server-side)
  - Volume normalization
  - Voice activity detection (skip silence segments)
  - Audio duration tracking
- [ ] Implement audio storage:
  - Store in S3 tenant bucket with KMS encryption
  - Path: `{tenant_id}/encounters/{encounter_id}/audio/{timestamp}.webm`
  - Retention policy: 30 days after note approval (configurable per tenant)
  - Auto-delete after retention period (PHI minimization)
- [ ] Create audio metadata record:
  ```json
  {
    "encounter_id": "uuid",
    "duration_seconds": 840,
    "format": "audio/webm;codecs=opus",
    "sample_rate": 16000,
    "file_size_bytes": 1258000,
    "s3_key": "tenant-uuid/encounters/.../audio.webm",
    "status": "uploaded"
  }
  ```

**Acceptance Criteria:**
- [ ] Audio captures in Chrome, Firefox, Safari, and Edge
- [ ] WebSocket streaming delivers audio chunks with < 1s latency
- [ ] Batch upload handles files up to 60 minutes (approx. 50MB)
- [ ] Audio encrypted at rest (S3 KMS) and in transit (WSS/HTTPS)
- [ ] Recording indicator clearly visible to provider
- [ ] Auto-delete after retention period verified

---

### T2: Speech-to-Text Engine (Whisper v3)
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T1, [[EPIC-001-aws-infrastructure-foundation]] T7 (ECS/GPU)
**References:** [[Ambient-AI-Documentation]], [[ADR-003-ai-agent-framework]]

**Description:**
Deploy and integrate Whisper v3 for medical speech-to-text transcription. Self-hosted to keep PHI off third-party APIs and to control latency.

**Subtasks:**
- [ ] Deploy Whisper v3 large model on GPU instance:
  - Instance type: `g5.xlarge` (1x A10G GPU, 24GB VRAM)
  - Container: `openai/whisper` with FastAPI wrapper
  - Model: `whisper-large-v3` (best accuracy for medical terminology)
  - Endpoint: internal service (`whisper.medos.local:8080`)
  - Auto-scaling: 0-3 instances based on queue depth
  - Scale-to-zero when idle (cost optimization for dev)
- [ ] Implement transcription service:
  ```
  POST /transcribe
  Content-Type: audio/webm

  Response:
  {
    "text": "full transcript",
    "segments": [
      {
        "start": 0.0,
        "end": 5.2,
        "text": "So what brings you in today?",
        "speaker": "SPEAKER_1",
        "confidence": 0.94
      }
    ],
    "language": "en",
    "duration": 840.5
  }
  ```
- [ ] Implement speaker diarization:
  - Identify provider vs. patient voice segments
  - Use `pyannote/speaker-diarization` or WhisperX
  - Label segments: PROVIDER, PATIENT, OTHER
  - Handle multi-speaker scenarios (family member present)
- [ ] Implement medical vocabulary boosting:
  - Custom vocabulary list for medical terms
  - Specialty-specific term lists (cardiology, orthopedics, etc.)
  - Drug name recognition (RxNorm integration)
  - Initial prompt engineering for medical context:
    ```
    "This is a medical encounter between a physician and patient.
     Medical terms, drug names, and abbreviations are expected."
    ```
- [ ] Implement streaming transcription (real-time):
  - Accept WebSocket audio stream
  - Return partial transcriptions every 5 seconds
  - Final transcription after audio end
  - Display live transcript in provider UI
- [ ] Implement batch transcription (for uploaded recordings):
  - SQS queue for transcription jobs
  - Process uploaded audio files asynchronously
  - Notify via WebSocket when transcription complete
- [ ] Performance benchmarking:
  - WER (Word Error Rate) target: < 10% on medical conversations
  - Latency target: < 15 seconds for 15-minute encounter
  - Test with various accents and speaking speeds

**Acceptance Criteria:**
- [ ] Whisper v3 deployed and accessible as internal service
- [ ] Transcription WER < 10% on medical test dataset
- [ ] Speaker diarization correctly identifies provider vs. patient 90%+ of the time
- [ ] Streaming transcription displays in < 5 seconds
- [ ] Batch transcription completes within 2x real-time (15 min audio = < 30 min processing)
- [ ] GPU auto-scaling works correctly (scale up on load, scale down when idle)
- [ ] All audio and transcription data stays within MedOS infrastructure (no third-party API)

---

### T3: Clinical NLU Pipeline
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T2
**References:** [[Ambient-AI-Documentation]], [[ADR-003-ai-agent-framework]]

**Description:**
Extract structured clinical information from the raw transcript. This pipeline transforms unstructured conversation into the components needed for SOAP note generation.

**Subtasks:**
- [ ] Define clinical extraction schema:
  ```json
  {
    "chief_complaint": "chest pain for 3 days",
    "hpi": {
      "onset": "3 days ago",
      "location": "substernal",
      "duration": "intermittent, lasting 5-10 minutes",
      "character": "sharp, pressure-like",
      "aggravating_factors": ["exertion", "stress"],
      "relieving_factors": ["rest", "nitroglycerin"],
      "severity": "7/10",
      "associated_symptoms": ["shortness of breath", "diaphoresis"]
    },
    "review_of_systems": {
      "constitutional": { "positive": [], "negative": ["fever", "weight loss"] },
      "cardiovascular": { "positive": ["chest pain", "palpitations"], "negative": ["leg swelling"] },
      "respiratory": { "positive": ["dyspnea on exertion"], "negative": ["cough", "hemoptysis"] }
    },
    "physical_exam": {
      "vitals": { "bp": "142/88", "hr": "92", "temp": "98.6", "rr": "18", "spo2": "97%" },
      "findings": [
        { "system": "cardiovascular", "finding": "regular rate and rhythm, no murmurs" },
        { "system": "respiratory", "finding": "clear to auscultation bilaterally" }
      ]
    },
    "assessment": [
      { "condition": "Acute coronary syndrome", "icd10_suggestion": "I24.9", "confidence": 0.85 },
      { "condition": "Essential hypertension", "icd10_suggestion": "I10", "confidence": 0.92 }
    ],
    "plan": [
      { "action": "Order troponin levels and ECG", "type": "diagnostic" },
      { "action": "Start aspirin 325mg", "type": "medication" },
      { "action": "Cardiology consultation", "type": "referral" }
    ],
    "medications_mentioned": [
      { "name": "nitroglycerin", "context": "patient takes as needed" },
      { "name": "aspirin", "context": "new prescription" }
    ],
    "allergies_mentioned": [
      { "substance": "penicillin", "reaction": "rash" }
    ]
  }
  ```
- [ ] Implement extraction pipeline using Claude:
  - Input: diarized transcript + patient context (from pgvector search)
  - System prompt: medical documentation specialist persona
  - Structured output via Claude tool_use / JSON mode
  - Context window: include relevant patient history from pgvector (T9 from EPIC-003)
  - Temperature: 0 (deterministic output for clinical accuracy)
- [ ] Implement medical entity recognition:
  - Medications: extract name, dose, route, frequency
  - Conditions: map to ICD-10 candidates
  - Procedures: map to CPT candidates
  - Anatomical locations: standardize terminology
  - Lab values: extract value + unit + reference range context
- [ ] Implement negation detection:
  - "No chest pain" should NOT generate chest pain as positive finding
  - "Denies fever" should appear in negative ROS
  - Handle double negatives and hedging language
- [ ] Implement specialty-specific extraction templates:
  - Primary care (general SOAP)
  - Cardiology (cardiac-specific ROS, exam)
  - Orthopedics (MSK-specific exam, ROM measurements)
  - Dermatology (lesion descriptions, body location mapping)
  - Psychiatry (mental status exam, PHQ-9, GAD-7)
- [ ] Output validation:
  - Verify required SOAP sections present
  - Flag low-confidence extractions for human review
  - Cross-reference medications with patient's active medication list

**Acceptance Criteria:**
- [ ] Extracts all SOAP sections from 90%+ of test encounters
- [ ] Chief complaint identified correctly in 95%+ of cases
- [ ] Negation detection accuracy > 95%
- [ ] Medication extraction includes name + dose in 90%+ of mentions
- [ ] Processing time: < 10 seconds per encounter transcript
- [ ] Structured output passes JSON schema validation

---

### T4: SOAP Note Generation with Claude
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T3
**References:** [[Ambient-AI-Documentation]], [[ADR-003-ai-agent-framework]]

**Description:**
Generate polished, provider-ready SOAP notes from the NLU extraction output. Notes must read as if written by the provider, in professional medical language.

**Subtasks:**
- [ ] Design SOAP note template system:
  ```
  SUBJECTIVE:
  Chief Complaint: [chief_complaint]

  History of Present Illness:
  [Generated HPI narrative from structured data]

  Review of Systems:
  [Generated ROS from positive/negative findings]

  Past Medical History: [from patient record]
  Medications: [from patient record + new mentions]
  Allergies: [from patient record + new mentions]
  Social History: [from patient record + conversation]
  Family History: [from patient record + conversation]

  OBJECTIVE:
  Vital Signs: [structured vitals]
  Physical Exam:
  [Generated exam findings by system]

  ASSESSMENT:
  [Numbered problem list with ICD-10 suggestions]

  PLAN:
  [Numbered plan items grouped by problem]
  ```
- [ ] Implement SOAP note generation with Claude:
  - Input: NLU extraction (T3) + patient context
  - System prompt: "You are a medical documentation assistant..."
  - Generate professional medical prose from structured data
  - Maintain provider's voice/style (learn from previous approved notes)
  - Temperature: 0.3 (slight variation for natural language, but conservative)
- [ ] Implement provider style learning:
  - After 10+ approved notes, analyze provider's writing patterns
  - Store style profile: abbreviation preferences, section ordering, level of detail
  - Apply style profile to future note generation
- [ ] Implement template customization per specialty:
  - Allow practices to customize section ordering
  - Allow adding custom sections (e.g., "Patient Education")
  - Allow default values for normal exam findings
- [ ] Generate multiple note formats:
  - Full SOAP note (default)
  - Brief note (for simple visits)
  - Procedure note (for procedures)
  - Addendum (for additions to existing notes)
- [ ] Implement note quality checks:
  - Completeness: all mentioned findings included
  - Consistency: assessment matches documented findings
  - No hallucination: every statement traceable to transcript
  - Medical accuracy: drug doses, lab values match source

**Acceptance Criteria:**
- [ ] Generated SOAP note reads as professional medical documentation
- [ ] Note includes all findings mentioned in transcript (no omissions)
- [ ] Note does not include findings NOT mentioned in transcript (no hallucination)
- [ ] Specialty-specific templates produce appropriate documentation
- [ ] Provider style learning improves note quality over time
- [ ] Generation time: < 8 seconds per note

---

### T5: Provider Review Interface (Next.js)
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T4
**References:** [[Ambient-AI-Documentation]], [[System-Architecture-Overview]]

**Description:**
Build the provider-facing review interface where providers can review, edit, and approve AI-generated SOAP notes. This is the critical human-in-the-loop step.

**Subtasks:**
- [ ] Build encounter documentation page:
  - Left panel: live/completed transcript with speaker labels
  - Right panel: generated SOAP note (editable)
  - Top bar: patient demographics, encounter info, recording controls
  - Bottom bar: action buttons (approve, edit, regenerate, reject)
- [ ] Implement note editor:
  - Rich text editor (TipTap or Lexical)
  - Section-aware editing (edit only affected section)
  - Inline AI suggestions (highlighted, clickable to accept/reject)
  - Track changes mode (show AI-generated vs. provider-edited)
  - Auto-save every 30 seconds
- [ ] Implement accept/edit/reject workflow:
  ```
  States: draft -> in_review -> approved / rejected

  draft: AI generates note, provider notified
  in_review: provider opens note, timer starts
  approved: provider clicks approve (with or without edits)
  rejected: provider rejects, can provide feedback for improvement
  ```
- [ ] Implement coding suggestions panel:
  - Display ICD-10 suggestions with confidence scores
  - Display CPT suggestions based on documentation
  - Click to accept/reject individual codes
  - Search for additional codes
  - Visual indicators: green (high confidence), yellow (review), red (low confidence)
- [ ] Implement real-time transcript display:
  - Live transcript scrolling during recording
  - Clickable transcript segments to highlight in note
  - Timestamp-to-transcript linking
  - Play audio segment from transcript click
- [ ] Mobile-responsive design for tablet use during encounters
- [ ] Keyboard shortcuts for common actions:
  - `Ctrl+Enter`: Approve note
  - `Ctrl+E`: Toggle edit mode
  - `Ctrl+R`: Regenerate section
  - `Tab`: Next suggestion

**Acceptance Criteria:**
- [ ] Provider can review and approve a note in < 2 minutes
- [ ] All edits tracked and attributable (AI vs. provider)
- [ ] Coding suggestions displayed with confidence scores
- [ ] Real-time transcript updates during recording
- [ ] Works on desktop (Chrome, Firefox) and tablet (iPad Safari)
- [ ] Auto-save prevents data loss on unexpected disconnection

---

### T6: AI Coding Suggestions (ICD-10 + CPT)
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T3, T4
**References:** [[Revenue-Cycle-Deep-Dive]], [[ADR-003-ai-agent-framework]]

**Description:**
Generate ICD-10 diagnosis codes and CPT procedure codes from the clinical documentation. This bridges clinical documentation and billing, reducing coding time and improving accuracy.

**Subtasks:**
- [ ] Implement ICD-10 suggestion engine:
  - Input: NLU extraction (conditions, symptoms, findings)
  - Map conditions to ICD-10-CM codes using:
    - Direct term matching against ICD-10 index
    - Claude-based contextual coding (laterality, severity, specificity)
    - pgvector similarity search against coded encounter corpus
  - Return ranked list with confidence scores
  - Include code description and HCC relevance indicator
  - Handle combination codes and manifestation codes
- [ ] Implement CPT suggestion engine:
  - Input: encounter type + documentation completeness + procedures
  - E/M level determination (99201-99215):
    - Count elements of HPI, ROS, exam
    - Apply MDM (Medical Decision Making) complexity rules
    - Suggest appropriate E/M level with justification
  - Procedure codes: map documented procedures to CPT
  - Include modifier suggestions (25, 59, etc.)
- [ ] Implement coding validation:
  - Check code combinations (exclusion rules)
  - Verify documentation supports code specificity
  - Flag potential upcoding risks
  - Flag missing codes that documentation supports
- [ ] Create coding knowledge base:
  - ICD-10-CM code database (updated annually)
  - CPT code database (updated annually)
  - LCD/NCD coverage rules (common payers)
  - HCC/RAF score lookup
- [ ] Implement feedback loop:
  - Track which suggestions are accepted/rejected by coders
  - Use feedback to improve future suggestions
  - Monthly accuracy reporting

**Acceptance Criteria:**
- [ ] ICD-10 suggestions match human coder in 85%+ of cases
- [ ] CPT E/M level matches human coder in 90%+ of cases
- [ ] Confidence scoring correctly identifies uncertain codes
- [ ] No upcoding suggestions (conservative by default)
- [ ] Suggestions generated within 3 seconds of note approval
- [ ] Coding knowledge base current with 2026 code updates

---

### T7: FHIR Resource Generation from Approved Notes
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T4, T5, [[EPIC-003-fhir-data-layer]] T4 (CRUD), T7 (Bundles)
**References:** [[FHIR-R4-Deep-Dive]], [[ADR-001-fhir-native-data-model]]

**Description:**
When a provider approves a SOAP note, generate and persist the corresponding FHIR resources as a transaction Bundle. The clinical documentation becomes structured, queryable, interoperable data.

**Subtasks:**
- [ ] Define resource generation mapping:
  | Note Section | FHIR Resource(s) |
  |---|---|
  | Chief complaint | Encounter.reasonCode |
  | HPI | Encounter.text (narrative) |
  | Vitals | Observation (vitals category) |
  | Exam findings | Observation (exam category) |
  | Assessment | Condition (each diagnosis) |
  | Plan - medications | MedicationRequest |
  | Plan - diagnostics | ServiceRequest |
  | Plan - referrals | ServiceRequest (referral) |
  | Plan - procedures | Procedure |
  | Allergies (new) | AllergyIntolerance |
  | ICD-10 codes | Condition.code |
  | CPT codes | Claim.item (feed to billing) |
- [ ] Implement resource generator:
  - Create FHIR transaction Bundle from approved note
  - Each resource includes:
    - US Core profile compliance
    - Reference to Encounter
    - Reference to Patient
    - Provenance reference (see T9)
  - Submit as FHIR transaction (EPIC-003 T7)
- [ ] Handle resource updates (amending an existing note):
  - Identify existing resources from previous note version
  - Update rather than duplicate resources
  - Track changes in resource history
- [ ] Implement resource linking:
  - Condition.encounter -> Encounter reference
  - Observation.encounter -> Encounter reference
  - MedicationRequest.encounter -> Encounter reference
  - All resources reference the same Patient
- [ ] Validate generated resources against US Core profiles (EPIC-003 T3)

**Acceptance Criteria:**
- [ ] Approved note generates all appropriate FHIR resources
- [ ] Resources pass US Core validation
- [ ] Resources stored atomically via FHIR transaction
- [ ] Amended notes update existing resources (no duplicates)
- [ ] All generated resources traceable to source note via Provenance

---

### T8: Confidence Scoring and Human-in-the-Loop Triggers
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T3, T4, T6
**References:** [[ADR-003-ai-agent-framework]], [[Ambient-AI-Documentation]]

**Description:**
Implement confidence scoring across the AI pipeline to identify when human review is critical. Low-confidence outputs should trigger explicit review rather than silent errors.

**Subtasks:**
- [ ] Define confidence scoring at each pipeline stage:
  | Stage | Scoring Method | Threshold |
  |---|---|---|
  | Transcription (T2) | Whisper segment confidence | < 0.7 -> flag segment |
  | NLU extraction (T3) | Claude output probability | < 0.8 -> flag section |
  | SOAP note (T4) | Completeness + consistency check | < 0.85 -> require review |
  | ICD-10 coding (T6) | Code match confidence | < 0.75 -> manual coding |
  | CPT coding (T6) | E/M level confidence | < 0.80 -> coder review |
- [ ] Implement aggregate encounter confidence score:
  ```
  encounter_confidence = weighted_avg(
    transcription_confidence * 0.2,
    extraction_confidence * 0.3,
    note_confidence * 0.3,
    coding_confidence * 0.2
  )
  ```
- [ ] Implement HITL (Human-in-the-Loop) triggers:
  - Low transcription confidence: highlight unclear segments in transcript
  - Low extraction confidence: highlight uncertain sections in note with "VERIFY" tag
  - Low coding confidence: present codes with "REVIEW REQUIRED" flag
  - Overall low confidence: do not auto-generate, send to documentation specialist queue
- [ ] Create review queue for documentation specialists:
  - Dashboard showing encounters requiring review
  - Sortable by confidence score, provider, date
  - Action buttons: verify, correct, escalate
- [ ] Implement confidence calibration:
  - Track actual vs. predicted confidence over time
  - Adjust thresholds based on provider feedback
  - Monthly calibration report

**Acceptance Criteria:**
- [ ] Every AI output includes a confidence score
- [ ] Low-confidence outputs are visually flagged in the UI
- [ ] Documentation specialist queue shows encounters needing review
- [ ] No AI-generated content is finalized without human review
- [ ] Confidence calibration demonstrates improvement over 30-day period

---

### T9: Audit Trail - FHIR Provenance
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T7, [[EPIC-002-auth-identity-system]] T8 (audit events)
**References:** [[FHIR-R4-Deep-Dive]], [[HIPAA-Deep-Dive]]

**Description:**
Create FHIR Provenance resources for every AI-generated clinical resource. Provenance establishes the chain of custody: who/what created the resource, from what source, and who approved it.

**Subtasks:**
- [ ] Create FHIR Provenance for AI-generated resources:
  ```json
  {
    "resourceType": "Provenance",
    "target": [{ "reference": "Condition/cond-123" }],
    "recorded": "2026-03-20T14:30:00Z",
    "activity": {
      "coding": [{
        "system": "http://terminology.hl7.org/CodeSystem/v3-DocumentCompletion",
        "code": "AU",
        "display": "authenticated"
      }]
    },
    "agent": [
      {
        "type": { "coding": [{ "code": "assembler", "display": "AI System" }] },
        "who": { "display": "MedOS AI Documentation Engine v1.0" }
      },
      {
        "type": { "coding": [{ "code": "verifier", "display": "Approving Provider" }] },
        "who": { "reference": "Practitioner/pract-456" }
      }
    ],
    "entity": [
      {
        "role": "source",
        "what": { "reference": "DocumentReference/audio-789" },
        "extension": [{
          "url": "http://medos.health/fhir/ai-confidence",
          "valueDecimal": 0.92
        }]
      }
    ]
  }
  ```
- [ ] Create Provenance for each stage:
  - Audio recording -> Transcript (source: audio, agent: Whisper)
  - Transcript -> NLU extraction (source: transcript, agent: Claude NLU)
  - NLU extraction -> SOAP note (source: extraction, agent: Claude generation)
  - SOAP note -> FHIR resources (source: note, agent: resource generator, verifier: provider)
- [ ] Store DocumentReference for audio recording:
  - Reference to S3 location (encrypted)
  - Content type, duration, size
  - Status: current / superseded / entered-in-error
- [ ] Implement provenance chain query:
  - `GET /fhir/Provenance?target=Condition/123` - who created this condition?
  - Trace back to original audio recording
  - Display in UI: "Generated by AI from encounter recording, approved by Dr. Smith"
- [ ] Implement "entered-in-error" workflow:
  - Provider marks resource as entered-in-error
  - Provenance updated to reflect correction
  - Original resource preserved in history (FHIR versioning)
  - All downstream resources flagged for review

**Acceptance Criteria:**
- [ ] Every AI-generated FHIR resource has a corresponding Provenance
- [ ] Provenance chain is queryable from resource back to audio source
- [ ] Approving provider clearly identified in Provenance
- [ ] AI model and version recorded in Provenance agent
- [ ] Confidence score recorded in Provenance entity extension
- [ ] Entered-in-error workflow preserves history and flags downstream resources

---

### T10: Performance Optimization (< 30s Pipeline)
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1-T9
**References:** [[Ambient-AI-Documentation]]

**Description:**
Optimize the end-to-end pipeline to achieve the < 30 second target from audio end to draft note presentation. This is the most critical UX metric.

**Subtasks:**
- [ ] Profile current pipeline latency:
  ```
  Target breakdown (15-minute encounter):
  Audio upload/finalize:     2s
  Transcription (Whisper):  12s
  NLU extraction (Claude):   6s
  SOAP note gen (Claude):    5s
  Coding suggestions:        3s
  FHIR resource prep:        1s
  UI notification:            1s
  ────────────────────────────
  Total target:             30s
  ```
- [ ] Optimize transcription:
  - Use streaming transcription (start processing while recording continues)
  - Parallel transcription of audio chunks
  - Whisper batch processing for GPU utilization
  - Pre-load model in memory (no cold start)
- [ ] Optimize Claude API calls:
  - Pipeline NLU extraction and SOAP generation (can they be one call?)
  - Use Claude streaming for progressive UI updates
  - Optimize prompt length (reduce context, increase cache hits)
  - Pre-fetch patient context during recording (before audio ends)
- [ ] Implement progressive UI updates:
  - Show transcript immediately as it completes
  - Show SOAP note sections as they're generated (streaming)
  - Show coding suggestions as they're calculated
  - Final state: complete note with all suggestions
- [ ] Implement warm-up and caching:
  - Keep Whisper model loaded (ECS min tasks = 1 for GPU)
  - Cache patient context embeddings
  - Cache code suggestion knowledge base in memory
  - Pre-warm Claude with system prompt
- [ ] Load testing:
  - Simulate 10 concurrent encounters
  - Verify < 30s SLA holds under load
  - Identify bottleneck and scaling strategy

**Acceptance Criteria:**
- [ ] End-to-end pipeline completes in < 30 seconds for 15-minute encounter
- [ ] Progressive UI updates show partial results within 5 seconds
- [ ] Pipeline handles 10 concurrent encounters without SLA degradation
- [ ] Cold start (first encounter after idle) completes in < 60 seconds
- [ ] Latency monitoring dashboard shows P50, P95, P99

---

## Dependencies Map

```
T1 (Audio) ──> T2 (Whisper) ──> T3 (NLU) ──> T4 (SOAP Gen)
                                           └──> T6 (Coding)
                                T4 ──> T5 (Review UI)
                                    ──> T7 (FHIR Gen) ──> T9 (Provenance)
                                T3 + T4 + T6 ──> T8 (Confidence)
                                T1-T9 ──> T10 (Performance)
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| AI coding suggestions (T6) | [[EPIC-005-revenue-cycle-mvp]] (charge capture, claims) |
| FHIR resources from notes (T7) | [[EPIC-005-revenue-cycle-mvp]] (Encounter, Condition for claims) |
| Provider review interface (T5) | [[EPIC-006-pilot-readiness]] (training materials, demo) |
| Audio pipeline (T1) | [[EPIC-006-pilot-readiness]] (demo with synthetic data) |
| Confidence scoring (T8) | [[EPIC-006-pilot-readiness]] (quality metrics) |

| This Epic Requires | Provided By |
|---|---|
| ECS Fargate + GPU compute | [[EPIC-001-aws-infrastructure-foundation]] T7 |
| S3 encrypted storage | [[EPIC-001-aws-infrastructure-foundation]] T5 |
| FHIR CRUD API | [[EPIC-003-fhir-data-layer]] T4 |
| FHIR Bundle transactions | [[EPIC-003-fhir-data-layer]] T7 |
| pgvector clinical search | [[EPIC-003-fhir-data-layer]] T9 |
| Event system (encounter events) | [[EPIC-003-fhir-data-layer]] T8 |
| Auth + RBAC | [[EPIC-002-auth-identity-system]] T2, T9 |
| Audit event pipeline | [[EPIC-002-auth-identity-system]] T8 |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Whisper accuracy too low for medical terminology | Medium | Critical | Custom vocabulary boosting; fine-tune on medical conversation dataset; fallback to commercial medical STT (Nuance) |
| Claude hallucination in clinical notes | Medium | Critical | Zero-temperature generation; strict input-output validation; mandatory human review; confidence scoring |
| GPU instance costs exceed budget | High | Medium | Auto-scaling to zero when idle; use spot instances for dev; batch processing during off-peak |
| 30-second latency target not achievable | Medium | High | Progressive UI updates reduce perceived latency; optimize pipeline parallelism; accept 45s as interim target |
| Provider adoption resistance ("I don't trust AI notes") | Medium | High | Show confidence scores; make edits easy; track accuracy metrics; start with tech-forward providers |
| HIPAA concerns with audio recording | Medium | High | Clear patient consent workflow; auto-delete after approval; encryption at every stage; BAA for all components |
| Speaker diarization failure in noisy environments | High | Medium | Offer manual speaker correction in UI; improve with directional microphone recommendations; per-room audio calibration |

---

## Definition of Done

- [ ] Provider can record encounter, receive SOAP note in < 30 seconds
- [ ] Provider can review, edit, and approve note in < 2 minutes
- [ ] AI-generated note matches provider-written note quality in blind comparison
- [ ] ICD-10 coding suggestions match human coder in 85%+ of cases
- [ ] Approved notes generate valid FHIR resources with Provenance
- [ ] Every AI output has confidence scoring
- [ ] All audio PHI encrypted and auto-deleted per retention policy
- [ ] Pipeline handles 10+ concurrent encounters
- [ ] Provider can reject note and provide feedback
- [ ] Audit trail traces every resource back to source audio
