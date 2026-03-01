---
type: handoff
date: "2026-02-28"
tags:
  - handoff
  - frontend
  - project-management
  - sprint-board
---

# Handoff: Sprint Board / Project Tracker for MedOS

> **From:** Sprint 3 session (Device/Context/System pages completed)
> **To:** New session — build project tracking board
> **Priority:** High — this gives visibility into all 140 tasks across 14 EPICs

---

## Context

MedOS is a Healthcare OS platform. The frontend is Next.js 15 (App Router) deployed on Vercel. We just completed Sprint 3 which added 3 new Settings sub-pages. The entire project has 140 tasks across 9 sprints and 14 EPICs — all documented in markdown but with NO visual tracker.

**The ask:** Build a project tracking board (like Linear/Jira) as a page inside MedOS itself. All data is inline mock (matching the real execution plan). This is Sprint 3F-T10 essentially.

---

## Tech Stack & Patterns

- **Framework:** Next.js 15 (App Router), TypeScript strict
- **Styling:** Tailwind CSS with MedOS design tokens (CSS variables)
- **Pattern:** "use client", inline mock data, tabbed interfaces, lucide-react icons
- **Reference files:**
  - `frontend/src/app/(dashboard)/settings/practice/page.tsx` — 650 lines, tabbed page pattern
  - `frontend/src/app/(dashboard)/settings/system/page.tsx` — 822 lines, latest pattern with collapsible sections
  - `frontend/src/app/(dashboard)/settings/context/page.tsx` — 743 lines, table + cards + filters
  - `frontend/src/components/sidebar.tsx` — NAV_ITEMS array (add new route here)

### MedOS CSS Variables (MUST use these, not raw colors)
```
--medos-primary          (blue)
--medos-primary-hover    (darker blue)
--medos-primary-light    (light blue bg)
--medos-navy             (dark text)
--medos-gray-50 through --medos-gray-900
shadow-medos-sm          (card shadow)
```

### Status Badge Pattern
```tsx
// Emerald for done/active
"bg-emerald-50 text-emerald-700 border border-emerald-200"
// Amber for in-progress/warning
"bg-amber-50 text-amber-700 border border-amber-200"
// Red for blocked/critical
"bg-red-50 text-red-700 border border-red-200"
// Blue for pending/info
"bg-blue-50 text-blue-700 border border-blue-200"
// Gray for not started
"bg-gray-50 text-gray-600 border border-gray-200"
```

---

## Page Specification

### Route: `/project`

Add to sidebar NAV_ITEMS in `frontend/src/components/sidebar.tsx` after the existing items. Use `LayoutGrid` or `Kanban` icon from lucide-react. Label: "Project".

### File: `frontend/src/app/(dashboard)/project/page.tsx`

Estimated: ~800-1000 lines (it's a data-heavy page)

---

## Layout

### Header
- Icon + "Project Tracker" title
- Subtitle: "MedOS Phase 1 — Foundation to Pilot (90 days, 140 tasks)"
- View toggle buttons: Board | List | Timeline | Stats

### 4 Views (tabs/toggle)

---

### View 1: Board (Kanban)

3 columns: **Pending** | **In Progress** | **Done**

Each card shows:
- Task ID badge (e.g., `S0-T01`) with sprint color
- Task title (truncated to 1 line)
- Owner badge (A or B or BOTH)
- EPIC tag (small colored badge)
- Estimated hours

Column headers show count + total hours.

Sprint filter dropdown at top (All, S0, S1, S2, S3, S4, S5, S6, S2.5, S3F).
EPIC filter dropdown (All, EPIC-001 through EPIC-014).

Cards should be visually compact — think Linear's density.

---

### View 2: List (Table)

Full table with ALL 140 tasks:

| ID | Task | Sprint | EPIC | Owner | Est. | Status | Deps |
|----|------|--------|------|-------|------|--------|------|

- Sortable columns (click header to sort)
- Search/filter bar at top
- Status badges: `pending` (gray), `in-progress` (amber), `done` (emerald)
- Sprint badges with distinct colors per sprint
- Expandable rows showing full acceptance criteria

---

### View 3: Timeline (Gantt-lite)

Horizontal timeline with sprints as swim lanes:

```
S0  [============================] W1-W2 (43 tasks) ✓
S1  [============================] W3-W4 (15 tasks) ✓
S2  [============================] W5-W6 (14 tasks) ✓
S3  [============================] W7-W8 (12 tasks) ✓
S4  [============================] W9-W10 (18 tasks) ✓
S5  [============================] W11-W12 (12 tasks) ✓
S6  [=============               ] W13 (9 tasks) ◐
S2.5[============================] (8 tasks) ✓
S3F [============================] (9 tasks) ✓
```

Each bar shows: sprint name, date range, task count, completion %.
Color: emerald if done, amber if in-progress, gray if pending.
Milestone diamonds on the timeline for M1-M7.

---

### View 4: Stats (Dashboard)

**Row 1: 4 KPI cards**
1. Total Tasks: 140 (breakdown: 126 done, 5 in-progress, 9 pending)
2. Completion: 90% (progress ring)
3. EPICs: 14 (12 done, 2 in-progress)
4. Days Remaining: calculated from 2026-05-29

**Row 2: Sprint Velocity**
Bar chart showing tasks completed per sprint:
- S0: 43, S1: 15, S2: 14, S3: 12, S4: 18, S5: 12, S6: 7/9, S2.5: 8, S3F: 9

**Row 3: EPIC Progress**
Horizontal progress bars for each EPIC (001-014):
- Name, task count, completion %, status badge

**Row 4: Team Allocation**
- Person A: 77 tasks (55%), Person B: 53 tasks (38%), Both: 10 tasks (7%)
- Pie chart or segmented bar

---

## Mock Data

### Sprint Colors (for badges and timeline)
```typescript
const SPRINT_COLORS: Record<string, { bg: string; text: string; border: string }> = {
  "S0": { bg: "bg-violet-50", text: "text-violet-700", border: "border-violet-200" },
  "S1": { bg: "bg-sky-50", text: "text-sky-700", border: "border-sky-200" },
  "S2": { bg: "bg-teal-50", text: "text-teal-700", border: "border-teal-200" },
  "S3": { bg: "bg-orange-50", text: "text-orange-700", border: "border-orange-200" },
  "S4": { bg: "bg-pink-50", text: "text-pink-700", border: "border-pink-200" },
  "S5": { bg: "bg-lime-50", text: "text-lime-700", border: "border-lime-200" },
  "S6": { bg: "bg-amber-50", text: "text-amber-700", border: "border-amber-200" },
  "S2.5": { bg: "bg-cyan-50", text: "text-cyan-700", border: "border-cyan-200" },
  "S3F": { bg: "bg-indigo-50", text: "text-indigo-700", border: "border-indigo-200" },
};
```

### Epic Data
```typescript
const EPICS = [
  { id: "EPIC-001", name: "AWS Infrastructure Foundation", sprint: "S0", status: "done", tasks: 17, done: 17 },
  { id: "EPIC-002", name: "Auth & Identity System", sprint: "S0", status: "done", tasks: 14, done: 14 },
  { id: "EPIC-003", name: "FHIR Data Layer", sprint: "S1", status: "done", tasks: 15, done: 15 },
  { id: "EPIC-004", name: "AI Clinical Documentation", sprint: "S2", status: "done", tasks: 14, done: 14 },
  { id: "EPIC-005", name: "Revenue Cycle MVP", sprint: "S3", status: "done", tasks: 12, done: 12 },
  { id: "EPIC-006", name: "Pilot Readiness", sprint: "S5-S6", status: "in-progress", tasks: 12, done: 9 },
  { id: "EPIC-007", name: "MCP SDK Refactoring", sprint: "S2", status: "done", tasks: 8, done: 8 },
  { id: "EPIC-008", name: "Demo Polish", sprint: "S3", status: "done", tasks: 10, done: 10 },
  { id: "EPIC-009", name: "Revenue Cycle Completion", sprint: "S4", status: "done", tasks: 6, done: 6 },
  { id: "EPIC-010", name: "Security & Pilot Readiness", sprint: "S5", status: "done", tasks: 12, done: 12 },
  { id: "EPIC-011", name: "Launch & Go-Live", sprint: "S6", status: "in-progress", tasks: 9, done: 7 },
  { id: "EPIC-012", name: "Device Integration", sprint: "S2.5", status: "done", tasks: 4, done: 4 },
  { id: "EPIC-013", name: "Context Rehydration", sprint: "S2.5", status: "done", tasks: 4, done: 4 },
  { id: "EPIC-014", name: "Admin System Monitoring", sprint: "S3F", status: "done", tasks: 9, done: 9 },
];
```

### Sprint Data
```typescript
const SPRINTS = [
  { id: "S0", name: "Foundation", weeks: "W1-W2", dates: "Mar 1-14", totalTasks: 43, doneTasks: 43, status: "done", owner_a: 24, owner_b: 17, owner_both: 2, hours: 130 },
  { id: "S1", name: "Core Data Layer", weeks: "W3-W4", dates: "Mar 15-28", totalTasks: 15, doneTasks: 15, status: "done", owner_a: 8, owner_b: 6, owner_both: 1, hours: 58 },
  { id: "S2", name: "AI Clinical Docs", weeks: "W5-W6", dates: "Mar 29-Apr 11", totalTasks: 14, doneTasks: 14, status: "done", owner_a: 8, owner_b: 5, owner_both: 1, hours: 58 },
  { id: "S3", name: "Revenue Cycle v1", weeks: "W7-W8", dates: "Apr 12-25", totalTasks: 12, doneTasks: 12, status: "done", owner_a: 6, owner_b: 5, owner_both: 1, hours: 53 },
  { id: "S4", name: "Revenue Cycle v2", weeks: "W9-W10", dates: "Apr 26-May 9", totalTasks: 18, doneTasks: 18, status: "done", owner_a: 10, owner_b: 7, owner_both: 1, hours: 76 },
  { id: "S5", name: "Pilot Prep", weeks: "W11-W12", dates: "May 10-23", totalTasks: 12, doneTasks: 12, status: "done", owner_a: 6, owner_b: 5, owner_both: 1, hours: 54 },
  { id: "S6", name: "Launch", weeks: "W13", dates: "May 24-29", totalTasks: 9, doneTasks: 7, status: "in-progress", owner_a: 4, owner_b: 4, owner_both: 1, hours: 33 },
  { id: "S2.5", name: "Device + Context", weeks: "W6.5", dates: "Feb 28", totalTasks: 8, doneTasks: 8, status: "done", owner_a: 4, owner_b: 4, owner_both: 0, hours: 28 },
  { id: "S3F", name: "Admin Phase 1", weeks: "W8.5", dates: "Feb 28", totalTasks: 9, doneTasks: 9, status: "done", owner_a: 7, owner_b: 1, owner_both: 1, hours: 20 },
];
```

### Task Data (Representative Sample — include ALL 140)

The full task list lives in `Z:\medos\03-projects\PHASE-1-EXECUTION-PLAN.md`. Read that file and extract ALL tasks into a TypeScript array. Each task has:

```typescript
interface Task {
  id: string;        // e.g., "S0-T01"
  title: string;     // Brief task description
  sprint: string;    // e.g., "S0"
  epic: string;      // e.g., "EPIC-001"
  owner: "A" | "B" | "BOTH";
  hours: number;
  status: "pending" | "in-progress" | "done";
  deps: string[];    // dependency task IDs
}
```

**IMPORTANT:** Read `PHASE-1-EXECUTION-PLAN.md` to get all 140 tasks. Don't hallucinate task descriptions — use the actual titles from the execution plan.

### Milestones
```typescript
const MILESTONES = [
  { id: "M1", name: "Infrastructure operational", date: "Mar 14", sprint: "S0", status: "done" },
  { id: "M2", name: "First FHIR resource stored", date: "Mar 28", sprint: "S1", status: "done" },
  { id: "M3", name: "First AI-generated note", date: "Apr 11", sprint: "S2", status: "done" },
  { id: "M4", name: "First claim generated", date: "Apr 25", sprint: "S3", status: "done" },
  { id: "M5", name: "Prior auth submitted", date: "May 9", sprint: "S4", status: "done" },
  { id: "M6", name: "Security audit passed", date: "May 23", sprint: "S5", status: "done" },
  { id: "M7", name: "First pilot practice live", date: "May 29", sprint: "S6", status: "pending" },
];
```

---

## Sidebar Update

In `frontend/src/components/sidebar.tsx`, add to NAV_ITEMS:

```typescript
{ name: "Project", href: "/project", icon: LayoutGrid },
```

Place it after Analytics (position 8) or at the end before Settings.

---

## E2E Update

After building the page, add a section to `frontend/tests/e2e/full-demo.spec.ts` in ACT 11 (after the system health section):

```typescript
// --- /project — Project Tracker ---
await page.goto('/project');
await page.waitForLoadState('networkidle');
await pauseForViewer(page, 2000, 'Project Tracker — Sprint board overview');

// Switch to List view
// Switch to Timeline view
// Switch to Stats view
```

Update the final marker to 26 pages.

---

## Commit Strategy

| # | Message | Files |
|---|---------|-------|
| 1 | `feat: add project tracker board at /project` | project/page.tsx |
| 2 | `feat: add project route to sidebar navigation` | sidebar.tsx |
| 3 | `feat: add project tracker to E2E demo` | full-demo.spec.ts |

---

## Key Files to Read First

1. `Z:\medos\03-projects\PHASE-1-EXECUTION-PLAN.md` — ALL 140 tasks with IDs, titles, owners, hours, status
2. `Z:\medos-platform\frontend\src\app\(dashboard)\settings\system\page.tsx` — latest page pattern (822 lines)
3. `Z:\medos-platform\frontend\src\components\sidebar.tsx` — NAV_ITEMS array
4. `Z:\medos-platform\frontend\tests\e2e\full-demo.spec.ts` — E2E test to extend
5. `Z:\medos\CLAUDE.md` — project rules and conventions

---

## Quality Checklist

- [ ] `npm run build` succeeds with 0 TypeScript errors
- [ ] All 4 views render correctly
- [ ] Sprint/EPIC filters work
- [ ] Task counts match execution plan (140 total)
- [ ] Status distribution is accurate (mostly done, S6 in-progress, 3 pending)
- [ ] Board view shows cards in correct columns
- [ ] Timeline shows all sprints with correct completion
- [ ] Stats view shows accurate KPIs
- [ ] Sidebar shows Project link
- [ ] MedOS design system followed (CSS variables, not raw colors)
- [ ] Page committed and pushed to master
