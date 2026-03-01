---
type: system-design
date: "2026-03-01"
status: draft
tags:
  - architecture
  - system-design
  - admin
  - frontend
  - module-g
  - phase-1
---

# MedOS /admin Page -- Comprehensive Design Analysis

> Design specification for the unified administration hub of MedOS Healthcare OS. This page consolidates platform management, project tracking, security compliance, system observability, and tenant configuration into a single surface for practice administrators and super-admins.

**Related:** [[EPIC-014-admin-system-monitoring]] | [[System-Architecture-Overview]] | [[ADR-002-multi-tenancy-schema-per-tenant]] | [[agent-architecture]] | [[mcp-integration-plan]] | [[HIPAA-Deep-Dive]] | [[monitoring-rotation-runbook]]

---

## 1. Design Philosophy

The /admin page is the **nerve center** of MedOS. It answers three questions at a glance:

1. **Is the system healthy?** -- Uptime, error rates, latency, cache performance, agent confidence.
2. **Is the practice configured correctly?** -- Tenants, users, roles, payers, integrations, feature flags.
3. **Is the project on track?** -- Sprints, tasks, blockers, metrics (the existing Kanban tracker).

The design follows the principle of **progressive disclosure**: the top-level view shows summary KPIs with traffic-light indicators. Drilling into any section reveals full configuration and action capabilities. An administrator should never need to leave /admin to manage the platform.

### Target Personas

| Persona | Role | Primary Sections |
|---------|------|-----------------|
| **Practice Administrator** | Day-to-day practice management: users, providers, payers, settings | Tenant Management, User & Roles, Billing Config, Integrations |
| **System Administrator** | Technical platform management: MCP servers, agents, cache, alerts | MCP Management, Agent Config, System Config, Monitoring |
| **Super Admin (MedOS Team)** | Cross-tenant oversight, project tracking, compliance | All sections, plus Project Management and cross-tenant views |
| **Compliance Officer** | HIPAA audit, security review, encryption verification | Security & Compliance (primary), Audit Logs, Data Management |

---

## 2. Navigation Structure

The /admin page uses a **left sidebar navigation** with 12 sections organized into 4 groups. The sidebar is always visible, with the active section highlighted. Each section may contain tabs for sub-views.

```
/admin
|
|-- OPERATIONS
|   |-- /admin/dashboard              (Section 1: Admin Dashboard / Overview)
|   |-- /admin/monitoring             (Section 2: Monitoring & Observability)
|   |-- /admin/project                (Section 3: Project Management -- moved from /project)
|
|-- PRACTICE
|   |-- /admin/tenants                (Section 4: Tenant / Practice Management)
|   |-- /admin/users                  (Section 5: User & Role Management)
|   |-- /admin/billing                (Section 6: Billing Configuration)
|   |-- /admin/integrations           (Section 7: Integration Management)
|
|-- PLATFORM
|   |-- /admin/mcp                    (Section 8: MCP Server Management)
|   |-- /admin/agents                 (Section 9: Agent Configuration)
|   |-- /admin/system                 (Section 10: System Configuration)
|   |-- /admin/features               (Section 11: Feature Flags)
|
|-- COMPLIANCE
|   |-- /admin/security               (Section 12: Security & Compliance)
|   |-- /admin/data                   (Section 13: Data Management)
```

### URL Mapping (from existing routes)

| Existing Route | New Route | Migration Notes |
|---|---|---|
| `/project` | `/admin/project` | Redirect `/project` -> `/admin/project` for backwards compatibility |
| `/settings/devices` | `/admin/system` (Devices tab) | Consolidate under system config, or keep as separate sub-route |
| `/settings/context` | `/admin/system` (Context tab) | Consolidate under system config |
| `/settings/system` | `/admin/monitoring` | MCP Inventory, Agent Perf, Cache move here |
| `/settings/practice` | `/admin/tenants` (current tenant) | Practice config becomes tenant self-management |
| `/settings/onboarding` | `/admin/tenants/onboard` | Onboarding wizard lives under tenant management |

### Responsive Design

- **Desktop (>1280px):** Full sidebar + content area + optional detail panel (3-column on wide screens).
- **Tablet (768-1280px):** Collapsible sidebar (icon-only mode), full content area.
- **Mobile (<768px):** Bottom tab bar with top 5 sections; remaining sections in a "More" menu. Admin pages are primarily desktop-oriented but must remain functional on tablet for on-site practice management.

---

## 3. Section Specifications

---

### Section 1: Admin Dashboard (Overview)

**Route:** `/admin/dashboard`
**Purpose:** Single-screen executive summary of the entire platform. This is the landing page of /admin.

#### What to Display

**Row 1 -- System Health Strip (5 KPI cards):**

| KPI | Source | Target | Warning | Critical |
|-----|--------|--------|---------|----------|
| System Uptime | Health endpoint | > 99.5% | < 99.5% | < 99.0% |
| API Error Rate (5xx) | MetricsCollector | < 1% | > 3% | > 5% |
| P99 Latency | MetricsCollector | < 1s | > 1.5s | > 2s |
| Active Users (today) | Auth logs | varies | -- | -- |
| AI Avg Confidence | Agent metrics | > 0.90 | < 0.87 | < 0.85 |

**Row 2 -- Platform Metrics Strip (5 KPI cards):**

| KPI | Value | Description |
|-----|-------|-------------|
| MCP Tools Active | 44 / 44 | Tools online vs. registered |
| Agents Running | 3 / 3 | Active agent types |
| Tenants Active | e.g. 3 | Practices currently onboarded |
| FHIR Resources | e.g. 12,847 | Total resources across tenants (aggregated, no PHI) |
| Context Freshness | 0.82 avg | System-wide average freshness score |

**Row 3 -- Subsystem Health Grid (6 cards, one per MCP server):**

Each card shows: server name, tool count, status dot (green/amber/red), last health check timestamp, requests/min.

```
+------------------+------------------+------------------+
| FHIR Server      | Scribe Server    | Billing Server   |
| 12 tools | GREEN | 6 tools  | GREEN | 8 tools  | GREEN |
| 142 req/min      | 28 req/min       | 67 req/min       |
+------------------+------------------+------------------+
| Scheduling       | Device Server    | Context Server   |
| 6 tools  | GREEN | 8 tools  | AMBER | 4 tools  | GREEN |
| 34 req/min       | 12 req/min       | 89 req/min       |
+------------------+------------------+------------------+
```

**Row 4 -- Quick Actions:**

- "Onboard New Practice" -> `/admin/tenants/onboard`
- "View Audit Log" -> `/admin/security?tab=audit`
- "Run Health Check" -> Trigger `/health/ready` and display results inline
- "View Alerts" -> `/admin/monitoring?tab=alerts`

**Row 5 -- Recent Activity Feed (last 20 events, auto-refresh via WebSocket):**

Timeline showing: new user created, agent task completed, alert triggered, tenant onboarded, deployment completed, payer config updated, etc.

#### Mock Data

```json
{
  "uptime": "99.87%",
  "uptime_since": "2026-02-15T00:00:00Z",
  "error_rate_5xx": "0.4%",
  "p99_latency_ms": 847,
  "active_users_today": 23,
  "ai_avg_confidence": 0.91,
  "mcp_tools_active": 44,
  "mcp_tools_total": 44,
  "agents_active": 3,
  "tenants_active": 3,
  "fhir_resources_total": 12847,
  "context_freshness_avg": 0.82,
  "recent_events": [
    {"ts": "2026-03-01T14:32:00Z", "type": "agent.task_completed", "detail": "Clinical Scribe processed encounter E-2847"},
    {"ts": "2026-03-01T14:28:00Z", "type": "user.created", "detail": "Dr. Sarah Chen invited to Sunshine Orthopedics"},
    {"ts": "2026-03-01T14:15:00Z", "type": "alert.resolved", "detail": "Device Server latency returned to normal"},
    {"ts": "2026-03-01T13:55:00Z", "type": "claim.submitted", "detail": "Batch of 12 claims submitted to Change Healthcare"},
    {"ts": "2026-03-01T13:40:00Z", "type": "tenant.config_updated", "detail": "Palm Beach Dermatology updated fee schedule"}
  ]
}
```

#### Actions an Admin Can Take

- Trigger manual health check across all subsystems
- Dismiss or acknowledge alerts from the dashboard
- Navigate to any subsystem with a single click
- Toggle between "All Tenants" and "My Tenant" view (super-admin vs practice admin)

---

### Section 2: Monitoring & Observability

**Route:** `/admin/monitoring`
**Purpose:** Real-time and historical system performance metrics, alerts, and logs.

#### Tabs

**Tab 1 -- Real-Time Metrics:**

- **API Performance Chart:** P50, P95, P99 latency over last 24 hours (line chart, 1-minute granularity). Clickable time ranges: 1h, 6h, 24h, 7d, 30d.
- **Error Rate Chart:** 5xx and 4xx rates over time (stacked area chart).
- **Throughput Chart:** Requests per minute by endpoint category (FHIR, AI, Billing, Scheduling, Admin).
- **Status Code Breakdown:** Donut chart showing 2xx/3xx/4xx/5xx distribution.

Configurable items:
- Time range selector
- Endpoint category filter
- Tenant filter (super-admin only)

**Tab 2 -- Alerts:**

- **Active Alerts Table:**

| Column | Example |
|--------|---------|
| Severity | P1 (Critical) |
| Alert Name | API Error Rate > 5% |
| Triggered At | 2026-03-01 14:15 ET |
| Duration | 12 min |
| Affected Service | Billing MCP Server |
| Status | Active |
| Actions | Acknowledge, Snooze (15m/1h/4h), Escalate |

- **Alert History:** Last 30 days of resolved alerts with resolution notes.
- **Alert Rules Management:** View/edit threshold rules (based on AlertManager config).

Alert rule configurable fields:
- Metric name, threshold value, evaluation window, severity level, notification channel (Slack, PagerDuty, email).

Mock alert rules:

| Rule | Metric | Warning | Critical | Window | Channel |
|------|--------|---------|----------|--------|---------|
| High Error Rate | API 5xx rate | > 3% | > 5% | 5 min | PagerDuty |
| Slow API | P99 latency | > 1.5s | > 2s | 5 min | PagerDuty |
| DB Pool Exhaustion | Connection pool utilization | > 70% | > 85% | 5 min | PagerDuty |
| High CPU | ECS CPU utilization | > 70% | > 80% | 5 min | Slack |
| High Memory | ECS memory utilization | > 75% | > 85% | 5 min | PagerDuty |
| LLM Degraded | Claude error rate | > 5% | > 10% | 5 min | Slack |
| Queue Buildup | Task queue depth | > 500 | > 1000 | 5 min | Slack |
| Cost Spike | AWS daily spend | > $75 | > $100 | 24h | Email |

**Tab 3 -- Infrastructure:**

- **ECS Services:** Task count, CPU/memory per service, deployment status, last restart.
- **Database:** Connection pool utilization, query latency, active queries, replication lag.
- **Redis:** Memory usage, hit rate, eviction rate, connected clients, keyspace stats.
- **S3:** Storage used per tenant, request counts.

**Tab 4 -- AI Observability (Langfuse):**

- **LLM Call Volume:** Calls per hour by agent type (line chart).
- **Token Usage:** Input/output tokens per agent (stacked bar chart, daily).
- **Cost Tracking:** LLM cost per day, projected monthly spend.
- **Confidence Distribution:** Histogram of confidence scores across all agent outputs.
- **Trace Viewer:** Link to Langfuse dashboard for deep trace inspection.
- **Quality Metrics:**

| Agent | Tasks (24h) | Avg Confidence | Human Review Rate | Avg Duration |
|-------|-------------|----------------|-------------------|-------------|
| Clinical Scribe | 47 | 0.91 | 12% | 8.2s |
| Prior Auth | 13 | 0.88 | 23% | 14.5s |
| Denial Management | 8 | 0.85 | 31% | 22.1s |

**Tab 5 -- Cost Dashboard:**

- **Daily AWS Spend:** Bar chart, last 30 days. Breakdown by service (EC2, RDS, S3, Bedrock, Data Transfer).
- **LLM Token Spend:** Daily cost by model (Claude Sonnet, Claude Haiku).
- **Budget vs Actual:** Monthly budget progress bar with projection.
- **Per-Tenant Cost Allocation:** Estimated cost per tenant based on usage (super-admin view).

Mock data:

```json
{
  "daily_spend_usd": 42.17,
  "monthly_projected_usd": 1265.10,
  "monthly_budget_usd": 1500.00,
  "breakdown": {
    "ecs_fargate": 18.50,
    "rds_postgresql": 12.30,
    "elasticache_redis": 4.20,
    "bedrock_claude": 3.80,
    "s3_storage": 1.20,
    "data_transfer": 0.90,
    "kms": 0.87,
    "other": 0.40
  },
  "llm_tokens_today": {
    "input": 1847000,
    "output": 523000,
    "cost_usd": 3.80
  }
}
```

#### Actions

- Create/edit/delete alert rules
- Acknowledge or snooze active alerts
- Export metrics data (CSV, JSON) for a selected time range
- Force ECS service restart (with confirmation dialog)
- Scale ECS task count manually (with guardrails)

---

### Section 3: Project Management

**Route:** `/admin/project`
**Purpose:** The existing Kanban project tracker, relocated from /project. Tracks all 140+ tasks across 9 sprints.

#### What Currently Exists

This is the existing project tracker with 4 views:
1. **Kanban Board:** Tasks as cards in columns (Pending, In Progress, Done, Blocked). Drag-and-drop to change status.
2. **Timeline (Gantt):** Sprint timeline with task bars showing dependencies.
3. **List View:** Filterable, sortable table of all tasks.
4. **Sprint Summary:** Per-sprint progress bars and burn-down chart.

#### Enhancements for /admin

- **Sprint Velocity Widget:** Tasks completed per sprint vs. planned.
- **Blocked Tasks Callout:** Red banner showing any blocked tasks with dependency chains.
- **Team Workload:** Task distribution between Person A and Person B.
- **Link to EPICs:** Each task card links to its parent EPIC document in the vault.
- **Automated Status Sync:** When backend tests pass CI/CD, auto-update related tasks.

#### Mock Data (Sprint 6 view)

| ID | Task | Owner | Status | Sprint |
|----|------|-------|--------|--------|
| S6-T01 | Deploy production environment | A | done | 6 |
| S6-T02 | Create demo environment | A | done | 6 |
| S6-T03 | Pilot practice onboarding | B | pending | 6 |
| S6-T04 | On-site training | B | pending | 6 |
| S6-T05 | Pilot success metrics | B | done | 6 |
| S6-T06 | Daily monitoring rotation | A | done | 6 |
| S6-T07 | Automated backup verification | A | done | 6 |
| S6-T08 | Pilot feedback channel | BOTH | done | 6 |
| S6-T09 | Launch day go-live support | BOTH | pending | 6 |

Sprint 6 Progress: 6/9 tasks done (67%).

#### Actions

- Create new tasks with sprint assignment
- Update task status (drag-and-drop or inline edit)
- Add blockers and link dependencies
- Filter by sprint, owner, status, EPIC
- Export sprint report

---

### Section 4: Tenant / Practice Management

**Route:** `/admin/tenants`
**Purpose:** Create, configure, and manage healthcare practice tenants in the multi-tenant system.

#### Tabs

**Tab 1 -- Tenant Directory:**

Table of all onboarded tenants (practices).

| Column | Example |
|--------|---------|
| Practice Name | Sunshine Orthopedics |
| Tenant ID | `tenant_a1b2c3d4...` |
| Slug | `sunshine-ortho` |
| Specialty | Orthopedics |
| Plan Tier | Standard |
| Status | Active |
| Providers | 8 |
| Patients | 2,340 |
| Created | 2026-03-01 |
| KMS Key Status | Active |
| Schema | `tenant_a1b2c3d4` |

Configurable items per tenant:
- Display name, plan tier, status (active/suspended/deactivated)
- Custom settings JSONB (appointment slot duration, default encounter types, specialty config)
- KMS key rotation schedule

**Tab 2 -- Onboarding Wizard:**

The multi-step wizard from S5-T05, now accessible at `/admin/tenants/onboard`:

1. **Organization Info:** Practice name, NPI (Type 2), Tax ID, address, phone, specialty, website.
2. **Admin User:** Email, name, role assignment (auto-assigned as Tenant Admin).
3. **Practice Configuration:** Business hours, appointment types, encounter templates, locations.
4. **Payer Setup:** Select payers from directory, enter contract terms, clearinghouse IDs.
5. **Data Import:** Upload patient CSV, parse HL7v2 ADT messages, or connect EHR bridge.
6. **Confirmation:** Review all settings, provision tenant schema, generate KMS key, activate.

Progress bar at the top showing completed steps.

**Tab 3 -- Tenant Health:**

Per-tenant health metrics (super-admin view):

| Tenant | Error Rate | P99 Latency | FHIR Resources | Daily Active Users | Storage Used |
|--------|-----------|-------------|----------------|-------------------|-------------|
| Sunshine Orthopedics | 0.3% | 720ms | 8,241 | 12 | 2.4 GB |
| Palm Beach Dermatology | 0.1% | 680ms | 3,102 | 8 | 0.9 GB |
| Miami Spine Center | 0.8% | 890ms | 1,504 | 5 | 0.6 GB |

#### Mock Data

```json
{
  "tenants": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "slug": "sunshine-ortho",
      "display_name": "Sunshine Orthopedics",
      "npi": "1234567890",
      "tax_id": "XX-XXXXXXX",
      "specialty": "Orthopedics",
      "plan_tier": "standard",
      "status": "active",
      "schema_name": "tenant_a1b2c3d4",
      "kms_key_arn": "arn:aws:kms:us-east-1:123456789:key/abc-def",
      "kms_key_status": "Enabled",
      "providers_count": 8,
      "patients_count": 2340,
      "locations_count": 2,
      "created_at": "2026-03-01T10:00:00Z",
      "settings": {
        "appointment_slot_minutes": 30,
        "default_encounter_type": "office-visit",
        "business_hours": {"start": "08:00", "end": "17:00"},
        "timezone": "America/New_York"
      }
    }
  ]
}
```

#### Actions

- Onboard new tenant (wizard)
- Suspend/reactivate tenant
- Rotate KMS encryption key (with confirmation)
- Export tenant configuration
- View tenant audit log
- Adjust plan tier and rate limits
- Trigger schema migration for a specific tenant

---

### Section 5: User & Role Management

**Route:** `/admin/users`
**Purpose:** Manage all users across the tenant, assign roles and permissions, enforce MFA, and audit access.

#### Tabs

**Tab 1 -- User Directory:**

| Column | Example |
|--------|---------|
| Name | Dr. Sarah Chen |
| Email | sarah.chen@sunshine-ortho.com |
| Role | Physician |
| Status | Active |
| MFA | Enabled |
| Last Login | 2026-03-01 09:12 ET |
| Created | 2026-02-15 |
| Locations | Main Office, Satellite |

**Tab 2 -- Roles & Permissions:**

Visual RBAC matrix showing which roles have which FHIR scopes:

| Permission | Physician | Nurse | MA | Biller | Front Desk | Admin |
|-----------|-----------|-------|-----|--------|-----------|-------|
| patient/*.read | Yes | Yes | Yes | Demographics only | Demographics only | Yes |
| patient/*.write | Yes | No | No | No | Demographics only | Yes |
| encounter/*.read | Yes | Yes | Yes | No | No | Yes |
| encounter/*.write | Yes | No | No | No | No | Yes |
| observation/*.write | Yes | Yes | Yes | No | No | Yes |
| claim/*.read | Yes | No | No | Yes | No | Yes |
| claim/*.write | No | No | No | Yes | No | Yes |
| system/*.*  | No | No | No | No | No | Yes |
| agent.trigger | Yes | No | No | Yes | No | Yes |
| admin.users | No | No | No | No | No | Yes |
| admin.config | No | No | No | No | No | Yes |

Custom roles: Admin can create custom roles by selecting individual permissions from the matrix.

**Tab 3 -- ABAC Policies:**

Attribute-based access policies (fine-grained, applied after RBAC):

| Policy | Description | Status |
|--------|-------------|--------|
| Care Relationship | Physicians can only access patients they are treating | Active |
| Location Restriction | Staff can only access patients at their assigned location | Active |
| Break-the-Glass | Emergency access with mandatory audit justification | Active |
| Billing Scope | Billers see demographics + financial data, not clinical notes | Active |
| Time-Based | Off-hours access triggers additional audit flag | Active |

**Tab 4 -- Invite & Onboard:**

- **Invite User Form:** Email, role, locations, optional ABAC overrides.
- **Pending Invitations:** Table showing sent but unaccepted invitations with resend/revoke actions.
- **Bulk Import:** Upload CSV of users (name, email, role) for batch onboarding.

**Tab 5 -- Access Audit:**

- Login history per user (last 90 days).
- Failed login attempts with IP addresses.
- Break-the-glass events requiring review.
- Session duration analytics.

#### Mock Data

```json
{
  "users": [
    {
      "id": "usr_001",
      "name": "Dr. Sarah Chen",
      "email": "sarah.chen@sunshine-ortho.com",
      "role": "physician",
      "status": "active",
      "mfa_enabled": true,
      "last_login": "2026-03-01T09:12:00Z",
      "locations": ["Main Office", "Satellite"],
      "npi": "1234567891",
      "specialty": "Orthopedic Surgery",
      "created_at": "2026-02-15T10:00:00Z"
    },
    {
      "id": "usr_002",
      "name": "Maria Rodriguez",
      "email": "maria.r@sunshine-ortho.com",
      "role": "biller",
      "status": "active",
      "mfa_enabled": true,
      "last_login": "2026-03-01T08:45:00Z",
      "locations": ["Main Office"],
      "created_at": "2026-02-20T10:00:00Z"
    },
    {
      "id": "usr_003",
      "name": "James Wilson",
      "email": "james.w@sunshine-ortho.com",
      "role": "nurse",
      "status": "active",
      "mfa_enabled": false,
      "last_login": "2026-02-28T16:30:00Z",
      "locations": ["Main Office"],
      "created_at": "2026-02-22T10:00:00Z"
    }
  ],
  "pending_invitations": [
    {
      "email": "new.provider@sunshine-ortho.com",
      "role": "physician",
      "sent_at": "2026-02-28T10:00:00Z",
      "expires_at": "2026-03-07T10:00:00Z"
    }
  ]
}
```

#### Actions

- Invite new user (email + role assignment)
- Deactivate/reactivate user (immediate access revocation)
- Change user role
- Force MFA enrollment
- Reset user password
- View user's complete access audit trail
- Create/edit custom roles
- Enable/disable ABAC policies
- Bulk import users from CSV

---

### Section 6: Billing Configuration

**Route:** `/admin/billing`
**Purpose:** Configure the revenue cycle engine: payer contracts, fee schedules, clearinghouse connections, scrubbing rules, and coding preferences.

#### Tabs

**Tab 1 -- Payer Contracts:**

| Column | Example |
|--------|---------|
| Payer Name | UnitedHealthcare |
| Payer ID | 87726 |
| Contract Status | Active |
| Effective Date | 2026-01-01 |
| Termination Date | 2026-12-31 |
| Timely Filing (days) | 180 |
| PA Required CPTs | 27447, 27130, 29881 |
| Clearinghouse | Change Healthcare |

Expandable row reveals:
- Allowed amounts per CPT code (fee schedule)
- Modifier rules
- Bundling/unbundling rules
- Prior authorization requirements by CPT
- Appeal filing deadlines by denial category
- Payer-specific EDI settings (receiver ID, ISA qualifier)

**Tab 2 -- Fee Schedules:**

Interactive fee schedule editor:

| CPT | Description | Medicare | UHC | Aetna | BCBS | Self-Pay |
|-----|-------------|----------|-----|-------|------|----------|
| 99213 | Office visit, est. pt, low complexity | $92.05 | $110.46 | $105.00 | $98.50 | $150.00 |
| 99214 | Office visit, est. pt, mod complexity | $133.63 | $160.36 | $155.00 | $142.00 | $200.00 |
| 27447 | Total knee arthroplasty | $1,524.88 | $1,829.86 | $1,750.00 | $1,600.00 | $2,500.00 |
| 29881 | Knee arthroscopy w/ meniscectomy | $571.95 | $686.34 | $650.00 | $610.00 | $900.00 |

Upload: CSV, Excel, or CMS Fee Schedule (MPFS) import.

**Tab 3 -- Clearinghouse Settings:**

| Connection | Clearinghouse | Protocol | Status | Last Successful |
|-----------|---------------|----------|--------|----------------|
| Claims (837P) | Change Healthcare | X12 EDI / SFTP | Connected | 2026-03-01 14:00 |
| Eligibility (270/271) | Availity | API | Connected | 2026-03-01 14:32 |
| Remittance (835) | Change Healthcare | X12 EDI / SFTP | Connected | 2026-03-01 06:00 |
| Prior Auth (278) | Availity | API | Connected | 2026-02-28 16:45 |

Configurable per connection:
- Host/endpoint URL
- Credentials (reference to AWS Secrets Manager -- never displayed)
- Sender/Receiver IDs (ISA05/ISA07 qualifiers)
- Schedule (batch frequency for 837P, real-time for 270/271)
- Test mode toggle (sandbox vs production)

**Tab 4 -- Scrubbing Rules:**

Claims scrubber rule management:

| Rule | Category | Severity | Status | Description |
|------|----------|----------|--------|-------------|
| Missing NPI | Provider | Error | Active | Rendering provider NPI required on all claims |
| Missing Diagnosis Pointer | Coding | Error | Active | Every service line needs at least one diagnosis pointer |
| Invalid Place of Service | Demographics | Warning | Active | POS code must match service type |
| Gender-Diagnosis Mismatch | Coding | Error | Active | ICD-10 code incompatible with patient gender |
| Duplicate Claim | Billing | Error | Active | Same patient, date, provider, CPT within 30 days |
| Timely Filing Risk | Billing | Warning | Active | Claim approaching payer timely filing deadline |
| Modifier Required | Coding | Warning | Active | CPT requires modifier 25, 59, or 76 based on context |
| Custom Rule | -- | -- | -- | Admin-defined scrubbing rules with regex/logic |

Admin can toggle rules active/inactive, adjust severity, and create custom rules.

**Tab 5 -- Coding Preferences:**

Specialty-specific coding favorites and defaults:

| Setting | Value | Scope |
|---------|-------|-------|
| Default ICD-10 Favorites | M17.11, M17.12, M25.561, M79.3 | Orthopedics |
| Default CPT Favorites | 99213, 99214, 27447, 29881, 20610 | Orthopedics |
| Default Modifiers | 25 (Significant E/M), RT/LT (laterality) | Practice-wide |
| AI Coding Confidence Threshold | 0.92 | Practice-wide |
| Auto-suggest Codes | Enabled | Practice-wide |
| Coding Audit Frequency | Every 20th encounter | Practice-wide |

#### Mock Data

```json
{
  "payer_contracts": [
    {
      "payer_name": "UnitedHealthcare",
      "payer_id": "87726",
      "contract_status": "active",
      "effective_date": "2026-01-01",
      "termination_date": "2026-12-31",
      "timely_filing_days": 180,
      "pa_required_cpts": ["27447", "27130", "29881"],
      "clearinghouse": "Change Healthcare",
      "fee_schedule": {
        "99213": 110.46,
        "99214": 160.36,
        "27447": 1829.86,
        "29881": 686.34
      }
    },
    {
      "payer_name": "Aetna",
      "payer_id": "60054",
      "contract_status": "active",
      "effective_date": "2026-01-01",
      "termination_date": "2026-12-31",
      "timely_filing_days": 120,
      "pa_required_cpts": ["27447", "27130"],
      "clearinghouse": "Availity"
    },
    {
      "payer_name": "Medicare Part B",
      "payer_id": "00882",
      "contract_status": "active",
      "effective_date": "2026-01-01",
      "timely_filing_days": 365,
      "pa_required_cpts": [],
      "clearinghouse": "Change Healthcare"
    }
  ]
}
```

#### Actions

- Add/edit/deactivate payer contracts
- Upload fee schedules (CSV/Excel)
- Test clearinghouse connectivity (send X12 270 test transaction)
- Toggle scrubbing rules
- Create custom scrubbing rules
- Set coding preferences per specialty
- View scrub failure analytics (which rules fire most often)
- Compare contract rates vs Medicare benchmark

---

### Section 7: Integration Management

**Route:** `/admin/integrations`
**Purpose:** Configure and monitor external system connections: EHR bridges, device integrations, lab connections, and third-party MCP apps.

#### Tabs

**Tab 1 -- EHR Connections:**

| Connection | EHR | Protocol | Status | Last Sync | Patients Synced | Conflicts |
|-----------|-----|----------|--------|-----------|-----------------|-----------|
| Epic Sandbox | Epic | FHIR R4 + SMART | Connected | 2026-03-01 14:15 | 847 | 3 |
| Cerner Sandbox | Oracle Health | FHIR R4 | Disconnected | 2026-02-28 10:00 | 0 | 0 |

Per-connection configuration:
- FHIR base URL
- Client ID (displayed), Private Key (reference to Secrets Manager)
- Sync schedule (every 15 min, hourly, daily, manual)
- Conflict resolution strategy: "EHR wins" (default) or "Manual review"
- Resource types to sync: Patient, Encounter, Observation, Condition, etc.
- Connection health: last 10 sync results with success/failure and record counts

**Tab 2 -- Device Integrations:**

Migrated and expanded from `/settings/devices`:

| Device Type | API | Status | Registered Devices | Readings (24h) | Alerts (24h) |
|------------|-----|--------|-------------------|----------------|-------------|
| Oura Ring Gen 4 | Oura Cloud API | Active | 12 | 1,847 | 2 |
| Apple Watch Ultra 3 | HealthKit (via app) | Active | 8 | 3,204 | 1 |
| Dexcom G8 CGM | Dexcom API | Active | 3 | 864 | 0 |
| Withings BPM Core | Withings API | Pending Setup | 0 | 0 | 0 |

Per-device-type configuration:
- API credentials (reference to Secrets Manager)
- Webhook endpoint URL
- Alert thresholds (HR high/low, SpO2 low, glucose high/low, temp high/low)
- Sync mode (push via webhook, pull via mobile app, batch)
- Data retention policy (days)
- LOINC mapping overrides

**Tab 3 -- Lab & Pharmacy:**

| Connection | System | Protocol | Status |
|-----------|--------|----------|--------|
| Quest Diagnostics | HL7v2 ORU/ORM | MLLP | Not Configured |
| Walgreens Pharmacy | NCPDP SCRIPT | API | Not Configured |

Future integrations -- shown as disabled cards with "Configure" button for roadmap visibility.

**Tab 4 -- Third-Party MCP Apps:**

Apps registered through the MedOS Developer Portal:

| App | Developer | Access Tier | Tools Granted | Status | Last Activity |
|-----|-----------|------------|---------------|--------|--------------|
| DermAI Image Analysis | DermTech Inc | Read-Write | fhir_read, fhir_search, fhir_create | Approved | 2026-03-01 13:00 |
| Coding Audit Pro | CodeRight Health | Read-Only | fhir_search, analytics_coding_accuracy | Pending Review | -- |

Per-app management:
- View granted MCP tool scopes
- Revoke access immediately
- View app's audit log (every MCP call)
- Rate limit configuration
- Sandbox vs production toggle

#### Actions

- Add/configure EHR connection (FHIR endpoint + SMART credentials)
- Trigger manual EHR sync
- View/resolve sync conflicts
- Register/deregister devices
- Configure device alert thresholds
- Approve/reject third-party MCP app requests
- Revoke app access
- Test connectivity for any integration

---

### Section 8: MCP Server Management

**Route:** `/admin/mcp`
**Purpose:** Monitor, configure, and control the 6 MCP servers and their 44 tools.

#### Tabs

**Tab 1 -- Server Overview:**

| Server | Tools | Status | Requests (24h) | Errors (24h) | P99 Latency | Circuit Breaker |
|--------|-------|--------|----------------|-------------|-------------|-----------------|
| FHIR Server | 12 | Healthy | 14,823 | 12 | 420ms | Closed |
| Scribe Server | 6 | Healthy | 2,847 | 3 | 890ms | Closed |
| Billing Server | 8 | Healthy | 6,912 | 8 | 340ms | Closed |
| Scheduling Server | 6 | Healthy | 3,421 | 2 | 280ms | Closed |
| Device Server | 8 | Degraded | 1,204 | 47 | 1,240ms | Half-Open |
| Context Server | 4 | Healthy | 8,934 | 5 | 190ms | Closed |

**Tab 2 -- Tool Inventory:**

Expanded from the existing `/settings/system` MCP Inventory tab. All 44 tools organized by server, with detailed metadata:

| Tool | Server | PHI Level | Approval Required | Status | Calls (24h) | Avg Latency |
|------|--------|-----------|-------------------|--------|-------------|-------------|
| fhir_read | FHIR | yes | no | Enabled | 8,421 | 120ms |
| fhir_search | FHIR | yes | no | Enabled | 4,102 | 380ms |
| fhir_create | FHIR | yes | no | Enabled | 1,247 | 250ms |
| fhir_update | FHIR | yes | no | Enabled | 823 | 220ms |
| billing_submit_claim | Billing | yes | **yes** | Enabled | 142 | 890ms |
| billing_scrub_claim | Billing | yes | no | Enabled | 2,847 | 180ms |
| device_ingest_reading | Device | limited | no | Enabled | 847 | 340ms |
| context_get_freshness | Context | limited | no | Enabled | 4,521 | 90ms |
| ... | ... | ... | ... | ... | ... | ... |

Collapsible server sections. Search/filter by tool name, server, PHI level, approval requirement.

**Tab 3 -- Gateway Configuration:**

MCP Gateway settings:

| Setting | Value | Description |
|---------|-------|-------------|
| Global Rate Limit | 100 req/s per agent | Maximum requests per agent per second |
| Circuit Breaker Threshold | 50% error rate / 30s window | Auto-disable server on sustained errors |
| Circuit Breaker Recovery | 60s half-open, 5 test requests | Gradual recovery before full open |
| Audit Log Level | Full (all parameters) | Log tool name + parameters (no PHI) |
| PHI Screening | Enabled | Strip PHI from gateway logs |
| Timeout | 30s per tool call | Kill requests exceeding timeout |
| Max Concurrent per Agent | 10 | Prevent agent from monopolizing server |

**Tab 4 -- Tool Usage Analytics:**

- **Most Called Tools:** Bar chart, top 10 tools by call volume (24h).
- **Slowest Tools:** Bar chart, top 10 by P99 latency.
- **Error-Prone Tools:** Bar chart, top 10 by error rate.
- **Agent-Tool Heatmap:** Matrix showing which agents call which tools and how often.

```
                 | fhir_read | fhir_search | billing_scrub | billing_submit | ...
Clinical Scribe  |    HIGH   |    MED      |     --        |       --       |
Prior Auth       |    MED    |    HIGH     |     LOW       |      MED       |
Denial Mgmt      |    LOW    |    MED      |     MED       |      HIGH      |
```

#### Actions

- Enable/disable individual MCP tools (with confirmation)
- Enable/disable entire MCP server
- Reset circuit breaker manually
- Adjust rate limits per server, per tool, or per agent
- View detailed error logs for a specific tool
- Export tool usage analytics

---

### Section 9: Agent Configuration

**Route:** `/admin/agents`
**Purpose:** Configure, monitor, and tune the 3 LangGraph AI agents (with slots for 2 future agents).

#### Tabs

**Tab 1 -- Agent Directory:**

| Agent | Status | Tasks (24h) | Avg Confidence | Human Review Rate | Avg Duration | Last Run |
|-------|--------|-------------|----------------|-------------------|-------------|----------|
| Clinical Scribe | Active | 47 | 0.91 | 12% | 8.2s | 5 min ago |
| Prior Authorization | Active | 13 | 0.88 | 23% | 14.5s | 22 min ago |
| Denial Management | Active | 8 | 0.85 | 31% | 22.1s | 1h ago |
| Billing Agent | Planned | -- | -- | -- | -- | -- |
| Scheduling Agent | Planned | -- | -- | -- | -- | -- |

Click into any agent for detailed configuration.

**Tab 2 -- Agent Configuration (per agent):**

Detailed configuration panel when you click an agent:

**Clinical Scribe Configuration:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| Confidence Threshold (Auto-Execute) | 0.90 | SOAP notes above this are auto-approved |
| Confidence Threshold (Flag for Review) | 0.75 | Notes between 0.75-0.90 are flagged |
| Confidence Threshold (Escalate) | < 0.75 | Notes below this are halted |
| Audio Quality Minimum (SNR) | 10 dB | Below this, prompt re-recording |
| Max Processing Time | 30s | Kill if processing exceeds this |
| Model | claude-sonnet-4-20250514 | LLM model for note generation |
| Prompt Template Version | v2.3 | Versioned prompt for SOAP generation |
| ICD-10 Auto-Execute Threshold | 0.95 | Higher threshold for coding |
| CPT Auto-Execute Threshold | 0.95 | Higher threshold for coding |
| MCP Tools Allowed | fhir_read, fhir_search, fhir_create, fhir_update, scribe_* | Tools this agent can call |
| FHIR Scopes | patient/*.read, patient/DocumentReference.write, patient/Condition.write | SMART scopes |

**Prior Auth Configuration:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| Auto-Submit Threshold | Never (always human review) | PA submissions always require approval |
| Evidence Gathering Confidence | 0.85 | Min confidence for auto-gathered evidence |
| Payer Rule Refresh Frequency | Weekly | How often to re-fetch payer PA requirements |
| Max Evidence Documents | 10 | Maximum clinical docs to attach |
| Appeal Auto-Generate | 0.80 | Generate appeal letter if confidence >= 0.80 |
| Model | claude-sonnet-4-20250514 | LLM model |
| Prompt Template Version | v1.8 | PA prompt template |

**Denial Management Configuration:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| Auto-Appealable Denial Codes | CO-4, CO-16, CO-18, PR-1, CO-197 | Denial codes eligible for auto-appeal |
| Appeal Letter Confidence Threshold | 0.85 | Min confidence to auto-generate appeal |
| Human Review Required | Always | All appeals reviewed before submission |
| Days Before Deadline Alert | 14 | Warn when appeal deadline approaching |
| Model | claude-sonnet-4-20250514 | LLM model |
| Prompt Template Version | v1.5 | Appeal generation prompt |

**Tab 3 -- Prompt Management:**

Version-controlled prompt templates for each agent:

| Prompt | Agent | Version | Status | Last Updated | Tokens (avg) |
|--------|-------|---------|--------|-------------|-------------|
| SOAP Note Generation | Clinical Scribe | v2.3 | Active | 2026-02-28 | 2,847 |
| ICD-10 Suggestion | Clinical Scribe | v1.9 | Active | 2026-02-25 | 1,203 |
| PA Evidence Summary | Prior Auth | v1.8 | Active | 2026-02-20 | 3,421 |
| Appeal Letter | Denial Mgmt | v1.5 | Active | 2026-02-18 | 4,102 |
| Denial Analysis | Denial Mgmt | v1.3 | Active | 2026-02-15 | 1,847 |

Each prompt shows:
- Full prompt text (editable with diff view)
- Version history with diffs between versions
- A/B test status (if running a prompt experiment)
- Performance metrics per version (confidence scores, human review rates)

**Tab 4 -- Agent Audit Trail:**

Every agent execution is logged:

| Timestamp | Agent | Task ID | Patient | Confidence | Duration | Decision | Reviewer |
|-----------|-------|---------|---------|------------|----------|----------|----------|
| 14:32 ET | Scribe | T-8847 | [redacted] | 0.93 | 7.8s | Auto-approved | -- |
| 14:28 ET | Scribe | T-8846 | [redacted] | 0.81 | 9.2s | Flagged | Dr. Chen |
| 14:15 ET | Prior Auth | T-8845 | [redacted] | 0.87 | 16.1s | Pending review | -- |
| 13:55 ET | Denial Mgmt | T-8844 | [redacted] | 0.78 | 24.3s | Pending review | -- |

Note: Patient identifiers are redacted in the admin view. Click through to view in the appropriate clinical context with proper authorization.

#### Actions

- Adjust confidence thresholds per agent
- Enable/disable agents
- Update prompt templates (with version control)
- Roll back to previous prompt version
- View agent execution traces (link to Langfuse)
- Set agent-specific MCP tool permissions
- Configure A/B prompt experiments
- Export agent performance reports

---

### Section 10: System Configuration

**Route:** `/admin/system`
**Purpose:** Configure platform-wide system settings: caching, context freshness, event bus, and operational parameters.

#### Tabs

**Tab 1 -- Cache Configuration:**

| Cache Layer | Engine | Capacity | Used | Hit Rate | TTL | Eviction Policy |
|------------|--------|----------|------|----------|-----|-----------------|
| Hot Cache | Redis | 2 GB | 1.2 GB | 87% | 15 min | LRU |
| Warm Cache | pgvector embeddings | 10 GB | 4.8 GB | 72% | 24 hours | -- |
| Cold Store | PostgreSQL JSONB | Unlimited | 24.3 GB | -- | -- | -- |

Configurable per cache layer:
- TTL values per context type
- Cache size limits
- Eviction policy
- Flush cache (with confirmation -- impacts performance)

**Tab 2 -- Context Freshness Configuration:**

Migrated and expanded from `/settings/context`:

| Context Type | Freshness Threshold | Auto-Refresh Policy | TTL (Hot) | TTL (Warm) | Priority |
|-------------|--------------------|--------------------|-----------|-----------|----------|
| Encounter | 0.75 | Immediate | 5 min | 1 hour | Critical |
| Clinical Summary | 0.75 | Immediate | 10 min | 4 hours | High |
| Billing | 0.75 | Soon (1 min) | 15 min | 6 hours | High |
| Medication | 0.80 | Immediate | 10 min | 2 hours | Critical |
| Analytics | 0.70 | Batch (15 min) | 30 min | 24 hours | Medium |
| Care Plan | 0.75 | Soon (1 min) | 15 min | 12 hours | Medium |
| Device Vitals | 0.80 | Immediate | 5 min | 1 hour | Critical |
| Scheduling | 0.75 | Immediate | 10 min | 4 hours | High |
| Payer Rules | 0.70 | Batch (15 min) | 1 hour | 7 days | Medium |
| Agent Config | 0.90 | Immediate | 5 min | 1 hour | Critical |
| Clinical Protocols | 0.70 | Batch (15 min) | 1 hour | 7 days | Low |
| Formulary | 0.70 | Batch (15 min) | 4 hours | 30 days | Low |
| Compliance | 0.80 | Immediate | 15 min | 24 hours | High |

Dependency graph visualization (interactive):
- Click any ChangeType to highlight which ContextTypes are affected.
- Click any ContextType to see which ChangeTypes trigger its rehydration.

**Tab 3 -- Event Bus Configuration:**

Redis Streams settings:

| Stream | Consumer Groups | Messages (24h) | Failed | Dead Letter |
|--------|----------------|----------------|--------|-------------|
| patient.* | clinical, rcm, engagement, context_rehydration | 2,847 | 3 | 0 |
| encounter.* | scheduling, analytics, ai_engine | 1,423 | 1 | 0 |
| claim.* | analytics, ai_engine | 892 | 0 | 0 |
| device.* | clinical, context_rehydration, engagement | 5,915 | 12 | 2 |
| agent.* | context_rehydration | 312 | 0 | 0 |
| payer.* | rcm, context_rehydration | 47 | 0 | 0 |

Per-stream settings:
- Max stream length (memory management)
- Consumer group timeout
- Dead letter queue policy (max retries before DLQ)
- Message retention period

**Tab 4 -- Rate Limiting:**

Global and per-category rate limits:

| Endpoint Category | Burst | Sustained | Window | Override Per-Tenant |
|-------------------|-------|-----------|--------|---------------------|
| Auth (login, token) | 10 | 10/min | 1 min | No |
| FHIR Search | 200 | 100/min | 1 min | Yes (Premium: 500) |
| FHIR Write | 100 | 50/min | 1 min | Yes (Premium: 200) |
| AI Pipeline | 20 | 10/min | 1 min | Yes (Enterprise: 50) |
| MCP Gateway | 100 | 50/min | 1 min | No |
| Webhook Ingest | 500 | 200/min | 1 min | No |
| Admin API | 50 | 20/min | 1 min | No |

#### Actions

- Flush individual cache layers (with impact warning)
- Adjust freshness thresholds per context type
- Adjust auto-refresh policies
- View and retry dead-letter queue messages
- Adjust rate limits per category
- View rate limit hit analytics (which endpoints are being throttled)
- Force context rehydration for a specific patient or system context

---

### Section 11: Feature Flags

**Route:** `/admin/features`
**Purpose:** Toggle features on/off globally or per-tenant. Enables gradual rollout, A/B testing, and emergency feature kill switches.

#### Feature Flag Categories

**Clinical Features:**

| Flag | Default | Description | Scope |
|------|---------|-------------|-------|
| `ai_scribe_enabled` | true | Enable ambient AI documentation | Per-tenant |
| `ai_coding_suggestions` | true | AI-suggested ICD-10/CPT codes | Per-tenant |
| `ai_coding_auto_approve` | false | Auto-approve codes above confidence threshold | Per-tenant |
| `whisper_v3_gpu` | true | Use GPU-accelerated Whisper v3 | Global |
| `clinical_decision_support` | false | CDS Hooks integration | Per-tenant |
| `ambient_capture_mode` | "manual" | "manual" / "auto" / "continuous" | Per-tenant |

**Revenue Cycle Features:**

| Flag | Default | Description | Scope |
|------|---------|-------------|-------|
| `auto_eligibility_check` | true | Auto-check eligibility at scheduling | Per-tenant |
| `claims_auto_submit` | false | Auto-submit clean claims (no human review) | Per-tenant |
| `denial_auto_appeal` | false | Auto-generate appeals for eligible denials | Per-tenant |
| `underpayment_detection` | true | Flag underpaid claims vs contract | Per-tenant |
| `x12_835_auto_post` | false | Auto-post payments from remittance | Per-tenant |

**Platform Features:**

| Flag | Default | Description | Scope |
|------|---------|-------------|-------|
| `device_integration` | true | Wearable device data ingestion | Per-tenant |
| `context_rehydration` | true | Event-driven context refresh | Global |
| `a2a_protocol` | false | Agent-to-agent communication | Global |
| `third_party_mcp_apps` | false | Allow third-party MCP app connections | Per-tenant |
| `patient_portal` | false | Patient-facing portal | Per-tenant |
| `telehealth` | false | Telehealth video integration | Per-tenant |
| `bulk_fhir_export` | false | FHIR $export for bulk data | Per-tenant |

**Operational Features:**

| Flag | Default | Description | Scope |
|------|---------|-------------|-------|
| `maintenance_mode` | false | Show maintenance page, block non-admin access | Global |
| `read_only_mode` | false | Allow reads, block all writes | Global |
| `enhanced_logging` | false | Verbose debug logging (performance impact) | Global |
| `synthetic_data_mode` | false | Use synthetic data instead of real data | Per-tenant |

#### Feature Flag Interface

Each flag shows:
- Toggle switch (on/off)
- Current state per tenant (if per-tenant flag)
- Override indicator (if a tenant overrides the global default)
- Last changed timestamp and by whom
- "Emergency Kill" button for critical flags (requires confirmation + reason)

#### Mock Data

```json
{
  "flags": [
    {
      "key": "ai_scribe_enabled",
      "default_value": true,
      "scope": "per-tenant",
      "overrides": {
        "sunshine-ortho": true,
        "palm-beach-derm": true,
        "miami-spine": false
      },
      "last_changed": "2026-02-28T10:00:00Z",
      "changed_by": "admin@medos.health"
    },
    {
      "key": "claims_auto_submit",
      "default_value": false,
      "scope": "per-tenant",
      "overrides": {},
      "last_changed": "2026-02-15T10:00:00Z",
      "changed_by": "admin@medos.health"
    }
  ]
}
```

#### Actions

- Toggle feature flags globally
- Set per-tenant overrides
- View feature flag change history (audit log)
- Emergency kill: disable a feature immediately with reason logged
- Create new feature flags
- Archive deprecated flags

---

### Section 12: Security & Compliance

**Route:** `/admin/security`
**Purpose:** HIPAA compliance dashboard, audit logs, encryption status, security events, and risk assessment status.

#### Tabs

**Tab 1 -- HIPAA Compliance Dashboard:**

**Compliance Score: 94/100** (based on automated checks)

| Category | Score | Status | Details |
|----------|-------|--------|---------|
| Access Controls | 98/100 | Pass | RBAC + ABAC + MFA enforced |
| Audit Logging | 100/100 | Pass | All PHI access logged, 6-year retention |
| Encryption at Rest | 95/100 | Pass | TDE + field-level encryption; 2 patients pending field migration |
| Encryption in Transit | 100/100 | Pass | TLS 1.3 on all connections |
| Backup & Recovery | 90/100 | Pass | Daily backups, tested restore, 15-min RPO |
| Breach Notification | 85/100 | Warn | Playbook documented; tabletop exercise due |
| Business Associate Agreements | 100/100 | Pass | AWS BAA, Anthropic BAA, Auth0 BAA active |
| Risk Assessment | 80/100 | Warn | Last assessment: 2026-02-28; next due: 2026-05-28 |
| Workforce Training | 70/100 | Warn | 3 users pending HIPAA training completion |

Each category expandable to show individual checks.

**Tab 2 -- Audit Log Viewer:**

The enterprise-grade audit log from `audit_log` table:

| Timestamp | Actor | Role | Action | Resource Type | Resource ID | IP Address | Outcome |
|-----------|-------|------|--------|--------------|-------------|-----------|---------|
| 14:32 ET | Dr. Chen | physician | read | Patient | P-1247 | 10.0.1.42 | success |
| 14:30 ET | Maria R. | biller | search | Claim | -- | 10.0.1.45 | success |
| 14:28 ET | System | system | create | AuditEvent | AE-8847 | 10.0.0.1 | success |
| 14:15 ET | James W. | nurse | read | Observation | O-3421 | 10.0.1.48 | success |
| 13:55 ET | Unknown | -- | login_failed | -- | -- | 203.0.113.42 | failure |

Filters:
- Date range
- Actor (user)
- Action type (read, create, update, delete, search, export, login, login_failed)
- Resource type
- Outcome (success, failure)
- Break-the-glass flag

Export: CSV, JSON, PDF (for compliance reporting).

**Tab 3 -- Encryption Status:**

| Layer | Mechanism | Status | Key Rotation | Details |
|-------|-----------|--------|-------------|---------|
| Database TDE | PostgreSQL + KMS | Active | 90 days | Last rotated: 2026-02-15 |
| S3 Storage | SSE-KMS | Active | 365 days | Per-tenant keys |
| Field: SSN | Fernet (deterministic) | Active | 180 days | 2,340 records encrypted |
| Field: MRN | Fernet (deterministic) | Active | 180 days | 2,340 records encrypted |
| Field: DOB | Fernet (randomized) | Active | 180 days | 2,340 records encrypted |
| Field: Subscriber ID | Fernet (deterministic) | Active | 180 days | 1,847 records encrypted |
| Transport | TLS 1.3 | Active | Auto-renew | Cert expires: 2027-02-15 |

Per-tenant encryption status:

| Tenant | KMS Key Status | Encrypted Fields | Unencrypted Records | Key Age (days) |
|--------|---------------|-----------------|---------------------|---------------|
| Sunshine Ortho | Enabled | SSN, MRN, DOB, SubID | 0 | 28 |
| Palm Beach Derm | Enabled | SSN, MRN, DOB, SubID | 0 | 14 |
| Miami Spine | Enabled | SSN, MRN, DOB | 2 (SubID pending) | 7 |

**Tab 4 -- Security Events:**

Real-time security event feed:

| Timestamp | Event | Severity | Source | Details |
|-----------|-------|----------|--------|---------|
| 14:35 ET | Rate limit hit | Info | InputValidator | IP 10.0.1.42 exceeded FHIR search limit |
| 13:55 ET | Failed login (5x) | Warning | Auth | 5 failed attempts from 203.0.113.42 |
| 12:00 ET | Off-hours access | Info | ABAC | Dr. Chen accessed patient at 12:00 ET (within hours) |
| Yesterday | SQL injection attempt | Critical | InputValidator | Blocked pattern in query parameter |
| Yesterday | XSS attempt | High | InputValidator | Blocked script tag in form submission |

**Tab 5 -- Risk Register:**

From the HIPAA Risk Assessment (S5-T01):

| Risk ID | Description | Severity | Likelihood | Current Controls | Residual Risk | Review Date |
|---------|-------------|----------|------------|-----------------|---------------|-------------|
| R-001 | Unauthorized PHI access via application bug | High | Medium | RLS, ABAC, audit logging | Low | 2026-05-28 |
| R-002 | PHI exposure in error messages | High | Low | PHISafeErrorHandler | Very Low | 2026-05-28 |
| R-003 | Cross-tenant data leakage | Critical | Low | Schema isolation + RLS + KMS | Very Low | 2026-05-28 |
| R-004 | LLM hallucination in clinical note | High | Medium | Confidence scoring + human review | Medium | 2026-05-28 |
| R-005 | Clearinghouse credential compromise | High | Low | Secrets Manager + rotation | Low | 2026-05-28 |

#### Actions

- Export compliance report (PDF) for practice due diligence
- Trigger key rotation for a specific tenant
- Review and dismiss security events
- Export audit logs for compliance period
- Mark risk register items as reviewed
- Schedule next risk assessment
- View encryption migration progress
- Generate BAA compliance attestation

---

### Section 13: Data Management

**Route:** `/admin/data`
**Purpose:** Manage data lifecycle: imports, exports, migrations, backups, and data retention.

#### Tabs

**Tab 1 -- Data Import:**

Import tools from S5-T06 (Data Migration Tool):

| Import Type | Source | Format | Status |
|------------|--------|--------|--------|
| Patient Demographics | CSV Upload | CSV with column mapping | Ready |
| Patient Demographics | HL7v2 ADT | HL7v2 message file | Ready |
| Patient Demographics | EHR Bridge (FHIR) | FHIR R4 Bundle | Ready |
| Fee Schedule | CSV Upload | CPT, Description, Amount | Ready |
| Payer Directory | CMS NPPES | Downloaded database | Ready |

**Import Wizard Steps:**
1. Select import type
2. Upload file or configure source
3. Map columns / preview data
4. Run validation (FHIR schema, duplicate detection)
5. Review: total records, valid, invalid, duplicates
6. Confirm import
7. View import report

**Recent Imports:**

| Date | Type | Records | Created | Matched | Duplicates | Errors | Status |
|------|------|---------|---------|---------|-----------|--------|--------|
| 2026-03-01 | Patient CSV | 1,247 | 1,189 | 42 | 12 | 4 | Complete |
| 2026-02-28 | HL7v2 ADT | 83 | 78 | 3 | 1 | 1 | Complete |
| 2026-02-27 | Fee Schedule | 342 | 342 | 0 | 0 | 0 | Complete |

**Tab 2 -- Data Export:**

| Export Type | Format | Scope | Schedule |
|------------|--------|-------|----------|
| FHIR Bulk Export ($export) | NDJSON | All patient resources | On-demand |
| Audit Logs | CSV / JSON | Date range, filters | On-demand |
| Claims Data | CSV | Date range, payer filter | On-demand |
| Analytics Report | PDF | Custom date range | Weekly (scheduled) |
| Full Backup | pg_dump | Tenant schema | Daily (automated) |

**Tab 3 -- Backup & Recovery:**

| Backup Type | Frequency | Last Backup | Last Test Restore | Retention | Status |
|------------|-----------|-------------|-------------------|-----------|--------|
| RDS Snapshot | Daily | 2026-03-01 02:00 ET | 2026-03-01 03:30 ET | 35 days | Healthy |
| S3 Cross-Region | Continuous | 2026-03-01 14:30 ET | 2026-02-28 | 365 days | Healthy |
| Redis Snapshot | Every 6 hours | 2026-03-01 12:00 ET | -- | 7 days | Healthy |
| Audit Log Archive | Monthly | 2026-02-28 | -- | 6 years (HIPAA) | Healthy |

Recovery point objective (RPO): 15 minutes.
Recovery time objective (RTO): 1 hour.

**Tab 4 -- Schema Migrations:**

Per-tenant schema migration status:

| Tenant | Current Version | Latest Version | Pending Migrations | Status |
|--------|----------------|----------------|-------------------|--------|
| Sunshine Ortho | v42 | v42 | 0 | Up to date |
| Palm Beach Derm | v42 | v42 | 0 | Up to date |
| Miami Spine | v41 | v42 | 1 | Pending |

Actions: Run pending migration, rollback last migration, view migration history.

**Tab 5 -- Data Retention:**

| Data Category | Retention Period | Regulation | Auto-Purge | Status |
|--------------|-----------------|-----------|------------|--------|
| Clinical Records (FHIR) | 10 years | FL State Law | No (manual) | Active |
| Audit Logs | 6 years | HIPAA | Auto-archive to S3 Glacier | Active |
| Session Logs | 90 days | Internal policy | Auto-purge | Active |
| Device Readings | 2 years | Internal policy | Auto-archive | Active |
| Cache Data | TTL-based | N/A | Auto-evict | Active |
| LLM Traces (Langfuse) | 1 year | Internal policy | Auto-purge | Active |

#### Actions

- Run data import (wizard)
- Trigger FHIR bulk export
- Export audit logs for compliance period
- Trigger manual backup
- Test backup restore (non-production)
- Run pending schema migrations
- View migration history and rollback if needed
- Configure data retention policies

---

## 4. Priority Ranking

### Tier 1: Must Have for Pilot Demo (Build First)

These sections directly impact the $100M pitch and pilot practice confidence.

| Priority | Section | Why |
|----------|---------|-----|
| P1 | **Admin Dashboard** (Section 1) | First thing investors and practice admins see. Shows the platform is real, healthy, and operational. |
| P2 | **Security & Compliance** (Section 12) | Healthcare buyers demand compliance evidence before signing. The HIPAA dashboard is a trust signal. |
| P3 | **User & Role Management** (Section 5) | Pilot practices need to add providers and staff on day one. RBAC is non-negotiable. |
| P4 | **Tenant Management** (Section 4) | Onboarding wizard is how practices join the platform. Without this, there is no pilot. |
| P5 | **Monitoring & Observability** (Section 2) | Proves the platform is production-grade. Real-time metrics build technical credibility. |

### Tier 2: Important for Pilot Operations (Build Second)

These enable day-to-day management during the pilot.

| Priority | Section | Why |
|----------|---------|-----|
| P6 | **Billing Configuration** (Section 6) | Practices must configure payer contracts and fee schedules before submitting claims. |
| P7 | **Agent Configuration** (Section 9) | Tuning AI confidence thresholds is critical for clinical adoption. |
| P8 | **Integration Management** (Section 7) | EHR bridge and device setup needed for pilot data flow. |
| P9 | **Project Management** (Section 3) | Existing page -- low effort to relocate. Useful for sprint tracking. |

### Tier 3: Nice to Have (Build Later)

These add depth but are not blockers for the pilot.

| Priority | Section | Why |
|----------|---------|-----|
| P10 | **MCP Server Management** (Section 8) | Power-user feature. MCP servers work fine with defaults during pilot. |
| P11 | **System Configuration** (Section 10) | Cache and freshness tuning can use defaults. Advanced config for later. |
| P12 | **Feature Flags** (Section 11) | Important for multi-tenant scaling but less critical with 1-3 tenants. |
| P13 | **Data Management** (Section 13) | Import tools exist; this wraps them in a UI. Manual imports acceptable for pilot. |

### Recommended Build Sequence

**Phase A (Week 1-2):** Sections 1, 4, 5 -- Dashboard, Tenants, Users. The core admin experience.
**Phase B (Week 2-3):** Sections 12, 2 -- Security, Monitoring. Trust and observability layer.
**Phase C (Week 3-4):** Sections 6, 9, 7 -- Billing, Agents, Integrations. Operational configuration.
**Phase D (Week 4+):** Sections 3, 8, 10, 11, 13 -- Project, MCP, System, Flags, Data. Advanced admin.

---

## 5. Dog-Fooding Opportunities

MedOS should use its own infrastructure within the admin interface. This demonstrates platform maturity and creates a compelling demo story ("the admin page runs on the same AI agents that manage your patients").

### 5.1 MCP Tools in the Admin UI

| Admin Feature | MCP Tool Used | How |
|--------------|---------------|-----|
| Tenant creation | `fhir_create` | Create Organization FHIR resource when onboarding a tenant |
| User management | `fhir_search`, `fhir_create` | Practitioners stored as FHIR Practitioner resources |
| Audit log display | `fhir_search` | Query FHIR AuditEvent resources through FHIR MCP |
| Claims analytics widgets | `billing_claims_analytics` | Admin dashboard revenue metrics via Billing MCP |
| Eligibility test | `billing_check_eligibility` | Test clearinghouse connection via Billing MCP |
| Schedule display | `scheduling_available_slots` | Show provider availability in user management |
| Device health | `device_check_alerts` | Display device alert count in integration management |
| Context freshness | `context_get_freshness`, `context_get_staleness_report` | System Config freshness display via Context MCP |
| Force refresh | `context_force_refresh` | Admin triggers context refresh via Context MCP |

### 5.2 AI Agents Powering Admin Features

| Admin Feature | Agent | How |
|--------------|-------|-----|
| **Smart Alerting** | Custom Admin Agent | Analyze alert patterns using Claude. Instead of raw threshold alerts, the agent summarizes: "Billing MCP latency increased 40% after the 2pm deploy. Recommend rollback." |
| **Compliance Report Generation** | Custom Compliance Agent | Generate the HIPAA compliance PDF by having an agent gather all security posture data, audit log summaries, and encryption status, then produce a formatted report. |
| **Import Data Validation** | Clinical Scribe (adapted) | When importing patient demographics via CSV, use the NLU pipeline to detect potential data quality issues (name formatting, address standardization, duplicate detection with confidence scoring). |
| **Denial Pattern Analysis** | Denial Management Agent | In billing config, show AI-analyzed denial patterns: "UHC denials increased 23% this month, primarily CO-16 (missing modifier). Recommend enabling modifier scrubbing rule." |
| **Cost Optimization Suggestions** | Custom Admin Agent | Analyze AWS cost data and suggest optimizations: "LLM token usage is 30% above baseline. 40% of tokens come from re-processing stale contexts. Reducing context TTL from 15min to 10min could save $120/month." |

### 5.3 Context Rehydration for Admin Data

The admin interface itself can use the context rehydration system:

| Admin Context | Data Source | Trigger |
|--------------|-------------|---------|
| Tenant config cache | `public.tenants` table | `tenant.config_updated` event |
| User role cache | Auth service | `user.role_changed` event |
| Payer rules cache | Clearinghouse API | `payer.rules_updated` event |
| Feature flag cache | Feature flag store | `feature.flag_changed` event |
| MCP tool registry | MCP Gateway | `agent.config_updated` event |
| System health cache | Health endpoints | Polling every 60s + `alert.triggered` event |

This means the admin UI benefits from the same sub-200ms context assembly that clinical workflows enjoy. When a payer updates their rules, the billing config page refreshes automatically via WebSocket without a manual page reload.

### 5.4 A2A Protocol in Admin

When the A2A protocol is enabled, the admin page can show inter-agent communication:

- **Agent Communication Map:** Visual graph showing which agents are talking to which other agents, message volume, and latency.
- **Task Delegation View:** When Clinical Scribe delegates coding validation to a future Coding Agent, this shows in the admin as a task chain.
- **External Agent Registry:** View external agents that have registered via Agent Cards at `/.well-known/agent.json`.

### 5.5 Demo Script: "Eating Our Own Dog Food"

For the $100M pitch, the demo script could include:

> "Let me show you the admin console. Notice the system health dashboard -- those metrics come from the same MCP tools that power your clinical workflows. When I click 'Test Clearinghouse Connection', we are calling the exact same billing_check_eligibility tool that your billing staff uses during patient check-in. The compliance report was generated by an AI agent -- the same agent technology that writes your SOAP notes. Everything on this platform uses the same infrastructure, from the patient encounter to the admin console."

---

## 6. Technical Implementation Notes

### Component Architecture

Each admin section should be a Next.js Server Component by default (no PHI in client components per CLAUDE.md rules). Client Components only where interactivity demands it (charts, drag-and-drop, form inputs, WebSocket connections).

```
frontend/
  app/
    admin/
      layout.tsx          # Sidebar navigation, auth guard (admin role check)
      page.tsx            # Redirects to /admin/dashboard
      dashboard/
        page.tsx          # Section 1: Overview
      monitoring/
        page.tsx          # Section 2: Metrics, Alerts, Infra, AI, Cost
      project/
        page.tsx          # Section 3: Kanban (migrated from /project)
      tenants/
        page.tsx          # Section 4: Tenant directory
        onboard/
          page.tsx        # Section 4: Onboarding wizard
      users/
        page.tsx          # Section 5: User & Role management
      billing/
        page.tsx          # Section 6: Payer, Fee, Clearinghouse, Scrub, Coding
      integrations/
        page.tsx          # Section 7: EHR, Devices, Labs, Third-party
      mcp/
        page.tsx          # Section 8: Server overview, Tool inventory, Gateway
      agents/
        page.tsx          # Section 9: Agent directory, Config, Prompts, Audit
      system/
        page.tsx          # Section 10: Cache, Context, Events, Rate limits
      features/
        page.tsx          # Section 11: Feature flags
      security/
        page.tsx          # Section 12: HIPAA, Audit logs, Encryption, Risk
      data/
        page.tsx          # Section 13: Import, Export, Backup, Migrations, Retention
```

### Auth Guard

The `/admin` layout must enforce:
1. Authenticated user (redirect to login if not).
2. Role check: `admin` role required for all sections. Super-admin role for cross-tenant views.
3. MFA must be enabled for admin access.

### Data Fetching Strategy

| Data Type | Strategy | Refresh |
|-----------|----------|---------|
| KPI cards | Server Component + ISR (30s) | Auto via revalidation |
| Charts/metrics | Client Component + SWR (polling 60s) | Periodic |
| Real-time feeds | Client Component + WebSocket | Instant |
| Configuration forms | Server Component (initial), Client Component (edit) | On submit |
| Audit logs | Server Component + pagination | On filter/page change |

### Mock Data Phase

During the initial implementation (while backend APIs are being built), all sections use inline mock data that matches the backend module schemas. This is the same pattern used in `/settings/devices`, `/settings/context`, and `/settings/system` (see [[EPIC-014-admin-system-monitoring]]).

---

## 7. Estimated Scope

| Section | Estimated Lines | Complexity | Dependencies |
|---------|----------------|------------|-------------|
| Admin Layout + Sidebar | ~200 | Low | Auth |
| Dashboard (Overview) | ~600 | Medium | All MCP servers |
| Monitoring & Observability | ~900 | High | MetricsCollector, AlertManager, Langfuse |
| Project Management | ~400 (migrate) | Low | Existing /project page |
| Tenant Management | ~800 | Medium | ADR-002 tenant provisioning |
| User & Role Management | ~700 | Medium | Auth service, RBAC/ABAC |
| Billing Configuration | ~900 | High | Revenue cycle module |
| Integration Management | ~700 | Medium | Device, EHR bridges |
| MCP Server Management | ~600 | Medium | MCP Gateway |
| Agent Configuration | ~800 | High | LangGraph, Langfuse |
| System Configuration | ~700 | Medium | Redis, Context engine |
| Feature Flags | ~500 | Low | Config store |
| Security & Compliance | ~900 | High | Audit, Encryption, HIPAA |
| Data Management | ~700 | Medium | Migration tools, backup |
| **Total** | **~8,900** | -- | -- |

Estimated implementation time: 3-4 weeks at current velocity (with AI tooling).

---

## References

- [[HEALTHCARE_OS_MASTERPLAN]] -- Platform vision and 8-module architecture
- [[System-Architecture-Overview]] -- Technical architecture, API design, event system
- [[ADR-002-multi-tenancy-schema-per-tenant]] -- Tenant isolation via schema + RLS + KMS
- [[ADR-003-ai-agent-framework]] -- Agent framework and confidence routing
- [[ADR-005-mcp-sdk-integration]] -- HIPAAFastMCP SDK approach
- [[ADR-006-patient-context-rehydration]] -- Context rehydration architecture
- [[ADR-007-wearable-iot-integration]] -- Device integration
- [[ADR-008-a2a-agent-communication]] -- A2A protocol
- [[agent-architecture]] -- Agent specifications (5 agents, state machines, audit)
- [[mcp-integration-plan]] -- MCP Gateway, 44 tools, security model
- [[EPIC-010-security-pilot-readiness]] -- Security hardening, encryption, monitoring
- [[EPIC-011-launch-go-live]] -- Sprint 6 go-live checklist
- [[EPIC-014-admin-system-monitoring]] -- Phase 1 admin pages (settings/devices, context, system)
- [[monitoring-rotation-runbook]] -- Monitoring procedures and alert thresholds
- [[admin-workflow-guide]] -- Admin training guide
- [[HIPAA-Deep-Dive]] -- HIPAA compliance requirements
- [[Revenue-Cycle-Deep-Dive]] -- Revenue cycle management
