---
type: moc
tags:
  - moc
  - domain
  - healthcare
---

# Domain Knowledge

> Everything we learn about healthcare: clinical workflows, regulations, standards, billing.
> This is our competitive advantage -- deep domain understanding encoded as a knowledge base.

## By Category

### Clinical
```dataview
LIST
FROM "05-domain/clinical"
SORT file.name ASC
```

### Regulatory & Compliance
```dataview
LIST
FROM "05-domain/regulatory"
SORT file.name ASC
```

### Standards & Interoperability
```dataview
LIST
FROM "05-domain/standards"
SORT file.name ASC
```

### Billing & Revenue Cycle
```dataview
LIST
FROM "05-domain/billing"
SORT file.name ASC
```

## Key Concepts to Master

### Must-Know Standards
- [ ] FHIR R4 -- the data model everything runs on
- [ ] HL7v2 -- legacy but 95% of hospitals still use it
- [ ] X12 EDI -- claims, eligibility, prior auth transactions
- [ ] ICD-10 / CPT -- diagnosis and procedure coding
- [ ] LOINC -- lab test identifiers
- [ ] SNOMED CT -- clinical terminology

### Must-Know Regulations
- [ ] HIPAA (Privacy + Security Rules)
- [ ] 21st Century Cures Act (info blocking)
- [ ] CMS Interoperability Rules (FHIR mandates)
- [ ] TEFCA (national health data exchange)
- [ ] State-level AI in healthcare laws

### Must-Know Business
- [ ] Revenue cycle management (RCM) workflow
- [ ] Prior authorization process
- [ ] Claims adjudication
- [ ] Denial management
- [ ] Value-based care models
- [ ] HCC risk adjustment

## All Domain Notes

```dataview
TABLE WITHOUT ID
  link(file.name) as "Topic",
  category,
  confidence,
  date
FROM "05-domain"
WHERE type = "domain-knowledge"
SORT category ASC, file.name ASC
```

---
*Install the Dataview community plugin to see live queries above.*
