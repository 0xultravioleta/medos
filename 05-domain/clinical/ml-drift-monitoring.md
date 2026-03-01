---
type: domain
date: "2026-02-28"
status: active
tags:
  - domain
  - ai
  - standards
  - learning
---

# ML Drift Monitoring in Healthcare

> Key concept for AI Ops role. How to detect and handle model degradation.

## Types of Drift

### Data Drift (input distribution changes)
- **KS Test (Kolmogorov-Smirnov):** Measures if new data distribution differs from training
  - Alert threshold: score > 0.2
- **PSI (Population Stability Index):** Common in finance and healthcare
  - Alert threshold: > 0.25 = strong drift

### Concept Drift (what the model predicts changes)
- **Accuracy decay:** Compare model precision on new data vs validation data
  - Alert threshold: 10-15% drop
- **F1-score drift:** Especially for classification (cancer detection, diagnosis)

### Feature Drift (individual variables shift)
- **Feature importance shift:** e.g., "age" or "blood pressure" no longer weight the same
- Can indicate protocol changes or population shifts

### Output Drift (model produces nonsense)
- **Hallucination rate:** How often the LLM invents medical data
- **Confidence score drop:** Model uncertainty increasing = something wrong

## Monitoring Stack

| Tool | Purpose |
|------|---------|
| Datadog / Prometheus | Metrics collection (KS/PSI every hour) |
| Grafana | Visualization dashboards |
| Slack / PagerDuty | Alerting when thresholds breached |
| SageMaker / MLflow | Retraining pipeline |

## Response Protocol

1. **Detect:** KS/PSI metrics exceed threshold
2. **Alert:** Automated notification to team
3. **Collect:** Gather fresh data from EMR (golden source)
4. **Retrain:** Incremental fine-tuning (LoRA, not full retrain) with last 30 days
5. **Canary release:** Test on 5% of users first
6. **Compare:** New model vs old model on same data
7. **Deploy:** If metrics improve, roll out to 100%

Key principle: **No downtime during retraining.** Online learning or shadow deployment.

## Relevance to MedOS

- Phase 3+ concern (after MVP)
- Monitoring infra: Prometheus + Grafana (already standard in our AWS stack)
- Models: Claude via Bedrock (Anthropic handles base model drift)
- Our responsibility: drift in fine-tuned layers and RAG retrieval quality

## MedOS Implementation Details

### Which Agents Are Affected

Each MedOS agent (see [[agent-architecture]]) has different drift exposure:

| Agent | Primary Drift Risk | Monitoring Approach |
|-------|-------------------|-------------------|
| Clinical Scribe | Transcription accuracy decay (Whisper), SOAP note quality regression | Compare AI-generated codes vs human-approved codes over time |
| Prior Auth | Payer rule changes causing incorrect PA determinations | Track PA approval rate by payer + code combination monthly |
| Denial Management | Appeal success rate decay as payer algorithms change | Track appeal success rate vs historical baseline |
| Patient Communication | Response quality degradation, misrouting | Monitor escalation rate and patient satisfaction signals |
| Quality Reporting | Measure calculation errors from data distribution changes | Validate against known test populations quarterly |

### Drift Metrics in MedOS

These are the metrics tracked via Langfuse and CloudWatch (see [[agent-architecture]] Section 6):

| Metric | Description | Alert Threshold | Action |
|--------|-------------|-----------------|--------|
| `agent.confidence.mean` | Rolling 7-day average confidence per agent | < 0.80 | Investigate model drift |
| `agent.correction.rate` | % of auto-approved outputs later corrected by humans | > 5% | Lower auto-approve threshold |
| `agent.escalation.rate` | % of actions escalated to humans | > 40% | Model may be underperforming |
| `coding.accuracy` | AI ICD-10/CPT suggestions vs human-finalized codes | < 85% | Retrain prompt engineering |
| `pa.approval.rate` | Prior auth approval rate (our submissions vs payer decisions) | Drop > 15% from baseline | Review payer rule cache freshness |

### Retraining in a BAA Environment

Since MedOS uses Claude via AWS Bedrock (see [[claude-for-healthcare-hipaa]]), our drift response differs from self-hosted models:

1. **We do NOT retrain Claude** -- Anthropic manages the base model
2. **We DO tune prompts** -- System prompts, few-shot examples, and RAG retrieval quality
3. **We DO retrain embeddings** -- pgvector embeddings for patient history, payer rules, denial patterns
4. **We DO update payer rules** -- Weekly refresh of payer-specific PA requirements and coding rules

### Canary Releases for Prompt Changes

When drift is detected and prompt engineering is updated:

1. Deploy updated prompts to 5% of agent invocations
2. Compare confidence scores and correction rates vs control group
3. If improvement >= 5%: roll out to 100%
4. If regression: revert immediately
5. All variants logged in Langfuse for A/B analysis

## Related

- [[context-rotting-and-agent-memory]]
- [[ADR-003-ai-agent-framework]]
- [[agent-architecture]] -- Agent metrics and observability (Section 6)
- [[claude-for-healthcare-hipaa]] -- BAA and model management
- [[EPIC-007-mcp-sdk-refactoring]] -- MCP tools for data access
- [[mcp-integration-plan]] -- Analytics MCP server for drift monitoring
