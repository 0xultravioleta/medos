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

## Related

- [[HEALTHCARE_OS_MASTERPLAN]]
- [[ADR-003-langgraph-claude-ai-agents]]
- [[ADR-001-fhir-native-data-model]]
