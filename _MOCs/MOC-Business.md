---
type: moc
tags:
  - moc
  - business
---

# Business

> Go-to-market, pricing, hiring, capital deployment, and competitive intelligence.

## Strategy Documents

```dataview
LIST
FROM "07-business"
SORT file.name ASC
```

## Key Numbers (from Master Plan)

Reference: [[HEALTHCARE_OS_MASTERPLAN]]

### Revenue Targets
| Month | MRR | ARR | Customers |
|-------|-----|-----|-----------|
| 6 | $30K | $360K | 5 pilots |
| 12 | $100K | $1.2M | 15-20 |
| 18 | $300K | $3.6M | 40-50 |
| 24 | $700K | $8.4M | 80-100 |
| 36 | $3M | $36M | 300-400 |

### Capital Deployment ($100M / 36mo)
| Phase | Period | Budget |
|-------|--------|--------|
| Foundation | Mo 0-12 | $8-12M |
| PMF | Mo 12-24 | $25-35M |
| Expansion | Mo 24-36 | $30-45M |
| Reserve | -- | $15-25M |

### Target Market
- **Segment:** Mid-size specialty practices (5-30 providers)
- **Geography:** Florida first
- **Specialty:** Orthopedics or Dermatology
- **Pricing:** $2K-50K/mo depending on tier

## Hiring Plan

| # | Role | When |
|---|------|------|
| 3 | Healthcare Domain Expert | Mo 3-4 |
| 4 | Senior Full-Stack Engineer | Mo 5-6 |
| 5 | Compliance & Security Lead | Mo 6-8 |
| 6 | Product Designer (Healthcare UX) | Mo 8-10 |
| 7 | Sales Engineer | Mo 10-12 |

## Meetings & Interactions

```dataview
TABLE WITHOUT ID
  link(file.name) as "Meeting",
  date,
  attendees
FROM "02-meetings"
SORT date DESC
LIMIT 10
```

## Retrospectives

```dataview
LIST
FROM "08-retrospectives"
SORT file.name DESC
```

---
*Install the Dataview community plugin to see live queries above.*
