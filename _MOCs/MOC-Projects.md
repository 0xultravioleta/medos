---
type: moc
tags:
  - moc
  - projects
---

# Projects

> Active epics, sprints, and feature work for MedOS.

## Active Projects

```dataview
TABLE WITHOUT ID
  link(file.name) as "Project",
  status,
  priority,
  owner,
  target-date as "Target"
FROM "03-projects"
WHERE status != "completed" AND status != "archived"
SORT priority ASC
```

## Completed Projects

```dataview
TABLE WITHOUT ID
  link(file.name) as "Project",
  status,
  date
FROM "03-projects"
WHERE status = "completed"
SORT date DESC
```

## 90-Day Roadmap (from Master Plan)

Reference: [[HEALTHCARE_OS_MASTERPLAN]]

### Week 1-2: Foundation
- [ ] AWS infrastructure (Terraform, HIPAA BAA, encryption)
- [ ] CI/CD pipeline
- [ ] FastAPI skeleton + FHIR models + PostgreSQL schema
- [ ] Auth system (Auth0/Clerk with HIPAA BAA)

### Week 3-6: Core MVP
- [ ] FHIR data ingestion pipeline
- [ ] Patient identity matching engine
- [ ] AI clinical documentation agent
- [ ] Provider review interface (Next.js)
- [ ] AI coding engine (ICD-10 + CPT)

### Week 7-10: Revenue Module
- [ ] Prior auth automation
- [ ] Eligibility verification (X12 270/271)
- [ ] Claims generation
- [ ] AI denial prediction
- [ ] Analytics dashboard

### Week 11-13: Pilot Ready
- [ ] Security hardening + pen testing
- [ ] Pilot onboarding workflow
- [ ] Training materials + demo environment

---
*Install the Dataview community plugin to see live queries above.*
