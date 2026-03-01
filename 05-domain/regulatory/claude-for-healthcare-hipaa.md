---
type: domain
date: "2026-02-28"
status: active
tags:
  - domain
  - regulatory
  - hipaa
  - ai
  - decision
  - phase-1
---

# Claude for Healthcare (Anthropic HIPAA Announcement)

> Critical data point for MedOS: Anthropic's Claude is HIPAA-ready for production healthcare.

## Announcement Details

- **Date:** January 11, 2026
- **Event:** J.P. Morgan Healthcare Conference (JPM26)
- **Context:** Announced days after OpenAI's ChatGPT Health (Jan 7, 2026)

## What It Includes

### HIPAA Compliance
- **Business Associate Agreement (BAA)** signed with healthcare clients
- For providers, payers, and health tech companies
- Zero-data-training: prompts and patient data are NOT used to train the model
- Opt-in total: can disconnect at any time

### Infrastructure Availability
- **AWS Bedrock** -- full Terraform support
- **Google Vertex AI** -- managed deployment
- **Microsoft Azure** -- also available
- Claude is the **only frontier model available on all three clouds** with HIPAA compliance

### Healthcare-Specific Features
- EMR connectors (via HealthEx for medical records)
- FHIR integration
- PubMed access for clinical evidence
- ICD-10 coding assistance
- Prior authorization automation
- Claims appeals support

### Partners
- Sanofi (pharma)
- Veeva (life sciences CRM)
- Commure (health tech infrastructure)

## Terraform Resources for Bedrock

```hcl
# Key Terraform resources for MedOS production deployment
aws_bedrock_model_invocation_logging_configuration  # Audit logs
aws_bedrock_custom_model                            # Fine-tuning
aws_bedrock_agent                                   # RAG + memory + tools
# Plus: KMS keys, IAM roles, VPC endpoints for security
```

## Implications for MedOS

### Production AI Stack
1. **Model:** Claude Opus 4.6 (or Sonnet for cost-sensitive tasks) via AWS Bedrock
2. **Compliance:** BAA through Bedrock -- HIPAA handled at infra level
3. **Integration:** FHIR connectors align perfectly with our FHIR-native architecture (ADR-001)
4. **IaC:** Full Terraform provisioning fits our no-CloudFormation rule

### What This Means
- We do NOT need to build custom compliance layers for the AI model
- Bedrock handles encryption, audit logging, data residency
- We focus on the agent logic (LangGraph) and clinical workflows
- Cost: Enterprise pricing (sales-assisted), not per-seat

### For the Demo
- Can reference this in conversations with Dr. Di Reze
- Shows MedOS is built on the RIGHT foundation -- production-ready AI infra exists
- Differentiator: most healthcare startups don't know about Claude for Healthcare yet

## GDPR Note

For European expansion:
- No official "GDPR-ready" stamp from Anthropic
- Must deploy in EU region with data residency
- De-identification and consent required
- More complex than HIPAA but achievable with proper setup

## MedOS Integration Architecture

### How MedOS Uses Claude via Bedrock

The MedOS agent framework (see [[agent-architecture]]) calls Claude through a layered stack:

```
LangGraph Agent -> MCP Gateway -> HIPAAFastMCP -> Claude via Bedrock API
```

Key integration points:
- **HIPAAFastMCP** (see [[ADR-005-mcp-sdk-integration]]) overrides `call_tool()` to inject security pipeline before every Claude call
- **Audit logging** captures every LLM invocation as a FHIR `Provenance` resource (see [[agent-architecture]] Section 2.1)
- **Credential injection** retrieves Bedrock API credentials from AWS Secrets Manager at runtime -- never hardcoded
- **PHI handling** ensures patient data sent to Claude stays within the BAA boundary (Bedrock endpoint in VPC via PrivateLink)

### Claude Model Selection per Agent

| Agent | Recommended Model | Rationale |
|-------|------------------|-----------|
| Clinical Scribe | Claude Opus 4.6 | Highest accuracy for clinical NLU and SOAP note generation |
| Prior Auth | Claude Sonnet 4.6 | Good balance of speed and quality for form generation |
| Denial Management | Claude Sonnet 4.6 | Appeal letters need quality but also fast turnaround |
| Patient Communication | Claude Haiku 4.5 | High volume, simple responses, cost-sensitive |
| Quality Reporting | Claude Sonnet 4.6 | Analytical tasks, report generation |

Model selection is configurable per tenant via `TenantAgentConfig` (see [[agent-architecture]] Section 4).

### Bedrock vs Direct API

MedOS uses Bedrock (not direct Anthropic API) because:
1. BAA is through AWS, covering all data in transit and at rest
2. VPC endpoints (PrivateLink) keep PHI within our network boundary
3. CloudWatch integration for LLM call monitoring
4. IAM-based access control (per-tenant roles)
5. Terraform-managed infrastructure aligns with our no-CloudFormation rule

### Cost Management

- **Token budgets** per agent type per tenant (prevent runaway costs)
- **Caching** of common prompts via Bedrock's prompt caching
- **Model routing** sends simple tasks to Haiku, complex tasks to Opus
- **Monitoring** via Langfuse: cost per encounter, cost per agent type

## Related

- [[HEALTHCARE_OS_MASTERPLAN]]
- [[ADR-003-ai-agent-framework]]
- [[HIPAA-Deep-Dive]]
- [[AWS-HIPAA-Infrastructure]]
- [[agent-architecture]] -- Agent framework consuming Claude
- [[ADR-005-mcp-sdk-integration]] -- HIPAAFastMCP security pipeline
- [[mcp-integration-plan]] -- MCP tools powered by Claude
- [[ml-drift-monitoring]] -- Drift management for Claude-based agents
- [[EPIC-007-mcp-sdk-refactoring]] -- Sprint 2: agents using Claude
- [[EPIC-008-demo-polish]] -- Sprint 3: agent runner API
