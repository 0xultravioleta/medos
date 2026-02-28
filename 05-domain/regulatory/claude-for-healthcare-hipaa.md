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

## Related

- [[HEALTHCARE_OS_MASTERPLAN]]
- [[ADR-003-langgraph-claude-ai-agents]]
- [[HIPAA Compliance Deep Dive]]
- [[AWS HIPAA Infrastructure]]
