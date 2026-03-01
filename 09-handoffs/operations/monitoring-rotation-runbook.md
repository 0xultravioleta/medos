---
type: operations
date: "2026-02-28"
tags:
  - operations
  - monitoring
  - pilot
  - launch
  - phase-1
---

# Monitoring Rotation Runbook

## Purpose

Daily monitoring procedures for MedOS pilot launch. This runbook ensures 24/7 awareness of system health, AI output quality, and cost management during the critical first weeks of pilot operation. Every team member on rotation follows these exact procedures -- no guessing, no shortcuts.

**References:** [[Incident-Response-Playbook]], [[HIPAA-Risk-Assessment]], [[EPIC-011-launch-go-live]]

---

## Daily Checklist

### Morning Check (8:00 AM ET)

- [ ] Review overnight error rates in CloudWatch dashboard
- [ ] Check failed background jobs (audio transcription queue, claims submission queue)
- [ ] Verify nightly backups completed successfully (RDS snapshots, S3 replication)
- [ ] Review any P1/P2 alerts that fired overnight
- [ ] Check SSL certificate expiration (must be > 30 days)
- [ ] Verify all ECS services running with expected task count
- [ ] Review Redis memory usage and eviction rate
- [ ] Check disk usage on all volumes (alert threshold: 80%)

### Mid-Day Check (12:00 PM ET)

- [ ] Check API latency trends (P50, P95, P99) -- flag if P99 > 2s
- [ ] Review AI output quality: sample 5 SOAP notes for accuracy
- [ ] Monitor AWS cost dashboard -- compare to daily budget baseline
- [ ] Check LLM API token usage (Claude via Bedrock) -- flag unusual spikes
- [ ] Review patient matching accuracy (false positive rate < 2%)
- [ ] Check event bus (EventBridge) for dead-letter queue messages
- [ ] Verify tenant isolation: spot-check 2 cross-tenant queries return empty

### End of Day Check (5:00 PM ET)

- [ ] Generate daily monitoring report (template below)
- [ ] Review all support tickets opened today -- triage by severity
- [ ] Update status page if any degraded services
- [ ] Review audit logs for unusual access patterns (bulk PHI exports, off-hours access)
- [ ] Confirm next day's on-call rotation is aware and available
- [ ] Log any anomalies in `#medos-pilot-support` Slack channel

---

## What to Monitor

### API Health

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Error rate (5xx) | < 1% | > 3% | > 5% |
| P99 latency | < 1s | > 1.5s | > 2s |
| P50 latency | < 200ms | > 500ms | > 1s |
| Throughput (req/min) | baseline +/- 30% | > 50% deviation | > 80% deviation |
| Auth failures | < 5/hour | > 10/hour | > 50/hour |

### AI Quality

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Coding accuracy (ICD-10/CPT) | > 92% | < 90% | < 85% |
| SOAP note quality (sampled) | > 90% acceptable | < 85% | < 80% |
| Confidence score avg | > 0.90 | < 0.87 | < 0.85 |
| Human review rate | < 15% | > 20% | > 30% |
| Transcription WER | < 8% | > 10% | > 15% |

### Infrastructure

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| CPU utilization | < 60% | > 75% | > 90% |
| Memory utilization | < 70% | > 80% | > 90% |
| DB connections | < 60% pool | > 75% pool | > 90% pool |
| Redis hit rate | > 90% | < 85% | < 75% |
| Disk usage | < 70% | > 80% | > 90% |
| ECS task health | all healthy | 1 unhealthy | > 1 unhealthy |

### Cost Tracking

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| AWS daily spend | < $50/day | > $75/day | > $100/day |
| LLM token usage | < 2M tokens/day | > 3M tokens/day | > 5M tokens/day |
| Projected monthly cost | < $1,500 | > $2,000 | > $3,000 |
| Data transfer out | < 10 GB/day | > 20 GB/day | > 50 GB/day |

---

## Alert Response Procedures

| Alert Name | Severity | Response Action | Escalation |
|------------|----------|-----------------|------------|
| API Error Rate > 5% | P1 | Check ECS logs, verify DB connectivity, check recent deployments. Rollback if deploy-related. | Both on-call immediately |
| P99 Latency > 2s | P2 | Check DB query performance, Redis cache status, ECS CPU/memory. Scale if needed. | Primary on-call |
| AI Confidence < 0.85 avg | P2 | Review recent SOAP notes, check model endpoint health, verify prompt templates unchanged. | Primary on-call + AI lead |
| Disk Usage > 90% | P2 | Identify largest files/tables, clean old logs, expand volume if needed. | Primary on-call |
| Backup Failure | P2 | Re-trigger backup manually, verify RDS snapshot status, check S3 replication. | Primary on-call |
| Auth Failures > 50/hr | P1 | Check for brute force attack, review IP origins, enable enhanced rate limiting. HIPAA incident if credential stuffing suspected. | Both on-call + security review |
| ECS Task Crash Loop | P1 | Check container logs, verify environment variables, check health check endpoint. Replace task. | Both on-call immediately |
| Cost Spike > 2x baseline | P3 | Review CloudWatch billing dashboard, identify source (LLM tokens, data transfer, compute). | Next business day |
| Dead Letter Queue > 0 | P3 | Review failed messages, identify root cause, replay if safe. | Next business day |
| SSL Cert < 7 days | P2 | Trigger certificate renewal via ACM, verify DNS validation. | Primary on-call |

---

## Weekly Metrics Review Template

```markdown
# Weekly Metrics Review - Week of YYYY-MM-DD

## API Performance
| Metric | This Week | Last Week | Trend |
|--------|-----------|-----------|-------|
| Avg error rate | | | |
| P99 latency | | | |
| Total requests | | | |
| Unique users | | | |

## AI Quality
| Metric | This Week | Last Week | Trend |
|--------|-----------|-----------|-------|
| Avg confidence score | | | |
| Human review rate | | | |
| Notes generated | | | |
| Coding accuracy (sampled) | | | |

## Infrastructure
| Metric | This Week | Last Week | Trend |
|--------|-----------|-----------|-------|
| Peak CPU | | | |
| Peak memory | | | |
| DB connections peak | | | |
| Incidents (P1/P2) | | | |

## Cost
| Category | This Week | Last Week | Trend |
|----------|-----------|-----------|-------|
| AWS compute | | | |
| LLM API | | | |
| Data transfer | | | |
| Total | | | |

## Action Items
- [ ] Item 1
- [ ] Item 2
```

---

## First Week Rotation Schedule

| Day | Primary On-Call | Secondary On-Call | Notes |
|-----|----------------|-------------------|-------|
| Monday (Launch) | Person A | Person B | Both on-site/available. Full day go-live support. |
| Tuesday | Person A | Person B | Morning: review Day 1 issues. Afternoon: normal rotation. |
| Wednesday | Person B | Person A | Mid-week check: AI quality deep review (20 notes). |
| Thursday | Person A | Person B | Cost review day: verify projections vs actuals. |
| Friday | Person B | Person A | Weekly metrics review. Prepare weekend handoff. |
| Saturday | Person A (on-call) | Person B (backup) | Reduced checks: morning + evening only. |
| Sunday | Person B (on-call) | Person A (backup) | Reduced checks: morning + evening only. |

**On-call expectations:**
- Primary: respond to P1 within 15 minutes, P2 within 1 hour
- Secondary: respond to P1 within 30 minutes if primary unavailable
- Weekend: morning check (9 AM) + evening check (7 PM) minimum

---

## Escalation Path

| Severity | Definition | Response Time | Who | Action |
|----------|-----------|---------------|-----|--------|
| **P1 - Critical** | System down, data loss risk, PHI exposure, all users affected | 15 min | Both on-call immediately | War room, all hands. HIPAA breach assessment if PHI involved. |
| **P2 - High** | Feature degraded, AI quality drop, single service down, subset of users affected | 1 hour | Primary on-call | Investigate and fix. Escalate to P1 if not resolved in 2 hours. |
| **P3 - Medium** | Minor feature issue, cosmetic bug, performance slightly degraded | Next business day | Primary on-call | Triage during daily check. Schedule fix for next sprint. |
| **P4 - Low** | Feature request, improvement suggestion, documentation update | 1 week | Triage in weekly review | Add to backlog. Prioritize in next sprint planning. |

**Escalation contacts:**
1. Primary on-call (per rotation schedule above)
2. Secondary on-call (backup)
3. Dr. Di Reze (technical operations) -- P1 only, after initial assessment
4. AWS Support (Business tier) -- infrastructure issues beyond team capability

---

## Daily Report Template

```markdown
# MedOS Daily Monitoring Report - YYYY-MM-DD

**On-call:** [Name]
**System Status:** GREEN / YELLOW / RED

## Summary
- Requests served: X
- Error rate: X%
- P99 latency: Xms
- SOAP notes generated: X
- AI avg confidence: X.XX
- AWS spend today: $X.XX

## Issues
| # | Severity | Description | Status | Resolution |
|---|----------|-------------|--------|------------|
| 1 | | | | |

## AI Quality Spot Check (5 notes)
| Note ID | Confidence | Manual Review | Quality |
|---------|------------|---------------|---------|
| | | Pass/Fail | Good/Fair/Poor |

## Action Items for Tomorrow
- [ ] Item 1
```
