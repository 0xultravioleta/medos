---
type: moc
tags:
  - moc
  - home
aliases:
  - Dashboard
  - Index
---

# MedOS - Home

> Healthcare OS: AI-Native Operating System for U.S. Healthcare

## Quick Navigation

| Area | Link |
|------|------|
| Master Plan | [[HEALTHCARE_OS_MASTERPLAN]] |
| Architecture | [[MOC-Architecture]] |
| AI Agent Architecture | [[MOC-Agent-Architecture]] |
| Domain Knowledge | [[MOC-Domain-Knowledge]] |
| Projects | [[MOC-Projects]] |
| Business | [[MOC-Business]] |

## Recent Daily Notes

```dataview
TABLE WITHOUT ID
  link(file.name) as "Date",
  file.frontmatter.tags as "Tags"
FROM "01-daily"
SORT file.name DESC
LIMIT 7
```

## Active Projects

```dataview
TABLE WITHOUT ID
  link(file.name) as "Project",
  status,
  priority,
  owner
FROM "03-projects"
WHERE status != "completed" AND status != "archived"
SORT priority ASC
```

## Recent Architecture Decisions

```dataview
TABLE WITHOUT ID
  link(file.name) as "ADR",
  status,
  date
FROM "04-architecture/adr"
SORT date DESC
LIMIT 5
```

## Open Bugs

```dataview
TABLE WITHOUT ID
  link(file.name) as "Bug",
  severity,
  status,
  module
FROM ""
WHERE type = "bug" AND status != "resolved"
SORT severity ASC
```

## Inbox (Unsorted Notes)

```dataview
LIST
FROM "00-inbox"
SORT file.ctime DESC
```

---
*Install the Dataview community plugin to see live queries above.*
