---
type: checklist
date: "2026-02-28"
status: active
tags:
  - project
  - checklist
  - phase-1
  - pre-development
priority: 0
---

# Pre-Development Checklist (Day 0)

> Everything that must be completed BEFORE Sprint 0 begins. No code gets written until the critical items here are resolved. This document is the gate between "we have an idea" and "we are building."

Related documents: [[HIPAA-Deep-Dive]] | [[SOC2-HITRUST-Roadmap]] | [[AWS-HIPAA-Infrastructure]] | [[Auth-SMART-on-FHIR]] | [[PHASE-1-EXECUTION-PLAN]]

---

## Cost Summary

| Category                        | One-Time Estimate | Monthly Recurring | Annual Recurring | Priority Level                  |
| ------------------------------- | ----------------: | ----------------: | ---------------: | ------------------------------- |
| Legal & Compliance Setup        |    $5,000-$10,000 |                -- |  $25,000-$45,000 | Must Have Before Sprint 0       |
| AWS Account Setup               |                $0 |        $100-$500* |               -- | Must Have Before Sprint 0       |
| Development Tools & Accounts    |              $15+ |          $450-500 |               -- | Must Have Before Sprint 0       |
| Auth Provider Setup             |                $0 |         $100-500+ |               -- | Must Have Before Sprint 0       |
| Communication & Project Mgmt    |                $0 |          $0-$50** |               -- | Must Have Before Sprint 0       |
| Compliance Automation           |                $0 |    $500-$2,000*** |               -- | Nice to Have / During Sprint 0  |
| AI/ML Providers                 |                $0 |        $200-2,000 |               -- | Can Be Done During Sprint 0     |
| Clinical Advisory               |     $2,000-$5,000 |                -- |   $5,000-$15,000 | Nice to Have / During Sprint 0  |
| Financial                       |          $0-$500  |           $30-$80 |               -- | Must Have Before Sprint 0       |
| Security Baseline               |        $500-$1,200 |          $8-$20/u |               -- | Must Have Before Sprint 0       |
| **TOTAL (Conservative)**        | **~$8,000-$17,000** | **~$1,400-$3,700** | **$30,000-$60,000** |                            |

\* AWS costs depend on usage; pre-Sprint 0 is mostly free-tier eligible.
\*\* Assumes free tiers for Slack, Linear, and existing Obsidian setup.
\*\*\* Compliance platforms often offer startup pricing.

> **Minimum cash needed before writing any code: ~$15,000-$25,000** (legal formation, insurance deposit, first month of tooling, security hardware). Plan for 6 months of runway at ~$5,000-$8,000/month in recurring costs before revenue.

---

## Responsibility Key

| Code       | Role                                     |
| ---------- | ---------------------------------------- |
| **CEO**    | Person A -- Business, legal, clinical    |
| **CTO**    | Person B -- Technical, infrastructure    |
| **BOTH**   | Joint responsibility                     |
| **EXT**    | External party (attorney, advisor, etc.) |

---

## 1. Legal & Compliance Setup

> Reference: [[HIPAA-Deep-Dive]] for detailed HIPAA policy requirements

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 1.1 | Form LLC/Corporation (Delaware C-Corp recommended for future fundraising) | $500-$2,000 (one-time) + $400/yr franchise tax | 1-2 weeks | CEO | Must Have Before Sprint 0 | None | - [ ] |
| 1.2 | Register to do business in Florida (foreign qualification) | $150-$300 (one-time) | 1-2 weeks | CEO | Must Have Before Sprint 0 | 1.1 | - [ ] |
| 1.3 | Healthcare attorney on retainer | $5,000-$10,000 retainer + $15-25K/year budget | 1-2 weeks to identify | CEO | Must Have Before Sprint 0 | 1.1 | - [ ] |
| 1.4 | Draft initial HIPAA policies (Privacy Policy, Security Policy, Breach Notification Policy) | Included in attorney retainer or $3,000-$5,000 standalone | 2-4 weeks | CEO + EXT (attorney) | Must Have Before Sprint 0 | 1.3 | - [ ] |
| 1.5 | Cyber liability insurance ($1-5M coverage) | $5,000-$15,000/year | 1-2 weeks | CEO | Must Have Before Sprint 0 | 1.1 | - [ ] |
| 1.6 | D&O insurance | $2,000-$5,000/year | 1-2 weeks | CEO | Nice to Have | 1.1 | - [ ] |
| 1.7 | Draft BAA template (for pilot practices) | Included in attorney retainer or $1,000-$2,000 standalone | 1-2 weeks | CEO + EXT (attorney) | Must Have Before Sprint 0 | 1.3, 1.4 | - [ ] |
| 1.8 | NDA template (for employees, contractors, pilots) | Included in attorney retainer or $500-$1,000 standalone | 1 week | CEO + EXT (attorney) | Must Have Before Sprint 0 | 1.3 | - [ ] |
| 1.9 | HIPAA risk assessment (initial -- required before handling any PHI) | $2,000-$5,000 if outsourced, $0 if DIY with template | 2-4 weeks | BOTH + EXT (attorney) | Must Have Before Sprint 0 | 1.4 | - [ ] |

**Section gate:** Items 1.1, 1.4, 1.7, 1.9 MUST be complete before any PHI touches any system. Items 1.1-1.3 should be initiated in parallel on Day 1.

---

## 2. AWS Account Setup

> Reference: [[AWS-HIPAA-Infrastructure]] for full architecture details

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 2.1 | Create AWS Organizations management account | $0 | 1 hour | CTO | Must Have Before Sprint 0 | 9.1 (bank account for billing) | - [ ] |
| 2.2 | Enable MFA on root account (hardware key recommended) | $50-$60 (YubiKey) | 30 min | CTO | Must Have Before Sprint 0 | 2.1, 10.2 | - [ ] |
| 2.3 | Create IAM Identity Center (SSO) | $0 | 1-2 hours | CTO | Must Have Before Sprint 0 | 2.1 | - [ ] |
| 2.4 | Sign AWS BAA (through AWS Artifact) | $0 | 15 min | CTO | Must Have Before Sprint 0 | 2.1 | - [ ] |
| 2.5 | Create OUs: Security, Infrastructure, Workloads, Sandbox | $0 | 1 hour | CTO | Must Have Before Sprint 0 | 2.1 | - [ ] |
| 2.6 | Create member accounts per OU (Security, Prod, Staging, Sandbox) | $0 | 1-2 hours | CTO | Must Have Before Sprint 0 | 2.5 | - [ ] |
| 2.7 | Enable CloudTrail in all regions | $0-$5/month (S3 storage) | 30 min | CTO | Must Have Before Sprint 0 | 2.6 | - [ ] |
| 2.8 | Enable AWS Config | $0-$10/month | 30 min | CTO | Must Have Before Sprint 0 | 2.6 | - [ ] |
| 2.9 | Enable GuardDuty | $0-$20/month (usage-based) | 30 min | CTO | Must Have Before Sprint 0 | 2.6 | - [ ] |
| 2.10 | Set up billing alerts ($100, $500, $1000 thresholds) | $0 | 15 min | CTO | Must Have Before Sprint 0 | 2.1 | - [ ] |
| 2.11 | Request service limit increases (ECS tasks, RDS instances, KMS keys) | $0 | 1-3 business days (AWS approval) | CTO | Can Be Done During Sprint 0 | 2.6 | - [ ] |

**Section gate:** Items 2.1-2.4 are non-negotiable before any Terraform runs. The AWS BAA (2.4) is a legal requirement for HIPAA-eligible services. Budget ~$50-$100/month for baseline AWS security tooling before any application workloads.

---

## 3. Development Tools & Accounts

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 3.1 | GitHub Organization (Team plan) | $4/user/month ($8/month for 2 seats) | 30 min | CTO | Must Have Before Sprint 0 | None | - [ ] |
| 3.2 | Create main repo: `medos-platform` (private) | $0 | 15 min | CTO | Must Have Before Sprint 0 | 3.1 | - [ ] |
| 3.3 | Create infra repo: `medos-terraform` (private) | $0 | 15 min | CTO | Must Have Before Sprint 0 | 3.1 | - [ ] |
| 3.4 | Branch protection rules (require PR, require CI pass, no force push to main) | $0 | 30 min | CTO | Must Have Before Sprint 0 | 3.2, 3.3 | - [ ] |
| 3.5 | Claude Code Max subscription (2 seats) | $200/seat/month ($400/month) | 15 min | CTO | Must Have Before Sprint 0 | None | - [ ] |
| 3.6 | Cursor Pro (2 seats) | $20/seat/month ($40/month) | 15 min | CTO | Nice to Have | None | - [ ] |
| 3.7 | Domain name registration (medos.health or similar) | $15-$50/year | 30 min | CEO | Must Have Before Sprint 0 | None | - [ ] |
| 3.8 | Terraform Cloud account (free tier for < 5 users) | $0 | 15 min | CTO | Must Have Before Sprint 0 | None | - [ ] |
| 3.9 | Docker Hub account or use ECR | $0 (ECR via AWS) | 15 min | CTO | Can Be Done During Sprint 0 | 2.1 | - [ ] |
| 3.10 | Sentry account for error tracking (free tier) | $0 (free tier: 5K events/month) | 15 min | CTO | Can Be Done During Sprint 0 | None | - [ ] |
| 3.11 | Langfuse account for LLM observability (free tier) | $0 (self-hosted) or $59/month (cloud) | 15 min | CTO | Can Be Done During Sprint 0 | None | - [ ] |

**Section gate:** GitHub repos (3.2, 3.3) and branch protection (3.4) must exist before anyone pushes code. Domain (3.7) should be secured early to avoid squatting.

---

## 4. Auth Provider Setup

> Reference: [[Auth-SMART-on-FHIR]] for authentication architecture details

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 4.1 | Auth0 Enterprise account (or Clerk) -- requires BAA signing | $100-$500/month (Enterprise tier for BAA) | 1-2 days (sales process) | CTO | Must Have Before Sprint 0 | 1.1 (legal entity for BAA) | - [ ] |
| 4.2 | Configure tenant (MedOS) | $0 | 1 hour | CTO | Must Have Before Sprint 0 | 4.1 | - [ ] |
| 4.3 | Set up development application | $0 | 1-2 hours | CTO | Must Have Before Sprint 0 | 4.2 | - [ ] |
| 4.4 | Configure MFA policies | $0 | 1-2 hours | CTO | Must Have Before Sprint 0 | 4.2 | - [ ] |
| 4.5 | Set up social connections if needed (Google for patients) | $0 | 30 min | CTO | Can Be Done During Sprint 0 | 4.2 | - [ ] |

**Section gate:** The BAA with Auth0/Clerk (4.1) is legally required before any user authentication touches the system. Start the sales/procurement process early -- enterprise BAA signing can take 1-2 weeks.

**Decision required:** Auth0 vs Clerk vs AWS Cognito. Evaluate against these criteria:
- BAA availability (required)
- SMART on FHIR support (needed for EHR integration)
- Pricing at scale (1,000+ users)
- Developer experience

---

## 5. Communication & Project Management

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 5.1 | Slack workspace or Discord server (internal comms) | $0 (free tier) | 30 min | CEO | Must Have Before Sprint 0 | None | - [ ] |
| 5.2 | Linear or GitHub Projects for task tracking | $0 (GitHub Projects) or $8/user/month (Linear) | 30 min | CTO | Must Have Before Sprint 0 | 3.1 (if GitHub Projects) | - [ ] |
| 5.3 | Notion or Obsidian (we are using Obsidian) for documentation | $0 (already set up) | -- | BOTH | Must Have Before Sprint 0 | None | - [x] |
| 5.4 | Loom for async video updates | $0 (free tier: 25 videos) | 15 min | CEO | Nice to Have | None | - [ ] |
| 5.5 | Google Workspace or Microsoft 365 (email: @medos.health) | $6-$12/user/month ($12-$24/month for 2 seats) | 1 hour | CEO | Must Have Before Sprint 0 | 3.7 (domain) | - [ ] |

**Section gate:** Professional email (5.5) is needed before any external communication with pilot practices, attorneys, or vendors. Set up the domain (3.7) first.

---

## 6. Compliance Automation

> Reference: [[SOC2-HITRUST-Roadmap]] for compliance framework details

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 6.1 | Evaluate Vanta vs Drata vs Secureframe | $0 | 1-2 days (demos) | BOTH | Nice to Have | None | - [ ] |
| 6.2 | Sign up for chosen platform | $500-$2,000/month (startup pricing available) | 1 day | CEO | Can Be Done During Sprint 0 | 6.1 | - [ ] |
| 6.3 | Connect AWS integration | $0 | 1-2 hours | CTO | Can Be Done During Sprint 0 | 6.2, 2.1 | - [ ] |
| 6.4 | Connect GitHub integration | $0 | 30 min | CTO | Can Be Done During Sprint 0 | 6.2, 3.1 | - [ ] |
| 6.5 | Start policy generation | $0 | 2-4 hours | BOTH | Can Be Done During Sprint 0 | 6.2, 1.4 | - [ ] |
| 6.6 | Begin evidence collection setup | $0 | 2-4 hours | CTO | Can Be Done During Sprint 0 | 6.3, 6.4 | - [ ] |

**Section gate:** Compliance automation is not a Sprint 0 blocker, but starting the evaluation (6.1) early saves time later. Many platforms offer 30-60 day free trials -- use this to evaluate before committing. The real value comes when you start SOC 2 preparation (6-12 months out).

---

## 7. AI/ML Providers

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 7.1 | Anthropic API account (Claude for production) | Usage-based (~$15 per 1M input tokens for Sonnet) | 15 min | CTO | Must Have Before Sprint 0 | None | - [ ] |
| 7.2 | Sign Anthropic BAA (Enterprise plan required for HIPAA) | Custom pricing (Enterprise tier) | 1-4 weeks (sales process) | CEO + CTO | Must Have Before Sprint 0 | 1.1 | - [ ] |
| 7.3 | OpenAI API account (backup/evaluation) | Usage-based (~$3 per 1M input tokens for GPT-4o-mini) | 15 min | CTO | Can Be Done During Sprint 0 | None | - [ ] |
| 7.4 | Hugging Face account (for Whisper v3 model weights) | $0 (model weights are free) | 15 min | CTO | Can Be Done During Sprint 0 | None | - [ ] |
| 7.5 | GPU instance planning (AWS p3/g5 for Whisper inference) | $1.50-$5.00/hour on-demand; reserved instances cheaper | 2-4 hours (planning) | CTO | Can Be Done During Sprint 0 | 2.1 | - [ ] |

**Section gate:** The Anthropic BAA (7.2) is critical -- without it, you cannot send any PHI to Claude in production. Start this process early as enterprise sales cycles can take weeks. For development, use synthetic/de-identified data until the BAA is signed.

**Important:** Until the Anthropic BAA is signed, all development and testing MUST use synthetic patient data only. No real PHI in API calls.

---

## 8. Clinical Advisory

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 8.1 | Identify 2-3 clinical advisors (practicing physicians) | $0 (identification phase) | 2-4 weeks | CEO | Nice to Have | None | - [ ] |
| 8.2 | Advisory agreement template | $500-$1,000 (attorney drafted) or included in retainer | 1 week | CEO + EXT (attorney) | Nice to Have | 1.3, 8.1 | - [ ] |
| 8.3 | Schedule initial clinical workflow observation sessions | $500-$2,000/session (advisor compensation) | 2-4 weeks | CEO | Can Be Done During Sprint 0 | 8.1, 8.2, 1.8 (NDA) | - [ ] |
| 8.4 | Identify 3-5 target pilot practices for outreach | $0 | 2-4 weeks | CEO | Can Be Done During Sprint 0 | 5.5 (professional email) | - [ ] |

**Section gate:** Clinical advisors are not a code blocker, but their input directly shapes what you build. Ideally, have at least one advisor identified (8.1) and one workflow observation (8.3) completed before finalizing Sprint 0 user stories. Advisor compensation is typically equity (0.25-1%) or cash ($200-$500/session) or a combination.

---

## 9. Financial

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 9.1 | Business bank account (Mercury, Relay, or traditional bank) | $0 | 1-3 days | CEO | Must Have Before Sprint 0 | 1.1 (EIN required) | - [ ] |
| 9.2 | Accounting software (QuickBooks, Xero) | $30-$80/month | 30 min | CEO | Must Have Before Sprint 0 | 9.1 | - [ ] |
| 9.3 | Expense tracking system | $0 (included in 9.2) | 30 min | CEO | Must Have Before Sprint 0 | 9.2 | - [ ] |
| 9.4 | Monthly budget template | $0 | 2-4 hours | CEO | Must Have Before Sprint 0 | None | - [ ] |
| 9.5 | Runway calculation spreadsheet | $0 | 2-4 hours | CEO | Must Have Before Sprint 0 | 9.4 | - [ ] |

**Section gate:** The bank account (9.1) is needed before any vendor payments. AWS, domain registrars, and SaaS tools all need a payment method tied to the business entity (not personal cards).

---

## 10. Security Baseline

> Reference: [[HIPAA-Deep-Dive]] for security requirements that drive these decisions

| # | Task | Est. Cost | Time | Owner | Priority | Dependencies | Status |
|---|------|-----------|------|-------|----------|-------------|--------|
| 10.1 | Password manager for team (1Password Business) | $8/user/month ($16/month for 2 seats) | 30 min | CTO | Must Have Before Sprint 0 | None | - [ ] |
| 10.2 | Hardware security keys (YubiKey 5) for critical accounts (AWS root, GitHub admin) | $50-$60 each, 2 per person minimum ($200-$240 one-time) | 1-2 weeks (shipping) | CTO | Must Have Before Sprint 0 | None | - [ ] |
| 10.3 | VPN for development (if needed for compliance) | $5-$15/user/month or $0 if self-hosted via AWS | 1-2 hours | CTO | Nice to Have | 2.1 | - [ ] |
| 10.4 | Encrypted laptops with disk encryption (BitLocker/FileVault) | $0 (built into OS) | 1 hour per machine | BOTH | Must Have Before Sprint 0 | None | - [ ] |
| 10.5 | Mobile Device Management (MDM) if handling PHI on mobile | $3-$10/device/month | 2-4 hours | CTO | Can Be Done During Sprint 0 | None | - [ ] |

**Section gate:** Password manager (10.1) and hardware keys (10.2) must be in place before creating ANY accounts. Order YubiKeys on Day 1 -- shipping takes time. Disk encryption (10.4) is a HIPAA requirement for any device that will access PHI, even in development.

---

## Execution Timeline

### Week 1 (Days 1-7): Foundation

**Parallel Track A (CEO):**
- [ ] 1.1 -- File for incorporation (Delaware C-Corp)
- [ ] 1.3 -- Begin healthcare attorney search (get 3 quotes)
- [ ] 3.7 -- Register domain name
- [ ] 9.4 -- Create monthly budget template
- [ ] 9.5 -- Build runway calculation spreadsheet

**Parallel Track B (CTO):**
- [ ] 10.1 -- Set up 1Password Business
- [ ] 10.2 -- Order YubiKeys (4 minimum -- 2 per person)
- [ ] 10.4 -- Verify disk encryption on all development machines
- [ ] 3.1 -- Create GitHub Organization
- [ ] 3.5 -- Set up Claude Code Max subscriptions

**Joint:**
- [ ] 5.1 -- Set up internal comms (Slack/Discord)
- [ ] 5.3 -- Confirm Obsidian vault structure

### Week 2 (Days 8-14): Accounts & Legal

**Parallel Track A (CEO):**
- [ ] 1.2 -- Florida foreign qualification filing
- [ ] 1.5 -- Get cyber liability insurance quotes
- [ ] 9.1 -- Open business bank account (needs EIN from 1.1)
- [ ] 8.1 -- Begin clinical advisor outreach

**Parallel Track B (CTO):**
- [ ] 2.1-2.6 -- AWS Organizations full setup
- [ ] 2.7-2.10 -- AWS security services (CloudTrail, Config, GuardDuty, billing alerts)
- [ ] 3.2-3.4 -- GitHub repos and branch protection
- [ ] 3.8 -- Terraform Cloud account

**Joint:**
- [ ] 7.2 -- Initiate Anthropic Enterprise/BAA conversation
- [ ] 4.1 -- Begin Auth0/Clerk evaluation and BAA process

### Week 3 (Days 15-21): Policies & Compliance

**Parallel Track A (CEO):**
- [ ] 1.4 -- Begin HIPAA policy drafting with attorney
- [ ] 1.7 -- Draft BAA template
- [ ] 1.8 -- Draft NDA template
- [ ] 5.5 -- Set up Google Workspace/M365 email
- [ ] 9.2 -- Set up accounting software

**Parallel Track B (CTO):**
- [ ] 2.4 -- Sign AWS BAA
- [ ] 4.2-4.4 -- Configure Auth0/Clerk tenant, dev app, MFA
- [ ] 7.1 -- Anthropic API account (for dev with synthetic data)
- [ ] 5.2 -- Set up project tracking (Linear or GitHub Projects)

**Joint:**
- [ ] 6.1 -- Schedule demos with Vanta/Drata/Secureframe
- [ ] 1.9 -- Begin HIPAA risk assessment

### Week 4 (Days 22-28): Finalization & Sprint 0 Prep

**Parallel Track A (CEO):**
- [ ] 1.6 -- D&O insurance (if budget allows)
- [ ] 1.9 -- Complete HIPAA risk assessment
- [ ] 8.2 -- Finalize advisory agreements
- [ ] 8.4 -- Compile pilot practice target list

**Parallel Track B (CTO):**
- [ ] 2.11 -- Request AWS service limit increases
- [ ] 3.9-3.11 -- Docker Hub/ECR, Sentry, Langfuse accounts
- [ ] 7.3-7.5 -- OpenAI account, HuggingFace, GPU planning

**Joint:**
- [ ] Sprint 0 readiness review (see checklist below)
- [ ] Final cost reconciliation against budget

---

## Sprint 0 Readiness Gate

Before Sprint 0 can begin, ALL of the following must be true:

### Legal (non-negotiable)
- [ ] Legal entity formed and EIN received (1.1)
- [ ] Healthcare attorney engaged (1.3)
- [ ] Initial HIPAA policies drafted (1.4)
- [ ] BAA template ready for pilot practices (1.7)
- [ ] NDA template ready (1.8)
- [ ] HIPAA risk assessment initiated or complete (1.9)
- [ ] Cyber liability insurance bound (1.5)

### Infrastructure (non-negotiable)
- [ ] AWS Organizations set up with OUs and member accounts (2.1-2.6)
- [ ] AWS BAA signed (2.4)
- [ ] CloudTrail, Config, GuardDuty enabled (2.7-2.9)
- [ ] MFA on all root and admin accounts (2.2)

### Development (non-negotiable)
- [ ] GitHub repos created with branch protection (3.2-3.4)
- [ ] Claude Code Max active (3.5)
- [ ] Terraform Cloud configured (3.8)
- [ ] Auth provider selected and dev environment configured (4.1-4.3)

### Security (non-negotiable)
- [ ] Password manager deployed to all team members (10.1)
- [ ] Hardware security keys received and configured (10.2)
- [ ] Disk encryption verified on all devices (10.4)

### Financial (non-negotiable)
- [ ] Business bank account open (9.1)
- [ ] Budget and runway documented (9.4, 9.5)

### Nice to have but not blocking
- [ ] Compliance automation platform selected (6.1)
- [ ] Anthropic BAA signed (7.2) -- can use synthetic data until signed
- [ ] Clinical advisor identified (8.1)
- [ ] Professional email operational (5.5)

---

## Notes

### On Ordering

The dependency chain that takes longest is: **Incorporation (1.1) -> EIN -> Bank Account (9.1) -> Vendor Payments**. This is the critical path. File for incorporation on Day 1. Delaware C-Corps can be formed in 24 hours with expedited processing (~$300 extra) but EIN can take 1-4 weeks by mail (apply online for same-day if you have an SSN).

### On BAAs

You will need BAAs with every vendor that touches PHI:
- AWS (2.4) -- signed through AWS Artifact, takes 5 minutes
- Auth0/Clerk (4.1) -- requires Enterprise tier, takes 1-2 weeks
- Anthropic (7.2) -- requires Enterprise plan, takes 2-4 weeks
- Compliance platform (6.2) -- most offer BAAs readily
- Any future SaaS that processes PHI

Do NOT assume a vendor will sign a BAA. Verify before committing to any platform.

### On Costs

The largest ongoing costs in pre-development are:
1. Healthcare attorney retainer: $15-25K/year
2. Cyber liability insurance: $5-15K/year
3. Claude Code Max: $400/month ($4,800/year)
4. Compliance automation: $500-2,000/month ($6,000-24,000/year)

All other tooling is under $100/month total at this stage. Costs scale significantly once you enter production with AWS infrastructure, LLM API calls, and additional team members.

### On Timeline

This checklist assumes a 4-week pre-development phase with 2 people working in parallel. If both founders can dedicate full-time effort, this can compress to 2-3 weeks. The bottlenecks are external dependencies: incorporation processing, attorney availability, insurance underwriting, and enterprise BAA negotiations.

---

*Last updated: 2026-02-28*
*Next review: Before Sprint 0 kickoff*
*Owner: BOTH*
