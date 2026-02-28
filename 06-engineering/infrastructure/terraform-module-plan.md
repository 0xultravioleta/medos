---
type: engineering
date: "2026-02-28"
status: planned
tags:
  - engineering
  - infra
  - phase-1
  - decision
---

# Terraform Module Plan: MedOS AWS Infrastructure

> **Purpose:** Detailed Terraform module specifications for building HIPAA-compliant AWS infrastructure.
> **Timeline:** Sprint 0, Days 1-5 (2026-03-02 to 2026-03-06)
> **References:** [[EPIC-001-aws-infrastructure-foundation]], [[ADR-002-multi-tenancy-schema-per-tenant]], [[AWS-HIPAA-Infrastructure]], [[PHASE-1-EXECUTION-PLAN]]
> **Rule:** All infrastructure MUST be Terraform-managed. Zero ClickOps. See [[HIPAA-Deep-Dive]] for regulatory justification.

---

## Directory Structure

```
terraform/
  backend.tf                          # S3 + DynamoDB state backend (mgmt account)
  versions.tf                         # Provider version pins
  environments/
    dev/
      main.tf                         # Module composition for dev
      terraform.tfvars                # Dev-specific values
      backend.tf                      # Dev state backend config
    staging/
      main.tf
      terraform.tfvars
      backend.tf
    prod/
      main.tf
      terraform.tfvars
      backend.tf
  modules/
    networking/                       # VPC, subnets, NAT, VPC endpoints
      main.tf
      subnets.tf
      endpoints.tf
      security_groups.tf
      flow_logs.tf
      variables.tf
      outputs.tf
    database/                         # RDS PostgreSQL 17 + pgvector
      main.tf
      parameters.tf
      backup.tf
      monitoring.tf
      variables.tf
      outputs.tf
    compute/                          # ECS Fargate, ALB, WAF, ACM
      main.tf
      alb.tf
      waf.tf
      acm.tf
      ecr.tf
      autoscaling.tf
      service_discovery.tf
      variables.tf
      outputs.tf
    cache/                            # ElastiCache Redis 7
      main.tf
      variables.tf
      outputs.tf
    secrets/                          # KMS keys, Secrets Manager
      main.tf
      kms.tf
      rotation.tf
      variables.tf
      outputs.tf
    monitoring/                       # CloudWatch, CloudTrail, GuardDuty, Security Hub
      main.tf
      cloudtrail.tf
      guardduty.tf
      security_hub.tf
      config.tf
      alarms.tf
      dashboards.tf
      variables.tf
      outputs.tf
    iam/                              # Roles, policies, OIDC for GitHub Actions
      main.tf
      ecs_roles.tf
      github_oidc.tf
      service_linked_roles.tf
      variables.tf
      outputs.tf
    storage/                          # S3 buckets (logs, backups, attachments, audit)
      main.tf
      lifecycle.tf
      replication.tf
      variables.tf
      outputs.tf
    bedrock/                          # AWS Bedrock for Claude (HIPAA BAA)
      main.tf
      iam.tf
      endpoints.tf
      logging.tf
      variables.tf
      outputs.tf
```

---

## Module Specifications

### 1. `modules/networking/` -- VPC & Network Isolation

**Purpose:** Create the foundational network layer. Every PHI-handling resource lives in private or database subnets with no direct internet access.

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_vpc.main` | `medos-{env}-vpc` | Primary VPC, DNS support enabled |
| `aws_subnet.public[*]` | `medos-{env}-public-{az}` | ALB placement only (3 AZs) |
| `aws_subnet.private[*]` | `medos-{env}-private-{az}` | ECS tasks, application layer (3 AZs) |
| `aws_subnet.database[*]` | `medos-{env}-db-{az}` | RDS, ElastiCache -- fully isolated (3 AZs) |
| `aws_nat_gateway.main` | `medos-{env}-nat-{az}` | Outbound internet for private subnets |
| `aws_eip.nat[*]` | `medos-{env}-nat-eip-{az}` | Elastic IPs for NAT gateways |
| `aws_internet_gateway.main` | `medos-{env}-igw` | Internet access for public subnets only |
| `aws_route_table.public` | `medos-{env}-rt-public` | Routes to IGW |
| `aws_route_table.private[*]` | `medos-{env}-rt-private-{az}` | Routes to NAT |
| `aws_route_table.database` | `medos-{env}-rt-database` | NO internet route (isolated) |
| `aws_flow_log.main` | `medos-{env}-vpc-flow-log` | ALL traffic logged (HIPAA) |
| `aws_vpc_endpoint.gateway[*]` | `medos-{env}-vpce-{svc}` | S3, DynamoDB gateway endpoints |
| `aws_vpc_endpoint.interface[*]` | `medos-{env}-vpce-{svc}` | ECR, KMS, Secrets Manager, etc. |
| `aws_route53_zone.private` | `medos.local` | Internal service discovery DNS |

**Key Configuration Decisions:**

- **CIDR:** `10.0.0.0/16` -- provides 65,536 IPs, more than enough for all environments
- **3 AZs:** High availability across `us-east-1a`, `us-east-1b`, `us-east-1c`
- **Subnet sizing:** `/24` for each subnet (254 usable IPs per subnet)
  - Public: `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`
  - Private: `10.0.10.0/24`, `10.0.20.0/24`, `10.0.30.0/24`
  - Database: `10.0.100.0/24`, `10.0.200.0/24`, `10.0.201.0/24`
- **NAT strategy:** Single NAT in dev (cost savings), one per AZ in prod (HA)
- **VPC Flow Logs:** ALL traffic, 60-second aggregation, sent to CloudWatch Logs (encrypted)

**HIPAA-Specific Settings:**

- Database subnets have NO route to the internet -- enforced by route table with only local and VPC endpoint routes
- VPC Flow Logs capture ALL traffic (not just REJECT) for forensic analysis
- NACLs on database subnets: allow only port 5432 from private subnets, deny all else
- All VPC endpoint connections use private DNS to avoid traffic leaving the VPC

**VPC Endpoints (required to avoid internet traversal for AWS API calls):**

| Endpoint | Type | Justification |
|---|---|---|
| `s3` | Gateway | RDS backups, log archival, attachments |
| `dynamodb` | Gateway | Terraform state locking |
| `ecr.api` | Interface | Container image metadata |
| `ecr.dkr` | Interface | Container image pulls |
| `ecs` | Interface | ECS API calls |
| `ecs-agent` | Interface | ECS agent communication |
| `ecs-telemetry` | Interface | Container Insights |
| `secretsmanager` | Interface | Secret retrieval at runtime |
| `kms` | Interface | Encryption/decryption operations |
| `logs` | Interface | CloudWatch Logs ingestion |
| `monitoring` | Interface | CloudWatch metrics |
| `ssm` | Interface | Systems Manager for bastion-less access |
| `ssmmessages` | Interface | SSM Session Manager |
| `ec2messages` | Interface | SSM agent communication |
| `sts` | Interface | STS for OIDC/assume-role |
| `bedrock-runtime` | Interface | Claude API calls via Bedrock |

**Dependencies:** None (this is the foundational module)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| NAT Gateway (single) | $35 |
| NAT Gateway data processing (~20GB) | $1 |
| VPC Endpoints (15 interface x $7.30) | $110 |
| Elastic IP | $4 |
| VPC Flow Logs (CloudWatch, ~5GB) | $3 |
| **Subtotal** | **$153** |

> VPC endpoints are the single most expensive "compliance tax" for HIPAA networking. The $110/month is unavoidable to keep AWS API traffic off the public internet.

---

### 2. `modules/database/` -- RDS PostgreSQL 17 + pgvector

**Purpose:** Managed relational database for all FHIR resource storage, using schema-per-tenant isolation per [[ADR-002-multi-tenancy-schema-per-tenant]].

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_db_instance.main` | `medos-{env}-postgres` | Primary PostgreSQL 17 instance |
| `aws_db_instance.read_replica` | `medos-{env}-postgres-ro` | Read replica (prod only) |
| `aws_db_subnet_group.main` | `medos-{env}-db-subnet` | Database subnet placement |
| `aws_db_parameter_group.main` | `medos-{env}-pg17-params` | PostgreSQL 17 parameter tuning |
| `aws_security_group.database` | `medos-{env}-sg-database` | Port 5432 from private subnets only |
| `random_password.db_master` | -- | Master password (stored in Secrets Manager) |
| `aws_secretsmanager_secret.db_master` | `medos/{env}/db-master-password` | Master credential storage |
| `aws_cloudwatch_log_group.postgres` | `/rds/medos-{env}/postgresql` | PostgreSQL query logs |

**Key Configuration Decisions:**

- **Engine:** PostgreSQL 17.2 (latest stable, pgvector support confirmed on RDS)
- **Extensions:** `pgvector` for clinical text embeddings, `pg_stat_statements` for query analysis, `pgaudit` for HIPAA audit logging
- **Storage:** gp3 (baseline 3000 IOPS, 125 MB/s throughput), autoscaling from 100GB to 500GB
- **Backup window:** `03:00-04:00 UTC` (off-peak for US healthcare)
- **Maintenance window:** `Mon:04:00-Mon:05:00 UTC`
- **Deletion protection:** Enabled in staging/prod, disabled in dev
- **Final snapshot:** Always created on deletion (never skip)
- **No TimescaleDB:** RDS does not support it natively. Use PostgreSQL native partitioning with `pg_partman` for time-series audit data. See [[AWS-HIPAA-Infrastructure]] Section 4 note.

**HIPAA-Specific Settings:**

- Encryption at rest via KMS (per-tenant keys for PHI tables, infrastructure key for shared tables)
- SSL enforced (`rds.force_ssl = 1` in parameter group)
- `pgaudit.log = 'all'` -- logs all DDL and DML for compliance audit
- `log_connections = 1`, `log_disconnections = 1` -- track all connection events
- Backup retention: 35 days in prod (HIPAA minimum: 30 days)
- Performance Insights encrypted with KMS
- CloudWatch log exports: `postgresql` and `upgrade` logs
- `log_min_duration_statement = 1000` -- log slow queries (>1s) for performance monitoring without logging all query data

**Parameter Group Settings:**

```
shared_preload_libraries = 'pg_stat_statements,pgvector'
log_statement            = 'ddl'
log_connections          = 1
log_disconnections       = 1
ssl                      = 1
rds.force_ssl            = 1
pgaudit.log              = 'all'
log_min_duration_statement = 1000
max_connections          = 200
shared_buffers           = '{25% of instance RAM}'
effective_cache_size     = '{75% of instance RAM}'
work_mem                 = '64MB'
maintenance_work_mem     = '256MB'
```

**Dependencies:** `networking` (VPC, subnets, security groups), `secrets` (KMS keys)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| db.t4g.medium (2 vCPU, 4 GB RAM, single-AZ) | $58 |
| 100 GB gp3 storage | $12 |
| Automated backups (7-day, ~100GB) | $10 |
| Performance Insights | $0 (free tier) |
| CloudWatch Logs (~2GB) | $1 |
| **Subtotal** | **$81** |

---

### 3. `modules/compute/` -- ECS Fargate, ALB, WAF, ACM

**Purpose:** Container orchestration for all MedOS microservices. Fargate eliminates server management overhead. ALB provides HTTPS termination and path-based routing.

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_ecs_cluster.main` | `medos-{env}-cluster` | Fargate cluster with Container Insights |
| `aws_ecs_service.api` | `medos-{env}-api` | Primary API service |
| `aws_ecs_service.worker` | `medos-{env}-worker` | Background job processor |
| `aws_ecs_task_definition.api` | `medos-api` | API task definition |
| `aws_ecs_task_definition.worker` | `medos-worker` | Worker task definition |
| `aws_lb.main` | `medos-{env}-alb` | Application Load Balancer |
| `aws_lb_target_group.api` | `medos-{env}-api-tg` | API target group |
| `aws_lb_listener.https` | -- | HTTPS listener (port 443) |
| `aws_lb_listener.http_redirect` | -- | HTTP -> HTTPS redirect |
| `aws_acm_certificate.main` | `*.medos.health` | Wildcard TLS certificate |
| `aws_wafv2_web_acl.api` | `medos-{env}-api-waf` | Web Application Firewall |
| `aws_ecr_repository.api` | `medos/api` | API container images |
| `aws_ecr_repository.worker` | `medos/worker` | Worker container images |
| `aws_ecr_repository.migration` | `medos/migration` | Database migration images |
| `aws_service_discovery_private_dns_namespace.main` | `medos.local` | Internal service discovery |
| `aws_appautoscaling_target.api` | -- | Auto-scaling target |
| `aws_appautoscaling_policy.api_cpu` | -- | CPU-based scaling policy |

**Key Configuration Decisions:**

- **Fargate only:** No EC2 instances to manage, patch, or harden
- **Capacity providers:** `FARGATE` for prod, `FARGATE_SPOT` mixed for dev (up to 70% cost savings)
- **Task sizing:** 0.5 vCPU / 1 GB per task (sufficient for FastAPI with async I/O)
- **ALB health check:** `/health` endpoint, 30s interval, 3 consecutive failures before unhealthy
- **TLS policy:** `ELBSecurityPolicy-TLS13-1-2-2021-06` (TLS 1.3 preferred, 1.2 minimum)
- **WAF rules:** AWS Managed Rules (Common Rule Set, Known Bad Inputs, SQL Injection), rate limiting at 2000 req/IP/5min
- **ECR lifecycle:** Keep last 10 tagged images, expire untagged after 7 days
- **Container Insights:** Enabled for CPU/memory/network metrics per task
- **Service discovery:** AWS Cloud Map with `medos.local` private DNS namespace

**HIPAA-Specific Settings:**

- All containers run as non-root user (enforced in Dockerfile)
- Secrets injected via `secrets` block referencing Secrets Manager ARNs (never in env vars or images)
- Log driver: `awslogs` with KMS-encrypted log groups
- No SSH/exec access to containers in prod (use CloudWatch Logs for debugging)
- WAF blocks common injection attacks before traffic reaches the application
- Access logs enabled on ALB, sent to S3 (encrypted)

**Dependencies:** `networking` (VPC, subnets, security groups), `iam` (execution role, task role), `secrets` (KMS, Secrets Manager), `storage` (S3 for ALB logs)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| ECS Fargate (2 tasks, 0.5 vCPU / 1 GB, 24/7) | $29 |
| Application Load Balancer (base) | $16 |
| ALB LCU charges (low traffic) | $5 |
| WAF Web ACL + managed rules | $11 |
| ECR storage (~5GB) | $1 |
| CloudWatch Container Insights | $3 |
| ACM certificate | $0 (free) |
| **Subtotal** | **$65** |

---

### 4. `modules/cache/` -- ElastiCache Redis 7

**Purpose:** In-memory cache for session management, FHIR search result caching, rate limiting counters, and background job queues.

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_elasticache_replication_group.main` | `medos-{env}-redis` | Redis 7 cluster |
| `aws_elasticache_subnet_group.main` | `medos-{env}-redis-subnet` | Database subnet placement |
| `aws_elasticache_parameter_group.main` | `medos-{env}-redis7-params` | Redis 7 parameter tuning |
| `aws_security_group.redis` | `medos-{env}-sg-redis` | Port 6379 from private subnets only |

**Key Configuration Decisions:**

- **Engine:** Redis 7 (latest LTS on ElastiCache)
- **Node type:** `cache.t4g.micro` for dev, `cache.r7g.large` for prod
- **Cluster mode:** Disabled in dev (single node), enabled in prod (3 shards, 1 replica each)
- **Encryption at rest:** KMS key
- **Encryption in transit:** TLS required
- **Authentication:** AUTH token stored in Secrets Manager
- **Eviction policy:** `allkeys-lru` (sessions expire by TTL, LRU for cache overflow)

**HIPAA-Specific Settings:**

- Encryption at rest and in transit (non-negotiable for session data containing user context)
- No PHI stored in Redis -- only session tokens, tenant_id references, and cache keys
- Deployed in database subnets (no internet access)
- AUTH required for all connections
- Auto-failover enabled in prod

**Dependencies:** `networking` (VPC, database subnets, security groups), `secrets` (KMS key, AUTH token)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| cache.t4g.micro (single node) | $12 |
| Backup storage (~1GB) | $0.10 |
| **Subtotal** | **$12** |

---

### 5. `modules/secrets/` -- KMS Keys & Secrets Manager

**Purpose:** Centralized encryption key management and secret storage. Implements the per-tenant KMS key strategy defined in [[ADR-002-multi-tenancy-schema-per-tenant]].

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_kms_key.rds` | `medos-{env}-rds-key` | RDS encryption at rest |
| `aws_kms_key.s3` | `medos-{env}-s3-key` | S3 bucket encryption |
| `aws_kms_key.logs` | `medos-{env}-logs-key` | CloudWatch Logs encryption |
| `aws_kms_key.secrets` | `medos-{env}-secrets-key` | Secrets Manager encryption |
| `aws_kms_key.redis` | `medos-{env}-redis-key` | ElastiCache encryption |
| `aws_kms_key.bedrock` | `medos-{env}-bedrock-key` | Bedrock invocation logs encryption |
| `aws_kms_key.tenant[*]` | `medos-{env}-tenant-{id}` | Per-tenant PHI encryption |
| `aws_kms_alias.infra[*]` | `alias/medos/{env}/{service}` | Human-readable key aliases |
| `aws_kms_alias.tenant[*]` | `alias/medos/{env}/tenant/{id}` | Tenant key aliases |
| `aws_secretsmanager_secret.app` | `medos/{env}/app-secrets` | Application secrets (DB URL, JWT, etc.) |
| `aws_secretsmanager_secret.db_master` | `medos/{env}/db-master` | RDS master password |
| `aws_secretsmanager_secret.redis_auth` | `medos/{env}/redis-auth` | Redis AUTH token |
| `aws_secretsmanager_secret.auth_provider` | `medos/{env}/auth-provider` | OAuth2 client credentials |
| `aws_secretsmanager_secret_rotation.db` | -- | Auto-rotation for DB password (30 days) |

**Key Configuration Decisions:**

- **Infrastructure keys:** One KMS key per service type (RDS, S3, Logs, Secrets, Redis, Bedrock)
- **Tenant keys:** Dynamically created via application code on tenant provisioning (not in base Terraform -- see [[ADR-002-multi-tenancy-schema-per-tenant]] for the provisioning flow)
- **Key rotation:** Automatic annual rotation enabled on all keys
- **Deletion window:** 30 days (maximum allowed) to prevent accidental data loss
- **Key policies:** Least privilege -- each key's policy only grants access to the specific service and application role that needs it
- **Secret rotation:** Database master password auto-rotated every 30 days via Lambda

**HIPAA-Specific Settings:**

- All encryption uses AWS-managed KMS (FIPS 140-2 Level 3 validated HSMs)
- Per-tenant keys enable "crypto-shredding" -- schedule key deletion to irreversibly destroy a tenant's data
- Key usage is logged in CloudTrail for audit
- Secrets Manager enforces encryption with customer-managed KMS key (not default AWS key)
- Secret versions retained for audit trail

**Dependencies:** `iam` (roles that need key access)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| KMS keys (6 infrastructure keys x $1) | $6 |
| KMS API calls (~10K/month) | $0.03 |
| Secrets Manager (5 secrets x $0.40) | $2 |
| Secrets Manager API calls (~5K/month) | $0.03 |
| Lambda rotation function | $0 (free tier) |
| **Subtotal** | **$8** |

> Per-tenant KMS keys add $1/month per tenant. At pilot scale (5 tenants) this is negligible. At 10,000 tenants, budget $10K/month for KMS alone. Consider KMS key hierarchy at scale.

---

### 6. `modules/monitoring/` -- CloudWatch, CloudTrail, GuardDuty, Security Hub

**Purpose:** Full observability and security monitoring stack. HIPAA requires logging of all access to PHI, audit trails for all system changes, and threat detection.

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_cloudtrail.main` | `medos-{env}-trail` | Organization-wide API audit trail |
| `aws_cloudwatch_log_group.cloudtrail` | `/cloudtrail/medos-{env}` | CloudTrail log delivery |
| `aws_cloudwatch_log_group.api` | `/ecs/medos-{env}/api` | API application logs |
| `aws_cloudwatch_log_group.worker` | `/ecs/medos-{env}/worker` | Worker application logs |
| `aws_cloudwatch_log_group.vpc_flow` | `/vpc/medos-{env}/flow-logs` | VPC traffic logs |
| `aws_guardduty_detector.main` | -- | Threat detection |
| `aws_securityhub_account.main` | -- | Security findings aggregation |
| `aws_securityhub_standards_subscription.hipaa` | -- | HIPAA standard |
| `aws_securityhub_standards_subscription.cis` | -- | CIS Benchmark |
| `aws_config_configuration_recorder.main` | -- | Resource configuration recording |
| `aws_config_conformance_pack.hipaa` | -- | HIPAA conformance pack |
| `aws_iam_access_analyzer.main` | `medos-{env}-analyzer` | External access detection |
| `aws_cloudwatch_metric_alarm.api_5xx` | `medos-{env}-api-5xx` | API error rate alarm |
| `aws_cloudwatch_metric_alarm.rds_cpu` | `medos-{env}-rds-cpu` | Database CPU alarm |
| `aws_cloudwatch_metric_alarm.rds_storage` | `medos-{env}-rds-storage` | Database storage alarm |
| `aws_cloudwatch_metric_alarm.rds_connections` | `medos-{env}-rds-connections` | Connection count alarm |
| `aws_cloudwatch_metric_alarm.ecs_cpu` | `medos-{env}-ecs-cpu` | Container CPU alarm |
| `aws_cloudwatch_metric_alarm.ecs_memory` | `medos-{env}-ecs-memory` | Container memory alarm |
| `aws_sns_topic.alerts` | `medos-{env}-alerts` | Alert notification topic |
| `aws_sns_topic.security` | `medos-{env}-security` | Security finding notifications |
| `aws_cloudwatch_dashboard.main` | `medos-{env}-overview` | Operational dashboard |
| `aws_budgets_budget.monthly` | `medos-{env}-monthly` | Monthly spend budget |
| `aws_budgets_budget.per_service[*]` | `medos-{env}-{service}` | Per-service budgets |
| `aws_ce_anomaly_detector.main` | `medos-{env}-anomaly` | Cost anomaly detection |

**Key Configuration Decisions:**

- **CloudTrail:** Multi-region, organization trail, data events for S3 (PHI buckets) and Lambda
- **Log retention:** 365 days in CloudWatch (1 year), then archived to S3 Glacier (7 years total for HIPAA)
- **GuardDuty:** S3 protection enabled, malware protection enabled, EKS disabled (not using EKS)
- **AWS Config:** Records ALL resource types, evaluates HIPAA conformance pack continuously
- **Security Hub:** HIPAA + CIS Benchmark standards enabled, aggregates findings from GuardDuty and Config
- **Alarms:** Tiered -- WARNING at 70% thresholds, CRITICAL at 90%
- **Budget alerts:** 50%, 80%, 100%, 120% of monthly target

**HIPAA-Specific Settings:**

- CloudTrail log file validation enabled (tamper detection)
- All log groups encrypted with KMS (logs may contain metadata related to PHI access patterns)
- CloudTrail S3 bucket: MFA delete enabled, versioning enabled, lifecycle to Glacier at 90 days
- IAM Access Analyzer identifies any external entity with access to MedOS resources
- All alarm notifications go to SNS (which can route to PagerDuty, Slack, or email)

**Dependencies:** `networking` (VPC for flow logs), `storage` (S3 for CloudTrail), `secrets` (KMS for log encryption), `iam` (service roles)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| CloudTrail (management events) | $0 (first trail free) |
| CloudTrail data events (~100K) | $1 |
| CloudWatch Logs ingestion (~15GB) | $8 |
| CloudWatch Logs storage (365 days) | $5 |
| CloudWatch alarms (10 alarms) | $1 |
| CloudWatch dashboard (1) | $3 |
| GuardDuty (baseline) | $10 |
| AWS Config (recording + rules) | $15 |
| Security Hub | $0.0010/check |
| SNS notifications | $0 (free tier) |
| IAM Access Analyzer | $0 (free) |
| **Subtotal** | **$45** |

---

### 7. `modules/iam/` -- Roles, Policies, OIDC

**Purpose:** Define all IAM roles and policies using least-privilege principles. No long-lived access keys anywhere in the system.

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_iam_role.ecs_execution` | `medos-{env}-ecs-execution` | ECS task execution (pull images, write logs) |
| `aws_iam_role.ecs_task_api` | `medos-{env}-ecs-task-api` | API task role (S3, KMS, Secrets, Bedrock) |
| `aws_iam_role.ecs_task_worker` | `medos-{env}-ecs-task-worker` | Worker task role (S3, SQS, KMS) |
| `aws_iam_role.github_actions` | `medos-{env}-github-deploy` | CI/CD deployment via OIDC |
| `aws_iam_role.cloudtrail` | `medos-{env}-cloudtrail` | CloudTrail to CloudWatch Logs |
| `aws_iam_role.vpc_flow_logs` | `medos-{env}-vpc-flow-logs` | VPC Flow Logs to CloudWatch |
| `aws_iam_role.rds_monitoring` | `medos-{env}-rds-monitoring` | RDS Enhanced Monitoring |
| `aws_iam_role.secret_rotation` | `medos-{env}-secret-rotation` | Lambda for secret auto-rotation |
| `aws_iam_role.bedrock_invoke` | `medos-{env}-bedrock-invoke` | Invoke Claude models via Bedrock |
| `aws_iam_openid_connect_provider.github` | -- | GitHub Actions OIDC provider |
| `aws_iam_policy.ecs_secrets_read` | `medos-{env}-ecs-secrets-read` | Read secrets for task injection |
| `aws_iam_policy.s3_attachments` | `medos-{env}-s3-attachments` | Read/write clinical attachments |
| `aws_iam_policy.bedrock_invoke` | `medos-{env}-bedrock-invoke` | Invoke specific Bedrock models |
| `aws_iam_policy.kms_tenant` | `medos-{env}-kms-tenant` | Per-tenant key encrypt/decrypt |

**Key Configuration Decisions:**

- **Zero long-lived keys:** All authentication via IAM roles + OIDC (GitHub Actions), task roles (ECS), or instance profiles
- **Separate task roles:** API and worker services get different IAM roles with different permissions
- **GitHub OIDC:** Repository and branch conditions on the trust policy (`repo:org/medos:ref:refs/heads/main`)
- **Permission boundaries:** Applied to all roles to prevent privilege escalation
- **Cross-account:** Roles use `sts:AssumeRole` with external ID for cross-account access from management

**HIPAA-Specific Settings:**

- All roles have explicit deny for actions outside approved regions
- KMS key policies restrict which roles can encrypt/decrypt per service
- Bedrock invoke role is scoped to specific model IDs (only Claude models allowed)
- Session duration: 1 hour maximum for all assumed roles
- MFA required for all human IAM users (enforced by SCP)

**Dependencies:** None (foundational, but resources reference other module outputs)

**Estimated Cost (dev):** $0 (IAM is free)

---

### 8. `modules/storage/` -- S3 Buckets

**Purpose:** Encrypted object storage for logs, backups, clinical document attachments, and audit archives.

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_s3_bucket.logs` | `medos-{env}-logs-{account_id}` | VPC flow logs, ALB access logs, app logs |
| `aws_s3_bucket.backups` | `medos-{env}-backups-{account_id}` | RDS snapshots, data exports |
| `aws_s3_bucket.attachments` | `medos-{env}-attachments-{account_id}` | Clinical documents (PDFs, images) |
| `aws_s3_bucket.audit` | `medos-{env}-audit-{account_id}` | HIPAA audit trail archives |
| `aws_s3_bucket.cloudtrail` | `medos-{env}-cloudtrail-{account_id}` | CloudTrail log delivery |
| `aws_s3_bucket_versioning.all[*]` | -- | Versioning on all buckets |
| `aws_s3_bucket_server_side_encryption_configuration.all[*]` | -- | KMS encryption |
| `aws_s3_bucket_public_access_block.all[*]` | -- | Block ALL public access |
| `aws_s3_bucket_lifecycle_configuration.logs` | -- | IA at 30d, Glacier at 90d |
| `aws_s3_bucket_lifecycle_configuration.audit` | -- | Glacier at 365d, retain 7 years |
| `aws_s3_bucket_policy.tls_only[*]` | -- | Deny non-TLS requests |
| `aws_s3_bucket_replication_configuration.attachments` | -- | Cross-region replication (prod) |

**Key Configuration Decisions:**

- **Naming:** Include AWS account ID to ensure global uniqueness
- **Versioning:** Enabled on ALL buckets (HIPAA audit trail + accidental deletion protection)
- **Lifecycle policies:**
  - Logs: Standard -> IA (30 days) -> Glacier (90 days) -> Delete (2555 days / 7 years)
  - Backups: Standard -> Glacier (30 days) -> Delete (365 days)
  - Attachments: No lifecycle (active data, always available)
  - Audit: Standard -> Glacier (365 days) -> Delete (2555 days / 7 years)
- **Cross-region replication:** Attachments bucket in prod replicated to `us-west-2` for disaster recovery
- **Object Lock:** Enabled on audit bucket (WORM compliance for immutable audit records)
- **Bucket key:** Enabled to reduce KMS API costs

**HIPAA-Specific Settings:**

- ALL public access blocked at both account level and bucket level
- SSE-KMS encryption (never SSE-S3 for PHI-adjacent data)
- Bucket policies enforce `aws:SecureTransport` (TLS-only access)
- MFA Delete enabled on audit and CloudTrail buckets (prod)
- Access logging enabled (logs bucket logs to itself with prefix)
- Attachments bucket uses per-tenant prefix: `tenant/{tenant_id}/...`

**Dependencies:** `secrets` (KMS keys for encryption)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| S3 storage (~50GB total) | $1 |
| S3 requests (~100K) | $0.50 |
| S3 Glacier (after lifecycle transitions) | $0.20 |
| **Subtotal** | **$2** |

---

### 9. `modules/bedrock/` -- AWS Bedrock for Claude (HIPAA BAA)

**Purpose:** Managed AI model inference via AWS Bedrock, providing HIPAA-compliant access to Claude for clinical documentation, medical coding assistance, and patient communication. Detailed in [[bedrock-claude-setup]].

**Resources Created:**

| Terraform Resource | Name Pattern | Purpose |
|---|---|---|
| `aws_vpc_endpoint.bedrock_runtime` | `medos-{env}-vpce-bedrock` | Private connectivity to Bedrock |
| `aws_iam_role.bedrock_invoke` | `medos-{env}-bedrock-invoke` | Role for invoking Claude models |
| `aws_iam_policy.bedrock_invoke` | `medos-{env}-bedrock-invoke` | Policy scoped to specific models |
| `aws_bedrock_model_invocation_logging_configuration.main` | -- | Audit logging for all invocations |
| `aws_cloudwatch_log_group.bedrock` | `/bedrock/medos-{env}/invocations` | Bedrock invocation logs |
| `aws_s3_bucket.bedrock_logs` | `medos-{env}-bedrock-logs-{account_id}` | Long-term invocation log storage |

**Key Configuration Decisions:**

- **Model access:** Request access to `anthropic.claude-sonnet-4-6-20250514` (primary) and `anthropic.claude-haiku-4-5-20251001` (high-volume, lower cost)
- **VPC endpoint:** `bedrock-runtime` interface endpoint in private subnets (all Bedrock API calls stay within VPC)
- **Invocation logging:** All prompts and completions logged to S3 (encrypted) for HIPAA audit compliance
- **No Bedrock Agents in Phase 1:** Direct API invocation via `bedrock-runtime` -- agent framework handled by LangGraph in the application layer

**HIPAA-Specific Settings:**

- Bedrock covered under AWS BAA (verified January 2026)
- VPC endpoint ensures no PHI traverses the public internet
- Invocation logs encrypted with dedicated KMS key
- IAM policy restricts which models can be invoked (prevent unauthorized model usage)
- Application code must strip direct patient identifiers before sending to Claude when possible, or rely on BAA coverage when clinical context requires full information

**Dependencies:** `networking` (VPC endpoint), `iam` (invoke role), `secrets` (KMS key), `storage` (S3 for logs)

**Estimated Cost (dev):**
| Component | Monthly Cost |
|---|---|
| Bedrock VPC endpoint | $7 |
| Claude Sonnet input (~2M tokens/month) | $6 |
| Claude Sonnet output (~500K tokens/month) | $8 |
| Claude Haiku input (~5M tokens/month) | $4 |
| Claude Haiku output (~1M tokens/month) | $5 |
| Invocation log storage (~5GB) | $0.50 |
| **Subtotal** | **$31** |

> Production AI costs will scale significantly with patient volume. At 100 encounters/day with full AI Scribe pipeline, estimate $500-1,500/month for Bedrock model invocations. See [[bedrock-claude-setup]] for detailed production cost modeling.

---

## Environment-Specific Configuration Matrix

| Configuration | Dev | Staging | Prod |
|---|---|---|---|
| **RDS Instance** | db.t4g.medium (2 vCPU, 4 GB) | db.r7g.large (2 vCPU, 16 GB) | db.r7g.xlarge + read replica (4 vCPU, 32 GB) |
| **RDS Multi-AZ** | No | Yes | Yes |
| **RDS Storage** | 50 GB gp3 | 100 GB gp3 | 200 GB gp3, autoscale to 1 TB |
| **RDS Backup Retention** | 7 days | 14 days | 35 days |
| **RDS Deletion Protection** | No | Yes | Yes |
| **ECS Tasks (API)** | 1 (0.5 vCPU / 1 GB) | 2 (1 vCPU / 2 GB) | 3-10 (1 vCPU / 2 GB, auto-scaling) |
| **ECS Tasks (Worker)** | 1 (0.25 vCPU / 0.5 GB) | 1 (0.5 vCPU / 1 GB) | 2-5 (0.5 vCPU / 1 GB, auto-scaling) |
| **ECS Capacity Provider** | FARGATE_SPOT (70%) + FARGATE | FARGATE | FARGATE |
| **NAT Gateway** | 1 (single AZ) | 1 (single AZ) | 3 (one per AZ, HA) |
| **Redis** | cache.t4g.micro (single) | cache.r7g.large (single + replica) | cache.r7g.large (3 shards, 1 replica) |
| **ALB** | 1 (shared) | 1 (shared) | 1 (dedicated, WAF rules stricter) |
| **WAF Rate Limit** | 5000 req/IP/5min | 3000 req/IP/5min | 2000 req/IP/5min |
| **CloudWatch Log Retention** | 90 days | 180 days | 365 days |
| **S3 Cross-Region Replication** | No | No | Yes (us-west-2) |
| **GuardDuty** | Enabled | Enabled | Enabled + EBS malware scan |
| **Bedrock Models** | Haiku only | Sonnet + Haiku | Sonnet + Haiku |
| **Budget Alert** | $500/month | $2,000/month | $8,000/month |

---

## Cost Estimate Summary

### Monthly Cost by Environment

| Resource | Dev | Staging | Prod |
|---|---|---|---|
| **Networking** (VPC, NAT, endpoints) | $153 | $153 | $263 |
| **Database** (RDS PostgreSQL 17) | $81 | $220 | $850 |
| **Compute** (ECS, ALB, WAF, ACM) | $65 | $130 | $350 |
| **Cache** (ElastiCache Redis 7) | $12 | $110 | $330 |
| **Secrets** (KMS + Secrets Manager) | $8 | $10 | $20 |
| **Monitoring** (CloudWatch, CT, GD, SH) | $45 | $60 | $120 |
| **IAM** | $0 | $0 | $0 |
| **Storage** (S3) | $2 | $5 | $25 |
| **Bedrock** (Claude AI) | $31 | $100 | $1,500 |
| **Data Transfer** (outbound ~50GB) | $5 | $10 | $50 |
| **Total** | **$402/mo** | **$798/mo** | **$3,508/mo** |

### Annual Projections

| Environment | Monthly | Annual | With RI Savings (1yr) |
|---|---|---|---|
| Dev | $402 | $4,824 | $3,860 (~20% savings on RDS + Redis) |
| Staging | $798 | $9,576 | $7,660 |
| Prod | $3,508 | $42,096 | $31,570 (~25% savings on RDS + Redis + Fargate) |
| **Total (all envs)** | **$4,708** | **$56,496** | **$43,090** |

### Cost Optimization Strategies

1. **Phase 1 (Pilot):** Run dev only until staging is needed for pilot practices. Estimated budget: $402/month.
2. **Reserved Instances:** Commit to 1-year RI for RDS and ElastiCache once workload is stable (30-40% savings).
3. **Fargate Spot:** Use FARGATE_SPOT for dev/staging worker tasks (up to 70% savings).
4. **NAT Gateway:** VPC endpoints reduce NAT data processing costs significantly. Monitor with Cost Explorer.
5. **Bedrock usage:** Use Haiku for high-volume, low-complexity tasks (eligibility checks, simple summaries). Reserve Sonnet for complex clinical documentation.
6. **S3 Intelligent-Tiering:** Consider for attachments bucket if access patterns are unpredictable.
7. **Scheduled scaling:** Scale ECS tasks to zero during non-business hours in dev (save ~50% on compute).

---

## Terraform State Management

```
# State backend in management account
S3 Bucket: medos-terraform-state-{mgmt_account_id}
  Key pattern: {environment}/terraform.tfstate
  Encryption: SSE-KMS
  Versioning: Enabled

DynamoDB Table: medos-terraform-locks
  Partition key: LockID
  Billing: PAY_PER_REQUEST

# One state file per environment
dev:     s3://medos-terraform-state-xxx/dev/terraform.tfstate
staging: s3://medos-terraform-state-xxx/staging/terraform.tfstate
prod:    s3://medos-terraform-state-xxx/prod/terraform.tfstate
```

---

## Deployment Order

Modules must be deployed in this order due to dependencies:

```
1. iam           (no dependencies)
2. secrets       (depends on: iam)
3. storage       (depends on: secrets)
4. networking    (depends on: secrets, storage)
5. monitoring    (depends on: networking, storage, secrets, iam)
6. database      (depends on: networking, secrets)
7. cache         (depends on: networking, secrets)
8. compute       (depends on: networking, iam, secrets, storage)
9. bedrock       (depends on: networking, iam, secrets, storage)
```

In practice, with Terraform module composition in the environment `main.tf`, Terraform resolves this automatically via `depends_on` and output references. The order above is for manual understanding and debugging.

---

## Related Documents

- [[EPIC-001-aws-infrastructure-foundation]] -- Task breakdown for building this infrastructure
- [[ADR-002-multi-tenancy-schema-per-tenant]] -- Per-tenant KMS key strategy and schema isolation
- [[AWS-HIPAA-Infrastructure]] -- HIPAA compliance requirements and service eligibility
- [[bedrock-claude-setup]] -- Detailed Bedrock/Claude setup for AI pipeline
- [[PHASE-1-EXECUTION-PLAN]] -- Sprint 0 tasks that implement this plan
- [[HIPAA-Deep-Dive]] -- Regulatory requirements driving these decisions
- [[System-Architecture-Overview]] -- How infrastructure fits into overall system design
