---
type: moc
tags:
  - moc
  - architecture
---

# Architecture

> System design, ADRs, and technical infrastructure for MedOS.

## Architecture Decision Records (ADRs)

```dataview
TABLE WITHOUT ID
  link(file.name) as "ADR",
  status,
  date,
  deciders
FROM "04-architecture/adr"
SORT date DESC
```

## System Design Documents

```dataview
LIST
FROM "04-architecture/system-design"
SORT file.name ASC
```

## Diagrams

```dataview
LIST
FROM "04-architecture/diagrams"
SORT file.name ASC
```

## The 8 Modules (from Master Plan)

Reference: [[HEALTHCARE_OS_MASTERPLAN]]

| Module | Purpose |
|--------|---------|
| A. Patient Identity | Universal patient identity, FHIR-native data |
| B. Workflow Engine | Ambient AI docs, scheduling, CDS |
| C. Revenue Cycle | AI coding, claims, prior auth, denials |
| D. Payer Integration | Clearinghouse, X12 EDI, contracts |
| E. Population Health | HCC scoring, readmission prediction |
| F. Patient Engagement | Portal, chat, RPM, telehealth |
| G. Compliance | Auth, RBAC, audit trail, encryption |
| H. Integration | FHIR R4, HL7v2, marketplace, SDK |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | FastAPI (Python 3.12+) |
| Frontend | Next.js 15 |
| Primary DB | PostgreSQL 17 + pgvector |
| Time-series | TimescaleDB |
| Cache | Redis 7+ |
| Events | Kafka / EventBridge |
| LLM | Claude API (HIPAA BAA) |
| Agents | LangGraph |
| Integration | MCP (Model Context Protocol) |
| Speech | Whisper v3 (self-hosted) |
| Cloud | AWS (HIPAA BAA) |
| IaC | Terraform |

## Engineering Standards

```dataview
LIST
FROM "06-engineering"
SORT file.name ASC
```

---
*Install the Dataview community plugin to see live queries above.*
