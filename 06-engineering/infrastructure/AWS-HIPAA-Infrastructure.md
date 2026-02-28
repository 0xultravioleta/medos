---
type: domain-knowledge
date: "2026-02-27"
tags:
  - engineering
  - infrastructure
  - aws
  - hipaa
  - terraform
  - module-g
category: infrastructure
confidence: high
sources: []
---

# AWS HIPAA Infrastructure

This document defines the complete AWS infrastructure strategy for a HIPAA-compliant healthcare SaaS platform. Every architectural decision here is driven by two constraints: regulatory compliance and operational cost-efficiency at early stage. See [[HEALTHCARE_OS_MASTERPLAN]] for overall product context and [[HIPAA-Deep-Dive]] for the regulatory foundation.

---

## 1. AWS HIPAA Eligible Services

AWS maintains a list of services covered under its Business Associate Agreement (BAA). Only services on this list may process, store, or transmit Protected Health Information (PHI).

### Covered Under BAA (services we will use)

| Service | Use Case |
|---------|----------|
| Amazon RDS (PostgreSQL) | Primary database for PHI |
| Amazon ECS (Fargate) | Container orchestration |
| Amazon ECR | Container image registry |
| Amazon S3 | Encrypted object storage (documents, backups) |
| AWS KMS | Encryption key management |
| AWS Secrets Manager | Credential and secret storage |
| Amazon CloudWatch | Monitoring and logging |
| AWS CloudTrail | API audit logging |
| AWS Config | Configuration compliance |
| Amazon GuardDuty | Threat detection |
| AWS WAF | Web application firewall |
| Amazon VPC | Network isolation |
| AWS IAM | Access control |
| Amazon SNS | Alerting and notifications |
| Amazon SQS | Message queuing |
| AWS Lambda | Serverless compute |
| Amazon API Gateway | API management |
| Amazon Route 53 | DNS |
| AWS Certificate Manager | TLS certificates |
| Elastic Load Balancing (ALB) | Load balancing |

### NOT Covered Under BAA (common mistakes)

- **Amazon Lightsail** - tempting for simplicity but NOT HIPAA eligible
- **Amazon WorkSpaces** (some tiers) - verify specific configuration
- **AWS Amplify Hosting** - the hosting layer is NOT covered; use CloudFront + S3 instead
- **Amazon Pinpoint** - NOT covered; do not use for patient communications
- **Amazon SES** - covered, but emails must never contain PHI in the body; use only for transactional notifications with links back to the secure portal
- **Amazon Cognito** - covered, but usernames and custom attributes must not contain PHI

### Critical Rule

> If a service is not on the BAA-eligible list, it must NEVER touch PHI. Period. Not even metadata that could identify a patient. Check the current list at `https://aws.amazon.com/compliance/hipaa-eligible-services-reference/` before adding any new service.

---

## 2. Day 1 AWS Setup for HIPAA

### 2.1 AWS Organizations Structure

```
Management Account (billing only, no workloads)
├── Security OU
│   ├── Log Archive Account (CloudTrail, Config aggregation)
│   └── Security Tooling Account (GuardDuty admin, Security Hub)
├── Infrastructure OU
│   ├── Shared Services Account (ECR, shared networking)
│   └── CI/CD Account (GitHub Actions runners, deployment pipelines)
├── Workloads OU
│   ├── Development Account
│   ├── Staging Account
│   └── Production Account
└── Sandbox OU
    └── Experimentation Account (NO PHI ever)
```

For Phase 1, we can simplify to three accounts: Management, Production, and Development. The structure above is the target state.

### 2.2 CloudTrail Configuration

CloudTrail must be enabled in ALL regions, not just the ones you use. Attackers target inactive regions.

```hcl
resource "aws_cloudtrail" "hipaa_trail" {
  name                          = "hipaa-organization-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id
  is_organization_trail         = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  kms_key_id                    = aws_kms_key.cloudtrail.arn
  include_global_service_events = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3"]
    }
  }

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cloudwatch.arn
}
```

### 2.3 AWS Config Rules for HIPAA

Enable the AWS Config conformance pack for HIPAA:

```hcl
resource "aws_config_conformance_pack" "hipaa" {
  name = "hipaa-security"

  template_body = file("${path.module}/templates/hipaa-conformance-pack.yaml")
}
```

Key rules that must be active:
- `encrypted-volumes` - all EBS volumes encrypted
- `rds-storage-encrypted` - all RDS instances encrypted
- `s3-bucket-server-side-encryption-enabled` - all S3 buckets encrypted
- `cloud-trail-encryption-enabled` - CloudTrail logs encrypted
- `vpc-flow-logs-enabled` - VPC flow logs active
- `iam-password-policy` - strong password requirements
- `multi-factor-authentication-enabled` - MFA on all IAM users
- `restricted-ssh` - no SSH open to 0.0.0.0/0

### 2.4 GuardDuty

```hcl
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = false  # Not using EKS
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
  }
}
```

### 2.5 KMS Key Strategy (Per-Tenant)

Each tenant gets its own KMS key. This enables cryptographic isolation and the ability to "crypto-shred" a tenant's data by scheduling key deletion.

```hcl
resource "aws_kms_key" "tenant" {
  for_each = toset(var.tenant_ids)

  description             = "Encryption key for tenant ${each.key}"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  policy                  = data.aws_iam_policy_document.tenant_key_policy[each.key].json

  tags = {
    Tenant      = each.key
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}
```

### 2.6 VPC Architecture

```
VPC: 10.0.0.0/16
├── Public Subnets (ALB only)
│   ├── 10.0.1.0/24 (AZ-a)
│   └── 10.0.2.0/24 (AZ-b)
├── Private Subnets (ECS Fargate tasks)
│   ├── 10.0.10.0/24 (AZ-a)
│   └── 10.0.20.0/24 (AZ-b)
├── Database Subnets (RDS only, no internet)
│   ├── 10.0.100.0/24 (AZ-a)
│   └── 10.0.200.0/24 (AZ-b)
├── NAT Gateways (one per AZ for HA)
└── VPC Endpoints (S3, ECR, Secrets Manager, CloudWatch, KMS)
```

### 2.7 Security Groups Strategy

- **ALB SG**: Inbound 443 from internet, outbound to ECS SG only
- **ECS SG**: Inbound from ALB SG only, outbound to DB SG and VPC endpoints
- **DB SG**: Inbound 5432 from ECS SG only, no outbound internet
- **VPC Endpoint SG**: Inbound 443 from private subnets only

No security group ever allows 0.0.0.0/0 inbound except the ALB on port 443.

---

## 3. Terraform Modules We Need

### Module Structure

```
terraform/
├── modules/
│   ├── networking/
│   │   ├── main.tf          # VPC, subnets, route tables, NAT
│   │   ├── endpoints.tf     # VPC endpoints for AWS services
│   │   ├── security_groups.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf          # RDS PostgreSQL instance
│   │   ├── parameters.tf    # Parameter group (pgvector, timescale)
│   │   ├── backup.tf        # Automated backup config
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf          # ECS cluster, services, task defs
│   │   ├── alb.tf           # Application Load Balancer
│   │   ├── autoscaling.tf   # Auto-scaling policies
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── secrets/
│   │   ├── main.tf          # Secrets Manager resources
│   │   ├── rotation.tf      # Automatic rotation lambdas
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── monitoring/
│   │   ├── main.tf          # CloudWatch dashboards
│   │   ├── alarms.tf        # Metric alarms
│   │   ├── logs.tf          # Log groups with encryption
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── iam/
│       ├── main.tf          # Roles and policies
│       ├── ecs_roles.tf     # Task execution and task roles
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
└── backend.tf               # S3 + DynamoDB state backend
```

### Example: Networking Module Core

```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.project}-${var.environment}-vpc"
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}

resource "aws_flow_log" "main" {
  vpc_id                   = aws_vpc.main.id
  traffic_type             = "ALL"
  log_destination_type     = "cloud-watch-logs"
  log_destination          = aws_cloudwatch_log_group.vpc_flow_logs.arn
  iam_role_arn             = aws_iam_role.vpc_flow_logs.arn
  max_aggregation_interval = 60
}
```

### Example: VPC Endpoints (avoid internet traversal)

```hcl
# modules/networking/endpoints.tf
locals {
  gateway_endpoints  = ["s3", "dynamodb"]
  interface_endpoints = [
    "ecr.api", "ecr.dkr", "ecs", "ecs-agent", "ecs-telemetry",
    "secretsmanager", "kms", "logs", "monitoring", "ssm",
    "ssmmessages", "ec2messages"
  ]
}

resource "aws_vpc_endpoint" "interface" {
  for_each = toset(local.interface_endpoints)

  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.${each.key}"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]

  tags = {
    Name = "${var.project}-${each.key}-endpoint"
  }
}
```

---

## 4. Database Architecture

### PostgreSQL 17 on RDS

```hcl
# modules/database/main.tf
resource "aws_db_instance" "main" {
  identifier     = "${var.project}-${var.environment}-postgres"
  engine         = "postgres"
  engine_version = "17.2"
  instance_class = var.environment == "prod" ? "db.r6g.large" : "db.t4g.medium"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = var.kms_key_arn

  db_name  = var.database_name
  username = var.master_username
  password = random_password.db_master.result

  multi_az               = var.environment == "prod" ? true : false
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [var.db_security_group_id]

  backup_retention_period   = 35  # HIPAA requires minimum 30 days
  backup_window             = "03:00-04:00"
  maintenance_window        = "Mon:04:00-Mon:05:00"
  copy_tags_to_snapshot     = true
  delete_automated_backups  = false
  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "${var.project}-${var.environment}-final-${formatdate("YYYY-MM-DD", timestamp())}"

  performance_insights_enabled    = true
  performance_insights_kms_key_id = var.kms_key_arn

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  parameter_group_name = aws_db_parameter_group.main.name

  tags = {
    Environment = var.environment
    Compliance  = "HIPAA"
    DataClass   = "PHI"
  }
}
```

### Extensions: pgvector and TimescaleDB

```hcl
resource "aws_db_parameter_group" "main" {
  name   = "${var.project}-${var.environment}-pg17"
  family = "postgres17"

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,pgvector"
  }

  parameter {
    name  = "log_statement"
    value = "ddl"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # Log queries over 1 second
  }

  parameter {
    name  = "ssl"
    value = "1"
    apply_method = "pending-reboot"
  }
}
```

> **Note on TimescaleDB**: AWS RDS does not natively support TimescaleDB as a managed extension. Options: (a) use RDS with native PostgreSQL partitioning for time-series workloads, (b) run TimescaleDB on ECS as a self-managed container, or (c) use Timescale Cloud with a VPC peering connection. For Phase 1, native PostgreSQL partitioning with `pg_partman` is sufficient and avoids operational complexity.

### Schema-Per-Tenant Design

```sql
-- Each tenant gets an isolated schema
CREATE SCHEMA tenant_abc123;
GRANT USAGE ON SCHEMA tenant_abc123 TO app_role;

-- Row-Level Security as defense in depth
ALTER TABLE tenant_abc123.patients ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tenant_abc123.patients
  USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### Backup Strategy

| Type | Frequency | Retention | Encryption |
|------|-----------|-----------|------------|
| Automated snapshots | Daily | 35 days | KMS (per-tenant key) |
| Manual snapshots | Weekly | 1 year | KMS |
| Cross-region replica | Continuous | Real-time | KMS (destination region key) |
| Logical exports (pg_dump) | Monthly | 7 years (compliance) | GPG + S3 SSE-KMS |

---

## 5. Container Architecture (ECS Fargate)

### Task Definition

```hcl
resource "aws_ecs_task_definition" "api" {
  family                   = "${var.project}-api"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = var.execution_role_arn
  task_role_arn            = var.task_role_arn

  container_definitions = jsonencode([
    {
      name  = "api"
      image = "${var.ecr_repository_url}:${var.image_tag}"
      portMappings = [{
        containerPort = 8000
        protocol      = "tcp"
      }]
      secrets = [
        {
          name      = "DATABASE_URL"
          valueFrom = "${var.secrets_arn}:DATABASE_URL::"
        },
        {
          name      = "JWT_SECRET"
          valueFrom = "${var.secrets_arn}:JWT_SECRET::"
        }
      ]
      environment = [
        { name = "ENVIRONMENT", value = var.environment },
        { name = "LOG_LEVEL",   value = "INFO" },
        { name = "AWS_REGION",  value = var.region }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = var.log_group_name
          "awslogs-region"        = var.region
          "awslogs-stream-prefix" = "api"
        }
      }
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}
```

### Secrets Injection

Secrets are injected via the `secrets` block in the task definition, referencing AWS Secrets Manager ARNs. Never bake secrets into container images or environment variables in Terraform state.

```hcl
resource "aws_secretsmanager_secret" "app" {
  name       = "${var.project}/${var.environment}/app-secrets"
  kms_key_id = var.kms_key_arn

  tags = {
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}
```

### Logging Rules (NO PHI in logs)

```python
# In application code - structured logging with PHI filtering
import structlog

def phi_filter(_, __, event_dict):
    """Remove any PHI fields before logging."""
    phi_fields = ['patient_name', 'ssn', 'dob', 'mrn', 'address', 'phone', 'email']
    for field in phi_fields:
        if field in event_dict:
            event_dict[field] = '[REDACTED]'
    return event_dict

structlog.configure(
    processors=[
        phi_filter,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)
```

### Auto-Scaling

```hcl
resource "aws_appautoscaling_target" "api" {
  max_capacity       = var.environment == "prod" ? 10 : 2
  min_capacity       = var.environment == "prod" ? 2 : 1
  resource_id        = "service/${var.cluster_name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "${var.project}-api-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## 6. Networking for HIPAA

### VPC Endpoints

Every AWS service call from private subnets must go through VPC endpoints, not the public internet. This is non-negotiable for HIPAA.

Required VPC endpoints:
- **Gateway**: S3, DynamoDB
- **Interface**: ECR (api + dkr), ECS, Secrets Manager, KMS, CloudWatch Logs, SSM, STS

### PrivateLink for Third-Party Integrations

When integrating with external services (e.g., Stripe for billing, SendGrid for email), use AWS PrivateLink where available. For services without PrivateLink support, route through NAT Gateway and document the data flow in your HIPAA security risk assessment.

### WAF Configuration

```hcl
resource "aws_wafv2_web_acl" "api" {
  name  = "${var.project}-api-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSetMetric"
    }
  }

  rule {
    name     = "RateLimiting"
    priority = 2
    action { block {} }
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitMetric"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "ApiWafMetric"
  }
}
```

### TLS 1.3 Everywhere

- ALB listener: TLS 1.2 minimum (1.3 preferred), using `ELBSecurityPolicy-TLS13-1-2-2021-06`
- RDS connections: enforce `ssl_mode=verify-full` in connection strings
- VPC endpoint connections: TLS by default
- Application-to-application: mTLS where practical

---

## 7. Monitoring and Alerting

### CloudWatch Alarms (Critical)

```hcl
resource "aws_cloudwatch_metric_alarm" "api_5xx" {
  alarm_name          = "${var.project}-api-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "API returning 5xx errors"
  alarm_actions       = [var.sns_topic_arn]

  dimensions = {
    LoadBalancer = var.alb_arn_suffix
  }
}

resource "aws_cloudwatch_metric_alarm" "rds_cpu" {
  alarm_name          = "${var.project}-rds-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "RDS CPU above 80%"
  alarm_actions       = [var.sns_topic_arn]
}

resource "aws_cloudwatch_metric_alarm" "rds_free_storage" {
  alarm_name          = "${var.project}-rds-storage-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 10737418240  # 10 GB in bytes
  alarm_description   = "RDS free storage below 10 GB"
  alarm_actions       = [var.sns_topic_arn]
}
```

### PHI Logging Rules

**What to log:**
- Authentication events (login, logout, failed attempts) -- user ID only, never credentials
- Authorization failures (access denied)
- Data access events (who accessed what record, when) -- record IDs only
- System errors and exceptions -- sanitized, no PHI in stack traces
- API request metadata (method, path, status code, latency, user ID)

**What NOT to log:**
- Patient names, dates of birth, SSNs, MRNs
- Diagnosis codes linked to identifiable patients
- Free-text clinical notes
- Any of the 18 HIPAA identifiers
- Request/response bodies containing PHI
- Database query results containing PHI

### Log Encryption

```hcl
resource "aws_cloudwatch_log_group" "api" {
  name              = "/ecs/${var.project}/${var.environment}/api"
  retention_in_days = 365  # 1 year minimum for HIPAA
  kms_key_id        = var.kms_key_arn

  tags = {
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}
```

---

## 8. Cost Estimation (Phase 1 - Small Scale)

Monthly cost estimate for a production environment serving 1-10 healthcare organizations, low-to-moderate traffic.

| Service | Configuration | Estimated Monthly Cost |
|---------|--------------|----------------------|
| RDS PostgreSQL | db.t4g.medium, Multi-AZ, 100GB gp3 | $150-200 |
| ECS Fargate | 2 tasks, 0.5 vCPU / 1GB each | $30-50 |
| Application Load Balancer | 1 ALB + LCUs | $25-40 |
| NAT Gateway | 2 (one per AZ) + data processing | $70-120 |
| KMS | 5-10 CMKs + API calls | $10-20 |
| Secrets Manager | 10-20 secrets | $5-10 |
| CloudWatch | Logs (10GB), metrics, alarms | $30-60 |
| CloudTrail | Organization trail + S3 storage | $10-20 |
| AWS Config | Rules evaluation | $20-40 |
| GuardDuty | Baseline monitoring | $10-30 |
| S3 | Logs, backups, documents (50GB) | $5-10 |
| ECR | Container images (10GB) | $1-3 |
| WAF | 1 web ACL + managed rules | $10-20 |
| VPC Endpoints | 10-12 interface endpoints | $70-90 |
| Route 53 | 1 hosted zone + queries | $2-5 |
| Data Transfer | Outbound ~50GB | $5-10 |
| **Total** | | **$453-728/month** |

### Cost Optimization Notes

- VPC endpoints are expensive ($7/month each) but required for HIPAA. This is the biggest "compliance tax."
- NAT Gateways at $0.045/GB processed add up. VPC endpoints for AWS services reduce NAT costs.
- Multi-AZ RDS doubles the instance cost but is required for production HIPAA workloads.
- Reserved Instances for RDS (1-year) can save 30-40% once workload is stable.
- For development environments, drop Multi-AZ and use smaller instances to cut costs by 50%.

---

## 9. CI/CD Pipeline

### GitHub Actions with HIPAA Considerations

```yaml
# .github/workflows/deploy.yml
name: Deploy to ECS

on:
  push:
    branches: [main]

permissions:
  id-token: write   # OIDC for AWS auth
  contents: read

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Run Semgrep SAST
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
            p/python

      - name: Run Checkov IaC scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: terraform/
          framework: terraform
          soft_fail: false

  build-and-deploy:
    needs: security-scan
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push container
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build --no-cache --platform linux/amd64 \
            -t $ECR_REGISTRY/${{ vars.ECR_REPO }}:$IMAGE_TAG \
            -t $ECR_REGISTRY/${{ vars.ECR_REPO }}:latest .
          docker push $ECR_REGISTRY/${{ vars.ECR_REPO }}:$IMAGE_TAG
          docker push $ECR_REGISTRY/${{ vars.ECR_REPO }}:latest

      - name: Container image scan
        run: |
          aws ecr start-image-scan \
            --repository-name ${{ vars.ECR_REPO }} \
            --image-id imageTag=${{ github.sha }}
          aws ecr wait image-scan-complete \
            --repository-name ${{ vars.ECR_REPO }} \
            --image-id imageTag=${{ github.sha }}

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster ${{ vars.ECS_CLUSTER }} \
            --service ${{ vars.ECS_SERVICE }} \
            --force-new-deployment

      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster ${{ vars.ECS_CLUSTER }} \
            --services ${{ vars.ECS_SERVICE }}
```

### Secret Management in CI/CD

- Use OIDC federation (not long-lived access keys) for AWS authentication
- Store deployment role ARNs in GitHub Environments, not repository secrets
- ECR image scanning runs automatically after push
- Never pass PHI or database credentials through CI/CD; those live only in Secrets Manager
- Use GitHub Environments with required reviewers for production deployments

### SAST/DAST Tools

| Tool | Purpose | Stage |
|------|---------|-------|
| Semgrep | Static analysis (Python, JS) | Pre-merge |
| Trivy | Container + dependency scanning | Pre-deploy |
| Checkov | Terraform/IaC compliance | Pre-merge |
| ECR Scanning | Container CVE detection | Post-push |
| OWASP ZAP | Dynamic API testing | Post-deploy (staging) |

---

## Architecture Diagram (Text)

```
Internet
    │
    ▼
[Route 53] → [ACM Certificate]
    │
    ▼
[WAF] → [ALB in Public Subnets]
              │
              ▼
    [ECS Fargate in Private Subnets]
         │              │
         ▼              ▼
    [RDS PostgreSQL  [VPC Endpoints]
     in DB Subnets]    │
                        ▼
                   [S3, Secrets Manager,
                    KMS, CloudWatch, ECR]

Monitoring: CloudTrail → S3 (encrypted)
            Config → Compliance Dashboard
            GuardDuty → SNS → PagerDuty
            CloudWatch → Alarms → SNS
```

---

## Next Steps

1. Sign the AWS BAA (Business Associate Agreement) in AWS Artifact -- this is a click-through agreement
2. Set up the Terraform state backend (S3 + DynamoDB for locking)
3. Deploy the networking module first
4. Deploy database and run schema migrations
5. Deploy the first ECS service and verify health checks
6. Enable all monitoring and alerting
7. Run a penetration test before handling real PHI
8. Document everything for SOC 2 evidence collection (see [[SOC2-HITRUST-Roadmap]])

See [[MOC-Architecture]] for how this infrastructure fits into the overall system design.
