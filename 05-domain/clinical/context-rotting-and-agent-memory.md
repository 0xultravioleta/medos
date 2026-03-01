---
type: domain
date: "2026-02-28"
status: active
tags:
  - domain
  - ai
  - standards
  - learning
  - phase-1
---

# Context Rotting & Agent Memory Architecture

> Extraido de la sesion de prep de entrevista. Key concepts for MedOS agent design.

## The Problem: Context Rotting

In long-running agent conversations (especially healthcare), context degrades over time:
- Agent starts hallucinating or repeating stale data
- Ignores recent patient history
- Mixes up patients or medications
- Loses track of clinical protocol updates

## Solution: Tiered Memory Architecture

### Short-term: Redis
- In-session memory with short TTL
- Fast but volatile
- Current encounter context, active conversation

### Mid-term: Vector Store (Pinecone/Weaviate)
- Embeddings with re-ranking for relevance
- RAG retrieval without inflating prompt
- Patient history summaries, recent lab results

### Long-term: PostgreSQL + JSONB
- Structured, auditable, HIPAA-compliant
- Full clinical history (FHIR resources)
- The persistent truth store within MedOS

## Context Rot Detection

**Metric:** Cosine similarity between initial prompt context and current context
- Threshold: **< 0.75** triggers refresh
- Monitor continuously during agent sessions
- Implementation: LangChain/LlamaIndex memory decay modules

**Refresh Process:**
1. System detects cosine similarity drop below threshold
2. Queries the **golden source** (EMR -- Epic, Cerner, or MedOS FHIR store)
3. Re-embeds fresh patient data into vector store
4. Prunes stale/irrelevant context from prompt
5. Agent continues with refreshed context

## Golden Source Principle

> The EMR is ALWAYS the source of truth. Everything else (Redis, Pinecone, in-memory) is cache.

- Never trust what the agent "remembers" -- always validate against EMR
- MedOS FHIR store IS the golden source for our system
- All agent responses must be traceable back to FHIR resources

## Context Pruning (not full reset)

Instead of wiping context when it rots:
1. Use a small/fast model (e.g., Haiku) to classify what to keep vs discard
2. Priority: **patient data > clinical protocols > chat history**
3. If context is unsalvageable: pass to human ("safe mode") with templated prompts

## Relevance to MedOS

This architecture maps directly to our stack:
- **Short-term:** Redis 7 (already in docker-compose)
- **Mid-term:** pgvector in PostgreSQL 17 (already provisioned)
- **Long-term:** FHIR JSONB in PostgreSQL (already designed in ADR-001)
- **Agent framework:** LangGraph (ADR-003) with memory management
- **Monitoring:** Structured logging + custom drift metrics via [[Audit Logging]]

## MedOS Implementation Details

### How It Maps to Our Agent Architecture

The tiered memory model is implemented concretely in the MedOS agent framework (see [[agent-architecture]] Section 5):

| Tier | MedOS Component | Agent Usage |
|------|----------------|-------------|
| Short-term | Redis 7 (ElastiCache) | Clinical Scribe holds current transcript + extracted entities during encounter. TTL: 2 hours. |
| Mid-term | pgvector in PostgreSQL 17 | Denial Management Agent queries similar past denials via vector similarity to inform appeal strategy. |
| Long-term | FHIR JSONB in PostgreSQL | Golden source. All 5 agents validate their context against FHIR resources before outputting results. |

### Context Rot in Clinical Documentation

The Clinical Documentation Agent is most vulnerable to context rot during long encounters (30+ minutes):

1. **Symptom:** Agent starts repeating information from the first 10 minutes, ignoring later clinical findings
2. **Detection:** Cosine similarity between the agent's current context summary and a fresh FHIR query drops below 0.75
3. **Mitigation:** LangGraph checkpointer snapshots state every 5 minutes. On refresh, stale context is pruned and re-embedded from the transcript segments

### Context Rot in Revenue Cycle Agents

Prior Auth and Denial Management agents face a different form of context rot:

1. **Payer rule staleness:** Insurance company PA requirements change quarterly. If the agent's payer rule cache is outdated, it may generate incorrect PA forms
2. **Detection:** Compare cached payer rules against the Billing MCP `billing_payer_rules` tool output
3. **Mitigation:** Payer rule embeddings re-indexed weekly via scheduled job. See [[mcp-integration-plan]] Section 3.2 for the Billing MCP tool details

### Confidence Score Integration

Context health directly affects confidence scores in the [[agent-architecture]] Bounded Autonomy Framework:

- Context similarity >= 0.85: No impact on confidence
- Context similarity 0.75-0.85: Confidence score reduced by 10%
- Context similarity < 0.75: Agent pauses, refreshes context, then continues
- Context unsalvageable: Agent enters "safe mode" and escalates to human

This is enforced by the `ContextHealthMonitor` class in the agent framework.

## Related

- [[HEALTHCARE_OS_MASTERPLAN]]
- [[ADR-003-ai-agent-framework]]
- [[ADR-001-fhir-native-data-model]]
- [[agent-architecture]] -- Section 5: Memory Architecture
- [[mcp-integration-plan]] -- MCP servers that provide fresh context
- [[ml-drift-monitoring]] -- Related: model output drift detection
- [[EPIC-007-mcp-sdk-refactoring]] -- MCP tools agents use to refresh context
- [[EPIC-008-demo-polish]] -- Sprint 3 agent runner and workflow orchestration
