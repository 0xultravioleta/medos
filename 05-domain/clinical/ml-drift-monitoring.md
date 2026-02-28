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

## Related

- [[context-rotting-and-agent-memory]]
- [[ADR-003-langgraph-claude-ai-agents]]
