---
type: epic
date: "2026-02-27"
status: planning
priority: 1
tags:
  - project
  - epic
  - phase-1
  - infra
  - aws
  - terraform
owner: ""
target-date: "2026-03-13"
---

# EPIC-001: AWS Infrastructure Foundation

> **Timeline:** Week 1-2 (2026-02-27 to 2026-03-13)
> **Phase:** 1 - Foundation
> **Dependencies:** None (this is the foundational epic)
> **Blocks:** [[EPIC-002-auth-identity-system]], [[EPIC-003-fhir-data-layer]], [[EPIC-004-ai-clinical-documentation]]

## Objective

Stand up the complete AWS infrastructure required to host a HIPAA-compliant, multi-tenant Healthcare OS. Every downstream epic depends on this foundation being secure, reproducible, and cost-controlled from day one.

---

## Timeline (Gantt)

```
Week 1 (Feb 27 - Mar 6)
|----- T1: AWS Organizations + Accounts ---------|
|---- T2: Terraform Module Structure -------------|
|--------- T3: VPC Architecture ------------------|
|------------- T4: KMS Key Strategy --------------|
|------------- T5: S3 Buckets -------------------|

Week 2 (Mar 7 - Mar 13)
|---- T6: RDS PostgreSQL 17 + pgvector -----------|
|---- T7: ECS Fargate Cluster --------------------|
|--------- T8: Security Services -----------------|
|------------- T9: CI/CD Pipeline ----------------|
|------------- T10: Environment Separation --------|
|------------------ T11: Cost Monitoring ---------|
```

---

## Tasks

### T1: AWS Organizations Setup
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** None
**References:** [[System-Architecture-Overview]], [[HIPAA-Deep-Dive]]

**Description:**
Set up AWS Organizations with a multi-account strategy: management account, security account, and workload accounts (dev, staging, prod). Enable SCPs for HIPAA guardrails.

**Subtasks:**
- [ ] Create AWS Organization in management account
- [ ] Create security account (CloudTrail aggregation, GuardDuty delegated admin)
- [ ] Create workload accounts: `medos-dev`, `medos-staging`, `medos-prod`
- [ ] Configure SSO/Identity Center for developer access
- [ ] Apply SCPs: deny non-approved regions, deny public S3, enforce encryption
- [ ] Enable AWS Health API in management account
- [ ] Document account IDs and roles in `terraform/accounts.tf`

**Acceptance Criteria:**
- [ ] Three workload accounts provisioned and accessible via SSO
- [ ] SCPs prevent launching resources in non-approved regions
- [ ] All account setup is codified in Terraform (no manual ClickOps)

---

### T2: Terraform Module Structure
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1
**References:** [[System-Architecture-Overview]]

**Description:**
Establish the Terraform module structure that all infrastructure will be built on. Modules must be reusable across dev/staging/prod with environment-specific variable files.

**Subtasks:**
- [ ] Initialize Terraform backend (S3 + DynamoDB state locking) in management account
- [ ] Create module directory structure:
  ```
  terraform/
  ├── modules/
  │   ├── networking/
  │   ├── database/
  │   ├── compute/
  │   ├── secrets/
  │   ├── monitoring/
  │   └── iam/
  ├── environments/
  │   ├── dev/
  │   ├── staging/
  │   └── prod/
  └── global/
  ```
- [ ] Define common variables: region, environment, project tags, tenant config
- [ ] Implement tagging strategy module (enforce `Environment`, `Project`, `CostCenter`, `HIPAA` tags)
- [ ] Configure Terraform provider with assume-role for cross-account deployment
- [ ] Add `terraform fmt` and `terraform validate` pre-commit hooks
- [ ] Pin all provider versions

**Acceptance Criteria:**
- [ ] `terraform plan` runs successfully for dev environment with no resources
- [ ] State file stored in S3 with encryption and locking
- [ ] Module structure supports environment promotion (dev -> staging -> prod)
- [ ] All resources tagged automatically via default_tags

---

### T3: VPC Architecture
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T2
**References:** [[System-Architecture-Overview]], [[HIPAA-Deep-Dive]]

**Description:**
Deploy a production-grade VPC with public, private, and database subnets across 3 AZs. No PHI-handling resource should ever be in a public subnet.

**Subtasks:**
- [ ] Create VPC module with configurable CIDR (default: `10.0.0.0/16`)
- [ ] Create subnet tiers:
  - Public subnets (3 AZs) - ALB only
  - Private subnets (3 AZs) - ECS tasks, application layer
  - Database subnets (3 AZs) - RDS, ElastiCache, isolated
- [ ] Configure NAT Gateway (single for dev, HA for prod)
- [ ] Set up VPC Flow Logs to S3 (all traffic, HIPAA requirement)
- [ ] Create VPC endpoints:
  - S3 (Gateway)
  - ECR (Interface)
  - Secrets Manager (Interface)
  - CloudWatch Logs (Interface)
  - KMS (Interface)
  - STS (Interface)
- [ ] Configure NACLs for database subnet isolation
- [ ] Set up Route 53 private hosted zone for internal service discovery

**Acceptance Criteria:**
- [ ] Database subnets have no route to internet (verified by route table inspection)
- [ ] VPC Flow Logs capturing all traffic
- [ ] VPC endpoints functional (ECR pulls work without NAT)
- [ ] Network diagram generated and stored in `docs/architecture/`

---

### T4: KMS Key Strategy
**Complexity:** M
**Estimate:** 1 day
**Dependencies:** T2
**References:** [[ADR-002-multi-tenancy-schema-per-tenant]], [[HIPAA-Deep-Dive]]

**Description:**
Design and implement KMS key hierarchy supporting per-tenant encryption. Each tenant's PHI must be encryptable with a tenant-specific key to support crypto-shredding on offboarding.

**Subtasks:**
- [ ] Create KMS module with key policies
- [ ] Create infrastructure keys:
  - `medos-rds-key` - RDS encryption
  - `medos-s3-key` - S3 bucket encryption
  - `medos-logs-key` - CloudWatch Logs encryption
  - `medos-secrets-key` - Secrets Manager encryption
- [ ] Design per-tenant key provisioning:
  - Tenant key created on onboarding via API
  - Key alias pattern: `alias/medos/tenant/{tenant_id}`
  - Key policy grants: application role + tenant admin
- [ ] Implement key rotation (automatic, annual)
- [ ] Document crypto-shredding procedure (key deletion = data destruction)
- [ ] Terraform module accepts tenant_id variable for key creation

**Acceptance Criteria:**
- [ ] All infrastructure keys created and functional
- [ ] Per-tenant key creation tested with a sample tenant
- [ ] Key rotation enabled on all keys
- [ ] Key policies follow least-privilege

---

### T5: S3 Buckets
**Complexity:** S
**Estimate:** 1 day
**Dependencies:** T4
**References:** [[HIPAA-Deep-Dive]], [[System-Architecture-Overview]]

**Description:**
Create S3 buckets for logs, backups, and clinical attachments. All buckets must have encryption, versioning, and lifecycle policies.

**Subtasks:**
- [ ] Create S3 module with HIPAA-compliant defaults:
  - SSE-KMS encryption (using keys from T4)
  - Versioning enabled
  - Public access blocked (account-level + bucket-level)
  - Access logging enabled
- [ ] Create buckets:
  - `medos-{env}-logs` - VPC flow logs, ALB access logs, application logs
  - `medos-{env}-backups` - RDS snapshots, data exports
  - `medos-{env}-attachments` - Clinical document attachments (per-tenant prefix)
  - `medos-{env}-audit` - HIPAA audit trail archives
  - `medos-{env}-terraform-state` - Terraform state (management account)
- [ ] Configure lifecycle policies:
  - Logs: transition to IA at 30 days, Glacier at 90 days, retain 7 years (HIPAA)
  - Backups: retain 1 year, Glacier at 30 days
  - Attachments: no lifecycle (active data)
  - Audit: retain 7 years minimum, Glacier at 1 year
- [ ] Set up cross-region replication for attachments bucket (prod only)
- [ ] Configure bucket policies to enforce TLS-only access

**Acceptance Criteria:**
- [ ] All buckets created with encryption verified
- [ ] Public access confirmed blocked via AWS CLI check
- [ ] Lifecycle policies applied and visible in console
- [ ] Upload/download test passes with KMS encryption

---

### T6: RDS PostgreSQL 17 with pgvector
**Complexity:** L
**Estimate:** 3 days
**Dependencies:** T3, T4
**References:** [[ADR-001-fhir-native-data-model]], [[ADR-002-multi-tenancy-schema-per-tenant]], [[System-Architecture-Overview]]

**Description:**
Deploy RDS PostgreSQL 17 with pgvector extension for clinical text embeddings. Must support schema-per-tenant multi-tenancy with encryption at rest and in transit.

**Subtasks:**
- [ ] Create RDS module with:
  - Engine: PostgreSQL 17
  - Instance class: `db.r6g.large` (dev), `db.r6g.xlarge` (prod)
  - Storage: 100GB gp3, autoscaling to 500GB
  - Multi-AZ: disabled (dev), enabled (staging/prod)
  - Encryption: KMS key from T4
- [ ] Configure parameter group:
  - `shared_preload_libraries = 'pg_stat_statements,pgvector'`
  - `log_statement = 'mod'` (HIPAA audit)
  - `log_connections = 1`
  - `log_disconnections = 1`
  - `ssl = 1` (enforce TLS)
  - `pgaudit.log = 'all'`
- [ ] Place in database subnets (from T3)
- [ ] Create security group: allow port 5432 from private subnets only
- [ ] Enable Performance Insights (encrypted with KMS)
- [ ] Enable Enhanced Monitoring (60s interval)
- [ ] Configure automated backups: 7-day retention (dev), 35-day (prod)
- [ ] Enable pgvector extension: `CREATE EXTENSION vector;`
- [ ] Create initial database: `medos`
- [ ] Create application user with limited privileges (no superuser)
- [ ] Store master password in Secrets Manager
- [ ] Test connection from private subnet via bastion/SSM

**Acceptance Criteria:**
- [ ] PostgreSQL 17 running with pgvector extension enabled
- [ ] Connection only possible from private subnets (security group verified)
- [ ] Encryption at rest confirmed (KMS key)
- [ ] SSL enforced for all connections
- [ ] Automated backups running
- [ ] Performance Insights dashboard accessible

---

### T7: ECS Fargate Cluster
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T3
**References:** [[System-Architecture-Overview]]

**Description:**
Set up ECS Fargate cluster with service discovery, ALB, and auto-scaling. This cluster will host the API services, workers, and background jobs.

**Subtasks:**
- [ ] Create ECS module:
  - Cluster with Container Insights enabled
  - Fargate capacity providers (FARGATE + FARGATE_SPOT for dev)
- [ ] Create Application Load Balancer:
  - Public-facing, in public subnets
  - HTTPS listener (port 443) with ACM certificate
  - HTTP listener (port 80) redirects to HTTPS
  - Target group health checks
  - WAF association (basic OWASP rules)
- [ ] Create service discovery namespace (AWS Cloud Map):
  - Private DNS namespace: `medos.local`
  - Services: `api.medos.local`, `worker.medos.local`
- [ ] Create ECS task execution role and task role:
  - Execution role: pull from ECR, write CloudWatch Logs, read Secrets Manager
  - Task role: S3 access, KMS access, RDS access (per-service)
- [ ] Create ECR repositories:
  - `medos/api`
  - `medos/worker`
  - `medos/migration`
  - Image scanning enabled, lifecycle policy (keep last 10 images)
- [ ] Configure CloudWatch Log groups with KMS encryption
- [ ] Create sample "hello world" task definition and service to validate

**Acceptance Criteria:**
- [ ] ECS cluster visible in console with Container Insights
- [ ] ALB serving HTTPS with valid certificate
- [ ] Sample service reachable via ALB
- [ ] Service discovery resolving internal DNS
- [ ] ECR repositories accepting image pushes

---

### T8: Security Services
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T1, T2
**References:** [[HIPAA-Deep-Dive]], [[SOC2-HITRUST-Roadmap]]

**Description:**
Enable AWS security services required for HIPAA compliance: CloudTrail, GuardDuty, AWS Config, and Security Hub.

**Subtasks:**
- [ ] Enable CloudTrail:
  - Organization trail (all accounts)
  - Multi-region
  - Log to S3 with KMS encryption
  - Enable log file validation
  - Enable data events for S3 (PHI buckets) and Lambda
- [ ] Enable GuardDuty:
  - Delegated admin in security account
  - All member accounts enrolled
  - S3 protection enabled
  - EKS protection enabled (future-proofing)
  - Findings -> SNS -> PagerDuty/Slack
- [ ] Enable AWS Config:
  - Record all resource types
  - Conformance pack: `Operational-Best-Practices-for-HIPAA-Security`
  - Custom rules:
    - RDS encryption enforced
    - S3 public access blocked
    - EBS encryption enforced
    - CloudTrail enabled
- [ ] Enable Security Hub:
  - HIPAA standard enabled
  - CIS Benchmark enabled
  - Aggregate findings from GuardDuty + Config
- [ ] Configure SNS topics for security alerts
- [ ] Create IAM Access Analyzer for external access findings

**Acceptance Criteria:**
- [ ] CloudTrail logging all API calls across all accounts
- [ ] GuardDuty active and sending findings to notification channel
- [ ] AWS Config evaluating HIPAA conformance pack
- [ ] Security Hub score > 80% on initial assessment
- [ ] Alert pipeline tested end-to-end (simulated finding -> notification)

---

### T9: GitHub Actions CI/CD Pipeline
**Complexity:** M
**Estimate:** 2 days
**Dependencies:** T7
**References:** [[System-Architecture-Overview]]

**Description:**
Build CI/CD pipeline using GitHub Actions for automated testing, building, and deploying to ECS Fargate. Pipeline must include security scanning and approval gates.

**Subtasks:**
- [ ] Create GitHub Actions workflow structure:
  ```
  .github/workflows/
  ├── ci.yml           # PR checks: lint, test, security scan
  ├── deploy-dev.yml   # Auto-deploy to dev on merge to main
  ├── deploy-staging.yml  # Deploy to staging (manual trigger)
  ├── deploy-prod.yml  # Deploy to prod (manual trigger + approval)
  └── terraform.yml    # Terraform plan/apply
  ```
- [ ] CI pipeline (`ci.yml`):
  - Rust: `cargo fmt --check`, `cargo clippy`, `cargo test`
  - TypeScript: `npm run lint`, `npm run test`, `npm run build`
  - Security: `trivy` container scan, `cargo audit`, `npm audit`
  - SAST: `semgrep` with HIPAA ruleset
- [ ] Deploy pipeline:
  - Build Docker image with `--no-cache` for code changes
  - Tag with git SHA + `latest`
  - Push to ECR
  - Update ECS task definition
  - Force new deployment
  - Wait for service stability
  - Health check verification
  - Rollback on failure
- [ ] Terraform pipeline:
  - `terraform plan` on PR (comment plan output)
  - `terraform apply` on merge (dev auto, staging/prod manual)
  - State locking verification
- [ ] Configure OIDC for GitHub -> AWS authentication (no long-lived keys)
- [ ] Set up required status checks on `main` branch
- [ ] Configure Dependabot for dependency updates

**Acceptance Criteria:**
- [ ] PR triggers CI checks automatically
- [ ] Merge to main deploys to dev within 5 minutes
- [ ] Staging/prod deploy requires manual approval
- [ ] Container security scan blocks deploy on CRITICAL findings
- [ ] OIDC authentication working (no AWS access keys in GitHub)

---

### T10: Environment Separation
**Complexity:** S
**Estimate:** 1 day
**Dependencies:** T2, T3
**References:** [[System-Architecture-Overview]]

**Description:**
Configure environment-specific Terraform variables and ensure complete isolation between dev, staging, and prod.

**Subtasks:**
- [ ] Create environment variable files:
  - `environments/dev/terraform.tfvars`
  - `environments/staging/terraform.tfvars`
  - `environments/prod/terraform.tfvars`
- [ ] Environment-specific configurations:
  | Config | Dev | Staging | Prod |
  |--------|-----|---------|------|
  | RDS Instance | db.r6g.large | db.r6g.large | db.r6g.xlarge |
  | Multi-AZ | No | Yes | Yes |
  | NAT Gateway | Single | Single | HA (per AZ) |
  | ECS Min Tasks | 1 | 2 | 3 |
  | Backup Retention | 7 days | 14 days | 35 days |
  | Deletion Protection | No | Yes | Yes |
- [ ] Ensure no cross-environment network routes exist
- [ ] Separate Terraform state files per environment
- [ ] Tag all resources with `Environment` tag
- [ ] Verify account-level isolation (dev account cannot access prod resources)

**Acceptance Criteria:**
- [ ] `terraform plan` for each environment shows independent resources
- [ ] No security group rules allow cross-environment traffic
- [ ] Each environment has its own state file
- [ ] Destroying dev has zero impact on staging/prod

---

### T11: Cost Monitoring and Budgets
**Complexity:** S
**Estimate:** 1 day
**Dependencies:** T1
**References:** [[System-Architecture-Overview]]

**Description:**
Set up AWS cost monitoring, budgets, and alerts to prevent runaway spending during development.

**Subtasks:**
- [ ] Create AWS Budgets:
  - Overall monthly budget: $2,000 (dev), $3,000 (staging), $10,000 (prod)
  - Per-service budgets: RDS, ECS, NAT Gateway, data transfer
  - Alert thresholds: 50%, 80%, 100%, 120%
- [ ] Configure Cost Explorer:
  - Enable hourly granularity
  - Create saved reports: by service, by environment, by tenant (via tags)
- [ ] Create cost anomaly detection:
  - Monitor by service
  - SNS notification on anomalies
- [ ] Set up Cost Allocation Tags:
  - `Environment`, `Project`, `CostCenter`, `TenantId`
- [ ] Create monthly cost report automation (Lambda -> S3 -> email)
- [ ] Document expected cost breakdown for MVP phase

**Acceptance Criteria:**
- [ ] Budget alerts configured and tested
- [ ] Cost Explorer showing resource breakdown by tag
- [ ] Anomaly detection active
- [ ] Estimated monthly cost documented for each environment

---

## Dependencies Map

```
T1 (AWS Orgs) ──> T2 (Terraform) ──> T3 (VPC) ──> T6 (RDS)
                                  └──> T10 (Envs)   └──> T7 (ECS)
                  T4 (KMS) ──> T5 (S3)
                           └──> T6 (RDS)
T1 ──> T8 (Security)
T7 ──> T9 (CI/CD)
T1 ──> T11 (Cost)
```

---

## Cross-Epic Dependencies

| This Epic Provides | Required By |
|---|---|
| VPC + Subnets (T3) | [[EPIC-002-auth-identity-system]], [[EPIC-003-fhir-data-layer]] |
| RDS PostgreSQL (T6) | [[EPIC-003-fhir-data-layer]] |
| ECS Cluster (T7) | [[EPIC-002-auth-identity-system]], [[EPIC-003-fhir-data-layer]], [[EPIC-004-ai-clinical-documentation]] |
| KMS Keys (T4) | [[EPIC-003-fhir-data-layer]] (per-tenant encryption) |
| CI/CD Pipeline (T9) | All downstream epics |
| Security Services (T8) | [[EPIC-006-pilot-readiness]] (compliance evidence) |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| AWS account provisioning delays (org approval) | Low | High | Start account requests on day 1; have backup manual setup plan |
| Terraform state corruption | Low | Critical | S3 versioning + DynamoDB locking + daily state backups |
| pgvector not available on RDS 17 in target region | Medium | High | Verify extension availability before region selection; fallback to Aurora |
| NAT Gateway costs exceed budget in dev | Medium | Medium | Use single NAT + VPC endpoints to reduce traffic; FARGATE_SPOT for dev |
| CI/CD OIDC setup complexity | Medium | Low | Document step-by-step; fallback to short-lived access keys if blocked |
| Multi-AZ RDS doubles cost in staging | Low | Medium | Defer Multi-AZ in staging until 60 days before pilot |

---

## Definition of Done

- [ ] All infrastructure provisioned via Terraform (zero ClickOps)
- [ ] `terraform plan` shows no drift in any environment
- [ ] Security Hub HIPAA compliance score > 80%
- [ ] Cost monitoring alerts verified with test notification
- [ ] CI/CD pipeline deploys sample service end-to-end
- [ ] Network diagram documented in `docs/architecture/`
- [ ] All secrets stored in Secrets Manager (none in code or env vars)
- [ ] Runbook for common operations documented in `docs/guides/`
