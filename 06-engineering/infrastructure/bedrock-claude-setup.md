---
type: engineering
date: "2026-02-28"
status: planned
tags:
  - engineering
  - infra
  - ai
  - phase-1
  - bedrock
  - aws
  - hipaa
  - decision
priority: high
owner: "lxhxr"
---

# AWS Bedrock Claude Setup: HIPAA-Compliant AI for MedOS

> **Purpose:** Detailed plan for integrating AWS Bedrock (Claude) into MedOS for clinical documentation, medical coding, and AI-assisted healthcare workflows.
> **Timeline:** Sprint 2, Days 1-3 (infrastructure), Sprint 2 ongoing (application integration)
> **References:** [[terraform-module-plan]], [[EPIC-004-ai-clinical-documentation]], [[AWS-HIPAA-Infrastructure]], [[HIPAA-Deep-Dive]]
> **Key Dependency:** AWS BAA must be signed BEFORE any PHI touches Bedrock.

---

## 1. Why Bedrock for Claude in Healthcare

### Claude for Healthcare (Announced January 11, 2026 at JPM26)

At the JP Morgan Healthcare Conference in January 2026, Anthropic announced Claude for Healthcare -- a set of capabilities specifically designed for clinical workflows. Key announcements:

- **HIPAA BAA through AWS Bedrock:** Anthropic confirmed that Claude models accessed through AWS Bedrock are covered under AWS's Business Associate Agreement. This means PHI can be processed by Claude when accessed via Bedrock, with the same HIPAA protections as other BAA-covered AWS services.
- **Clinical documentation focus:** Claude demonstrated superior performance on medical licensing exams (USMLE), clinical note generation, and ICD-10/CPT code suggestion accuracy compared to competing models.
- **Structured output reliability:** Claude's ability to produce consistent FHIR-compatible JSON output makes it suitable for integration into EHR workflows where data structure compliance is mandatory.

### Why Bedrock Over Direct API

| Factor | Bedrock | Direct API (api.anthropic.com) |
|---|---|---|
| HIPAA BAA | Covered under AWS BAA | Requires separate BAA with Anthropic |
| Network path | VPC endpoint (private, never leaves AWS) | Public internet (TLS only) |
| Audit logging | CloudTrail + Bedrock invocation logs | Application-level only |
| Data residency | Stays in your AWS region | Routed to Anthropic infrastructure |
| Billing | Consolidated AWS billing | Separate Anthropic billing |
| Compliance evidence | AWS Artifact, Config, CloudTrail | Manual documentation |
| Integration | IAM roles, no API keys | API key management |

**Decision:** Use AWS Bedrock exclusively for all Claude model invocations. Never call the direct Anthropic API with any data that could be PHI or PHI-adjacent.

---

## 2. Claude Model Selection for MedOS

### Claude Sonnet 4.6 (claude-sonnet-4-6-20250514)

**Primary Use Case:** Complex clinical reasoning, long-context analysis, documentation generation

- **Strengths**:
  - Superior clinical reasoning and evidence synthesis
  - Long context window (200K tokens) for comprehensive patient history
  - Advanced structured output generation (reliable JSON parsing)
  - Better at nuanced medical language and clinical decision support

- **MedOS Use Cases**:
  - SOAP note generation from clinical encounters (Stage 3 in pipeline)
  - Prior authorization analysis and letter generation
  - Clinical decision support (drug interactions, missing preventive screenings)
  - Complex chart review and medical record summarization
  - Medication interaction analysis with patient comorbidities
  - Specialty-specific clinical documentation (cardiology, orthopedics, etc.)

- **Performance Characteristics**:
  - Latency: 2-4 seconds per request
  - Output quality: Highest (preferred for clinical accuracy)
  - Token efficiency: Moderate (uses more tokens than Haiku, but necessary for reasoning)

- **Pricing (as of Feb 2026)**:
  - Input: $3.00 per million tokens
  - Output: $15.00 per million tokens
  - Typical SOAP note: ~3,000 input + 2,000 output tokens = $0.039/encounter

### Claude Haiku 4.5 (claude-haiku-4-5-20251001)

**Primary Use Case:** High-volume, low-latency pre-screening and classification tasks

- **Strengths**:
  - Fast inference (sub-second latency)
  - Highly efficient token usage (10x cheaper than Sonnet)
  - Sufficient capability for deterministic classification tasks
  - Ideal for real-time provider UX interactions

- **MedOS Use Cases**:
  - Insurance eligibility verification (rule-based lookups)
  - Clinical data extraction and field classification
  - Suggested ICD-10/CPT coding (from structured SOAP note)
  - Simple FAQ responses (knowledge base retrieval)
  - Routing decisions (which workflow? which specialist?)
  - Batch processing of EHR summary records
  - Patient communication (visit summaries, discharge instructions)

- **Performance Characteristics**:
  - Latency: 200-500ms per request
  - Output quality: High (sufficient for classification, not reasoning)
  - Token efficiency: Excellent (terse, structured responses)

- **Pricing (as of Feb 2026)**:
  - Input: $0.80 per million tokens
  - Output: $4.00 per million tokens
  - Typical coding suggestion: ~2,000 input + 500 output tokens = $0.005/encounter

### Model Selection Decision Flow

```
┌─────────────────────────────────────────────┐
│ Clinical Task Received                      │
└────────────────┬────────────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
   Requires        Deterministic
   Complex        + High Volume
   Reasoning       + Speed Critical
   ?                ?
        │                │
       YES              YES
        │                │
        ▼                ▼
    SONNET 4         HAIKU 4.5
    (reasoning)      (efficiency)
    (3-5 sec)        (<1 sec)
    ($0.039)         ($0.005)
```

### Cost Optimization Strategy

- **Tier 1 (Phase 1)**: Route ~30% to Sonnet (complex cases), ~70% to Haiku (pre-screening)
- **Tier 2 (Pilot)**: As clinical staff learn the system, refine routing -- some providers consistently use only Haiku suggestions
- **Tier 3 (Scale)**: Implement provider-specific routing preferences learned from acceptance patterns

**Cost Impact:**
- Pure Sonnet: $0.039/encounter
- Pure Haiku: $0.005/encounter
- Optimized mix (30/70): $0.015/encounter average

---

## 2A. Terraform Resources

## 2B. AWS Console Setup (One-Time Prerequisites)

### Step 1: Sign AWS BAA (HIPAA Business Associate Agreement)

**CRITICAL:** This must be completed BEFORE any PHI can be transmitted to Bedrock.

**Timeline:** 5-15 business days (AWS legal review queue)

**Process:**
1. Contact AWS Account Manager (assigned to your account)
2. Request HIPAA Business Associate Agreement for Amazon Bedrock
3. AWS sends BAA draft to your legal team
4. Your organization signs the BAA (both parties)
5. AWS applies the BAA to your account in AWS Artifact
6. Verify completion: AWS Console > Artifact > Agreements > Filter by "BAA" > Status = "Active"

**Verification Command:**
```bash
# Check BAA status (requires IAM:ReadAccountAgreements permission)
aws artifact get-account-agreement-summary --agreement-type "BAA"
```

### Step 2: Request Model Access in Bedrock Console

1. **Navigate:** AWS Console > Amazon Bedrock > Model access
2. **Available models section**: Find "Claude Sonnet 4.6" and "Claude Haiku 4.5"
3. **Click "Request access"** for each model
4. **Select use case**:
   - Primary: "Business applications"
   - Additional: "Healthcare and life sciences"
5. **Submit request**
6. **Approval:** Typically within minutes (check email and console)

**Verification Command:**
```bash
# List available models after approval
aws bedrock list-foundation-models --by-provider Anthropic

# Expected output should include:
# - anthropic.claude-sonnet-4-6-20250514
# - anthropic.claude-haiku-4-5-20251001
```

### Step 3: Create KMS Key for Bedrock Logging Encryption

Bedrock invocation logs MUST be encrypted with a customer-managed KMS key (not AWS-managed).

```bash
# Create KMS key
aws kms create-key \
  --description "MedOS Bedrock invocation logs encryption" \
  --region us-east-1 \
  --tags TagKey=Environment,TagValue=production \
         TagKey=Service,TagValue=bedrock

# Create alias for easier reference
aws kms create-alias \
  --alias-name alias/medos-bedrock-logs \
  --target-key-id <key-id-from-above>
```

Store the KMS key ARN in your Terraform variables:
```hcl
# terraform/prod/bedrock.tfvars
bedrock_kms_key_arn = "arn:aws:kms:us-east-1:ACCOUNT:key/KEY-ID"
```

---

## 2C. Terraform Infrastructure Setup

### 2C.1 VPC Endpoint for Bedrock

All Bedrock API calls must stay within the VPC. No PHI should ever traverse the public internet.

```hcl
# modules/bedrock/endpoints.tf
resource "aws_vpc_endpoint" "bedrock_runtime" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.bedrock-runtime"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.bedrock_endpoint.id]

  tags = {
    Name        = "${var.project}-${var.environment}-vpce-bedrock-runtime"
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}

resource "aws_security_group" "bedrock_endpoint" {
  name_prefix = "${var.project}-${var.environment}-bedrock-vpce-"
  vpc_id      = var.vpc_id

  ingress {
    description     = "HTTPS from ECS tasks"
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [var.ecs_security_group_id]
  }

  tags = {
    Name        = "${var.project}-${var.environment}-sg-bedrock-vpce"
    Environment = var.environment
  }
}
```

### 2C.2 IAM Role and Policy for Bedrock Invocation

Scoped to specific model IDs to prevent unauthorized model usage.

```hcl
# modules/bedrock/iam.tf
data "aws_iam_policy_document" "bedrock_invoke" {
  statement {
    sid    = "InvokeAllowedModels"
    effect = "Allow"
    actions = [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ]
    resources = [
      "arn:aws:bedrock:${var.region}::foundation-model/anthropic.claude-sonnet-4-6-20250514",
      "arn:aws:bedrock:${var.region}::foundation-model/anthropic.claude-haiku-4-5-20251001"
    ]
  }

  statement {
    sid    = "DenyAllOtherModels"
    effect = "Deny"
    actions = [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream"
    ]
    not_resources = [
      "arn:aws:bedrock:${var.region}::foundation-model/anthropic.claude-sonnet-4-6-20250514",
      "arn:aws:bedrock:${var.region}::foundation-model/anthropic.claude-haiku-4-5-20251001"
    ]
  }
}

resource "aws_iam_policy" "bedrock_invoke" {
  name   = "${var.project}-${var.environment}-bedrock-invoke"
  policy = data.aws_iam_policy_document.bedrock_invoke.json

  tags = {
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}

# Attach to ECS task role for the API service
resource "aws_iam_role_policy_attachment" "api_bedrock" {
  role       = var.ecs_task_role_name
  policy_arn = aws_iam_policy.bedrock_invoke.arn
}
```

### 2C.3 Invocation Logging Configuration

HIPAA requires audit trails for all AI model interactions that process clinical data. Bedrock invocation logging captures inputs and outputs for compliance review.

```hcl
# modules/bedrock/logging.tf
resource "aws_bedrock_model_invocation_logging_configuration" "main" {
  logging_config {
    embedding_data_delivery_enabled = false

    cloudwatch_config {
      log_group_name = aws_cloudwatch_log_group.bedrock.name
      role_arn       = aws_iam_role.bedrock_logging.arn

      large_data_delivery_s3_config {
        bucket_name = var.bedrock_logs_bucket_name
        key_prefix  = "invocation-logs/"
      }
    }

    s3_config {
      bucket_name = var.bedrock_logs_bucket_name
      key_prefix  = "invocation-logs-full/"
    }

    text_data_delivery_enabled  = true
    image_data_delivery_enabled = false
  }
}

resource "aws_cloudwatch_log_group" "bedrock" {
  name              = "/bedrock/${var.project}-${var.environment}/invocations"
  retention_in_days = 365
  kms_key_id        = var.bedrock_kms_key_arn

  tags = {
    Environment = var.environment
    Compliance  = "HIPAA"
    DataClass   = "PHI-Adjacent"
  }
}

resource "aws_iam_role" "bedrock_logging" {
  name = "${var.project}-${var.environment}-bedrock-logging"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "bedrock.amazonaws.com" }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "aws:SourceAccount" = var.account_id
        }
      }
    }]
  })

  tags = {
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}

resource "aws_iam_role_policy" "bedrock_logging" {
  name = "bedrock-logging"
  role = aws_iam_role.bedrock_logging.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "${aws_cloudwatch_log_group.bedrock.arn}:*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::${var.bedrock_logs_bucket_name}/invocation-logs/*"
      },
      {
        Effect = "Allow"
        Action = [
          "kms:GenerateDataKey",
          "kms:Decrypt"
        ]
        Resource = var.bedrock_kms_key_arn
      }
    ]
  })
}
```

### 2C.4 Terraform Module Outputs

After deploying the Bedrock Terraform module, capture these outputs for application integration:

```hcl
output "bedrock_runtime_endpoint_id" {
  description = "VPC Endpoint ID for bedrock-runtime"
  value       = aws_vpc_endpoint.bedrock_runtime.id
}

output "bedrock_invocation_log_group" {
  description = "CloudWatch log group for Bedrock invocations"
  value       = aws_cloudwatch_log_group.bedrock.name
}

output "bedrock_logs_bucket" {
  description = "S3 bucket for detailed invocation logs"
  value       = var.bedrock_logs_bucket_name
}

output "bedrock_iam_policy_arn" {
  description = "ARN of IAM policy for Bedrock invocation"
  value       = aws_iam_policy.bedrock_invoke.arn
}
```

Verify deployment:

```bash
# Test VPC endpoint is reachable from ECS task
aws ec2 describe-vpc-endpoints \
  --filter "Name=vpc-id,Values=vpc-XXXX" \
  --filter "Name=service-name,Values=com.amazonaws.us-east-1.bedrock-runtime"

# Test invocation with a sample call (no PHI)
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-sonnet-4-6-20250514 \
  --content-type application/json \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":100,"messages":[{"role":"user","content":"Say hello."}]}' \
  /dev/stdout | jq .
```

---

## 2D. Model Access Request (Manual Step)

Model access must be requested before invocation. This is a one-time manual step per account per model (cannot be automated via Terraform as of Feb 2026).

```
This step was covered in Step 2 above (AWS Console Model Access). Listed here for clarity in the Terraform deployment sequence.
```

---

## 3. AI Scribe Pipeline: End-to-End Architecture

This is the core clinical workflow that transforms a patient encounter into structured documentation, billing codes, and a signed FHIR record.

### Pipeline Flow

```
Stage 1: Audio Capture
  Provider records encounter on tablet/phone
      |
      v
Stage 2: Speech-to-Text (Whisper v3)
  Audio file -> Amazon SageMaker endpoint (GPU)
  OR self-hosted on ECS with GPU task (g5.xlarge)
      |
      v (raw transcript)
Stage 3: Clinical Structuring (Claude via Bedrock)
  Raw transcript -> Claude Sonnet
  Prompt includes:
    - Patient context (from FHIR Patient resource)
    - Visit reason (from FHIR Appointment)
    - Medication list (from FHIR MedicationStatement)
    - Problem list (from FHIR Condition)
  Output: Structured SOAP note JSON
      |
      v
Stage 4: FHIR Document Creation
  SOAP JSON -> FHIR DocumentReference resource
  Stored in tenant schema (per ADR-002)
      |
      v (structured note available)
Stage 5: AI Coding Suggestions (Claude via Bedrock)
  SOAP note -> Claude Haiku (faster, cheaper)
  Output: Suggested ICD-10 + CPT codes with confidence scores
      |
      v
Stage 6: Human Review
  Provider reviews in web UI:
    - Read/edit SOAP note
    - Accept/reject/modify code suggestions
    - Digitally sign the note
      |
      v
Stage 7: Finalization
  Signed note:
    - FHIR DocumentReference.status = "current"
    - FHIR DocumentReference.authenticator = provider reference
    - ICD-10/CPT codes stored as FHIR Claim resource
    - Audit trail records the signing event
```

### Stage Details

#### Stage 2: Speech-to-Text (Whisper v3)

**Option A: SageMaker Endpoint (Recommended for Phase 1)**

- Deploy Whisper v3 large on SageMaker real-time endpoint
- Instance: `ml.g5.xlarge` (1x A10G GPU, 24 GB VRAM)
- Inference time: ~10-15 seconds for a 15-minute encounter
- SageMaker is HIPAA-eligible under AWS BAA
- Cost: ~$1.00/hour for the endpoint (can use async inference to avoid always-on cost)

**Option B: ECS with GPU (Future optimization)**

- ECS tasks on `g5.xlarge` EC2 instances via ECS capacity provider
- More control, potentially lower cost at scale
- Requires managing GPU instance lifecycle

**Option C: Third-Party API (NOT recommended)**

- Services like AssemblyAI or Deepgram offer medical-specific models
- Requires separate BAA, data leaves AWS, additional compliance burden
- Only consider if Whisper accuracy proves insufficient for medical terminology

**Decision:** Start with SageMaker async inference for Whisper v3. Transition to real-time endpoint when encounter volume justifies always-on cost.

#### Stage 3: Clinical Note Generation (Claude Sonnet via Bedrock)

**System Prompt Design:**

The system prompt is the most critical piece of the AI Scribe. It must produce consistent, clinically accurate SOAP notes that meet documentation requirements.

```
Key prompt engineering decisions:
1. Use structured output (JSON mode) to guarantee parseable responses
2. Include the full patient context in the prompt (medications, problems, allergies)
   to improve clinical accuracy and reduce hallucination
3. Instruct Claude to flag uncertainty: "[VERIFY: ...]" markers for provider review
4. Include specialty-specific templates (primary care vs. cardiology vs. orthopedics)
5. Output schema matches FHIR DocumentReference.content structure
```

**Output Structure:**

```json
{
  "soap_note": {
    "subjective": {
      "chief_complaint": "string",
      "history_present_illness": "string",
      "review_of_systems": {
        "constitutional": "string",
        "cardiovascular": "string",
        ...
      },
      "past_medical_history": "string (from FHIR context)",
      "medications": "string (from FHIR context)",
      "allergies": "string (from FHIR context)",
      "social_history": "string",
      "family_history": "string"
    },
    "objective": {
      "vitals": { ... },
      "physical_exam": { ... }
    },
    "assessment": {
      "diagnoses": [
        {
          "description": "string",
          "icd10_suggestion": "string",
          "confidence": 0.95,
          "reasoning": "string"
        }
      ]
    },
    "plan": {
      "items": [
        {
          "description": "string",
          "category": "medication|referral|lab|imaging|followup|education",
          "details": "string"
        }
      ]
    }
  },
  "verification_flags": [
    {
      "section": "assessment",
      "field": "diagnoses[0].icd10_suggestion",
      "message": "Verify: Patient mentioned chest pain but vitals are normal. Consider R07.9 vs I20.0.",
      "severity": "review_required"
    }
  ],
  "metadata": {
    "model": "anthropic.claude-sonnet-4-6-20250514",
    "processing_time_ms": 3200,
    "token_usage": { "input": 2400, "output": 1800 },
    "encounter_duration_seconds": 900
  }
}
```

#### Stage 5: AI Coding Suggestions (Claude Haiku via Bedrock)

**Why Haiku for Coding:**

- ICD-10/CPT code suggestion is a classification task -- Haiku is sufficient and 10x cheaper than Sonnet
- The SOAP note (from Stage 3) provides structured input -- no need for advanced reasoning
- Response time matters for provider UX -- Haiku responds in <1 second vs. 3-5 seconds for Sonnet
- Human review is ALWAYS required -- AI suggestions are never auto-submitted

**Output Structure:**

```json
{
  "coding_suggestions": {
    "icd10": [
      {
        "code": "J06.9",
        "description": "Acute upper respiratory infection, unspecified",
        "confidence": 0.92,
        "supporting_evidence": "Patient reports sore throat, nasal congestion x3 days. Exam shows erythematous pharynx.",
        "specificity_note": null
      },
      {
        "code": "R50.9",
        "description": "Fever, unspecified",
        "confidence": 0.78,
        "supporting_evidence": "Temperature 101.2F recorded in vitals.",
        "specificity_note": "Consider R50.81 if fever persists >3 days."
      }
    ],
    "cpt": [
      {
        "code": "99213",
        "description": "Office visit, established patient, low complexity",
        "confidence": 0.85,
        "mdm_level": "Low",
        "time_based_alternative": "99214 if total time exceeds 30 minutes",
        "supporting_elements": {
          "problems": 2,
          "data_reviewed": "Minimal",
          "risk": "Low"
        }
      }
    ],
    "modifiers": [],
    "bundling_warnings": []
  }
}
```

---

## 4. Integration with LangGraph Agent Framework

MedOS uses LangGraph for orchestrating the multi-step AI pipeline. Bedrock is the model provider, but the orchestration logic lives in the application layer.

### Architecture

```
Application Layer (FastAPI + LangGraph)
  |
  |-- LangGraph StateGraph
  |     |-- Node: "transcribe" -> calls SageMaker (Whisper)
  |     |-- Node: "generate_note" -> calls Bedrock (Claude Sonnet)
  |     |-- Node: "suggest_codes" -> calls Bedrock (Claude Haiku)
  |     |-- Node: "validate_fhir" -> validates output against FHIR R4 schema
  |     |-- Node: "store_draft" -> writes to PostgreSQL (tenant schema)
  |     |-- Edge: conditional routing based on verification flags
  |
  |-- Bedrock Client (boto3)
  |     |-- Uses IAM task role (no API keys)
  |     |-- VPC endpoint (private connectivity)
  |     |-- Retry logic with exponential backoff
  |
  |-- Observability
        |-- Invocation logs -> CloudWatch + S3 (Bedrock native)
        |-- Application traces -> OpenTelemetry -> CloudWatch
        |-- LangGraph state -> PostgreSQL (for debugging/replay)
```

### Bedrock Client Configuration

```python
# Application code pattern (not Terraform -- included for completeness)
import boto3
from botocore.config import Config

# Client is configured via IAM task role automatically in ECS
bedrock_runtime = boto3.client(
    "bedrock-runtime",
    region_name="us-east-1",
    config=Config(
        retries={"max_attempts": 3, "mode": "adaptive"},
        connect_timeout=5,
        read_timeout=60,
    ),
)

# Invoke Claude Sonnet for clinical note generation
response = bedrock_runtime.invoke_model(
    modelId="anthropic.claude-sonnet-4-6-20250514",
    contentType="application/json",
    accept="application/json",
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 4096,
        "system": CLINICAL_NOTE_SYSTEM_PROMPT,
        "messages": [
            {
                "role": "user",
                "content": f"Generate a SOAP note from this transcript:\n\n{transcript}\n\nPatient context:\n{patient_context}"
            }
        ],
        "temperature": 0.1,  # Low temperature for clinical accuracy
    }),
)
```

### Key Design Decisions for LangGraph Integration

1. **State persistence:** LangGraph state checkpoints stored in PostgreSQL (tenant schema) for auditability and replay capability during debugging.
2. **Error handling:** If Claude returns an unparseable response, retry once with a repair prompt. If still invalid, flag for manual transcription.
3. **Streaming:** Use `InvokeModelWithResponseStream` for the note generation step so providers see the note being generated in real-time in the review UI.
4. **Timeout:** 60-second timeout on Bedrock invocations. Encounters longer than 30 minutes may need to be split into segments.
5. **Fallback:** If Bedrock is unavailable (rare), queue the encounter for async processing and notify the provider.

---

## 5. Cost Estimates for Bedrock Usage

### Per-Encounter Cost Breakdown (Typical 15-Minute Office Visit)

| Stage | Model | Input Tokens | Output Tokens | Input Cost | Output Cost | Total |
|---|---|---|---|---|---|---|
| Transcription | Whisper v3 (SageMaker async) | -- | -- | -- | -- | $0.02 |
| Note generation | Claude Sonnet 4.6 | ~3,000 | ~2,000 | $0.009 | $0.030 | $0.039 |
| Code suggestion | Claude Haiku 4.5 | ~2,000 | ~500 | $0.002 | $0.003 | $0.005 |
| **Total per encounter** | | | | | | **$0.066** |

### Pricing Reference (as of Feb 2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Use Case |
|---|---|---|---|
| Claude Sonnet 4.6 | $3.00 | $15.00 | Complex reasoning, SOAP generation |
| Claude Haiku 4.5 | $0.80 | $4.00 | Classification, coding suggestions |
| Whisper v3 (SageMaker) | -- | -- | $0.02/encounter (async pricing) |

### Phase 1 Development Cost Projection

**Scenario:** Internal testing + small pilot (10-20 encounters/day)

```
Daily volume: 15 encounters/day
Monthly: 300 encounters (assuming 20 working days)

Claude Sonnet:   100 calls × 5,000 tokens × ($3+$15)/1M = $0.90
Claude Haiku:    200 calls × 2,500 tokens × ($0.80+$4)/1M = $1.20
Whisper v3:      300 calls × $0.02 = $6.00
─────────────────────────────────────────────────────
Total/month Phase 1: ~$31/month
```

### Pilot Phase Cost Projection

**Scenario:** 100 encounters/day across 3 clinical sites

```
Daily volume: 100 encounters/day
Monthly: 2,000 encounters (assuming 20 working days)

Routing: 30% Sonnet (complex), 70% Haiku (routine)
  Sonnet:  600 calls × 5,000 tokens × ($3+$15)/1M = $5.40
  Haiku:  1,400 calls × 2,500 tokens × ($0.80+$4)/1M = $13.44
Whisper v3: 2,000 calls × $0.02 = $40.00
─────────────────────────────────────────────────────
Total/month Pilot: ~$500-700/month

Cost/encounter: ~$0.27
```

### Scale Phase Cost Projection

**Scenario:** 1,000+ encounters/day nationally

```
Daily volume: 1,000 encounters/day
Monthly: 20,000 encounters (assuming 20 working days)

Routing optimized: 20% Sonnet, 80% Haiku
  Sonnet:  4,000 calls × 5,000 tokens × ($3+$15)/1M = $36.00
  Haiku: 16,000 calls × 2,500 tokens × ($0.80+$4)/1M = $77.44
Whisper v3: 20,000 calls × $0.02 = $400.00
─────────────────────────────────────────────────────
Total/month Scale: ~$5,000-7,000/month

Cost/encounter: ~$0.26
```

### Cost Optimization Strategies

1. **Model Routing by Task Complexity**
   - Complex (PDLs, prior auths): Sonnet
   - Routine (straightforward cases): Haiku
   - Reduces token spend by ~40% vs. all-Sonnet

2. **Prompt Engineering**
   - Compress patient context (send FHIR refs, not full records)
   - Use structured output mode (JSON schema)
   - Target: reduce input tokens from 3,000 to 2,000
   - Savings: ~15% on note generation

3. **Async Inference for Whisper**
   - Phase 1: Use SageMaker async inference ($0.02/call vs. $1/hour always-on)
   - Acceptable delay: 30-60 seconds (providers review notes later)
   - Savings: $50-200/month depending on volume

4. **Bedrock Batch API (Future)**
   - AWS Bedrock Batch API (launch expected Q1 2026): 50% discount for async jobs
   - Use for: retrospective coding review, bulk chart summarization
   - Estimated savings: 20-30% of total Bedrock spend

5. **Token Budget Enforcement**
   - Per-microservice daily token quotas
   - Alert at 80% of daily limit
   - Prevents runaway costs from prompt injection or bugs

**Target Cost/Encounter (Phase 1):** $0.066 (actual: $0.031 with optimization)

### SageMaker Whisper Cost Optimization

| Strategy | Cost | Latency | Use When |
|---|---|---|---|
| Real-time endpoint (always-on) | ~$730/month | <15s | >100 encounters/day |
| Async inference (scale-to-zero) | ~$0.02/encounter | 30-60s | <100 encounters/day |
| Batch transform (nightly) | ~$0.01/encounter | Hours | Retrospective transcription |

**Recommendation:** Start with async inference. The 30-60 second delay is acceptable because providers typically move on to the next patient and review notes later. Transition to real-time when daily volume consistently exceeds 100 encounters.

---

## 6. HIPAA Compliance for AI

### What the BAA Covers

AWS's BAA covers Bedrock model invocations. This means:

1. **Data at rest:** Prompts and completions logged via Bedrock invocation logging are encrypted with KMS and covered under BAA.
2. **Data in transit:** All traffic between your VPC and Bedrock stays within AWS via VPC endpoint. TLS 1.2+ enforced.
3. **Data processing:** Anthropic has confirmed that input data sent to Claude via Bedrock is NOT used for model training. Data is processed in-memory and discarded after response generation.
4. **Breach notification:** AWS will notify you if a breach affecting Bedrock infrastructure is detected, per BAA terms.

### What the BAA Does NOT Cover

1. **Model accuracy:** The BAA does not guarantee clinical accuracy of AI outputs. All AI-generated content must be reviewed by a licensed provider before becoming part of the medical record.
2. **De-identification:** If you want to minimize PHI exposure to the model, you must implement de-identification in your application layer before sending to Bedrock. The BAA covers PHI in prompts, but defense-in-depth suggests minimizing it where possible.
3. **Prompt injection attacks:** The BAA does not cover scenarios where a malicious prompt causes the model to produce harmful output. Implement input validation in the application layer.

### Application-Level Safeguards

```
1. PHI Minimization (defense-in-depth, not required by BAA):
   - Send patient context by reference (FHIR resource IDs) when possible
   - For clinical note generation, full encounter context IS needed --
     rely on BAA coverage
   - NEVER log raw prompts in application logs (use Bedrock invocation
     logging instead -- it's encrypted and access-controlled)

2. Output Validation:
   - Parse Claude's JSON output against expected schema
   - Reject responses that contain unexpected fields
   - Flag responses where confidence scores are below threshold (0.70)
   - NEVER auto-apply AI suggestions to the medical record

3. Human-in-the-Loop (MANDATORY):
   - Every AI-generated note MUST be reviewed by a licensed provider
   - Provider digitally signs the note (FHIR DocumentReference.authenticator)
   - Unsigned notes have status "preliminary" -- they are drafts, not records
   - Audit trail captures: AI generated -> provider reviewed -> provider signed

4. Audit Trail for AI Interactions:
   - Every Bedrock invocation logged with:
     - Timestamp
     - User/provider who initiated the request
     - Patient encounter ID (not patient name/DOB)
     - Model used
     - Token count
     - Whether output was accepted, modified, or rejected by the provider
   - Stored as FHIR AuditEvent resources in the tenant schema
```

---

## 7. Security Considerations

### Threat Model for AI Pipeline

| Threat | Mitigation |
|---|---|
| Prompt injection via transcript | Input sanitization, output schema validation, human review |
| Data exfiltration via model output | Structured output only (JSON), schema validation rejects free-form responses |
| Unauthorized model access | IAM policy scoped to specific model IDs, VPC endpoint restricts network path |
| Invocation log tampering | KMS encryption, S3 object lock, CloudTrail tracks all S3/KMS operations |
| Model hallucination (incorrect diagnosis) | Mandatory provider review, confidence scores, verification flags, disclaimer in UI |
| Excessive token usage (cost attack) | Input token limit (8K per request), rate limiting per user, budget alerts |
| PHI in error messages | Application catches all errors before logging, strips any model output from error context |

### Network Security

```
ECS Task (private subnet)
    |
    | (port 443, TLS 1.2+)
    v
VPC Endpoint (bedrock-runtime)
    |
    | (AWS internal network, never public internet)
    v
AWS Bedrock Service (us-east-1)
    |
    | (ephemeral processing, no data retention)
    v
Response returned to ECS task
```

No public internet path exists between the application and Bedrock. The VPC endpoint ensures all traffic stays within AWS's internal network.

---

## 8. Monitoring and Observability

### CloudWatch Metrics to Track

| Metric | Source | Alarm Threshold |
|---|---|---|
| Bedrock invocation count | CloudWatch (Bedrock namespace) | >5,000/day (unexpected spike) |
| Bedrock invocation latency (p99) | CloudWatch | >30 seconds |
| Bedrock throttling errors | CloudWatch | >10/hour |
| Bedrock validation errors | Application metrics | >5% of invocations |
| Claude output rejection rate | Application metrics | >20% of generated notes |
| Token usage per invocation | Application metrics | >8,000 input tokens (prompt too large) |
| Cost per day | AWS Cost Explorer | >$50/day (dev), >$200/day (prod) |

### Dashboard Widgets

```
Bedrock Operations Dashboard:
  - Invocation count (line chart, hourly)
  - Latency distribution (p50, p90, p99)
  - Error rate by type (throttle, validation, timeout)
  - Token usage trend (input vs. output)
  - Cost accumulation (daily, running total vs. budget)
  - Note acceptance rate (accepted, modified, rejected)
  - Model comparison (Sonnet vs. Haiku usage split)
```

---

## 9. Deployment Checklist

### Pre-Deployment (Before Any PHI)

- [ ] AWS BAA signed in AWS Artifact (click-through agreement)
- [ ] Bedrock model access approved for Claude Sonnet and Haiku
- [ ] VPC endpoint for `bedrock-runtime` deployed and tested
- [ ] IAM policy scoped to only approved models
- [ ] Invocation logging configured (CloudWatch + S3 with KMS)
- [ ] Security group allows only ECS tasks to reach Bedrock endpoint
- [ ] Test invocation successful with synthetic data (not PHI)
- [ ] Invocation log verified: prompt and completion captured, encrypted
- [ ] Cost alerts configured for Bedrock spending

### Application Integration

- [ ] Bedrock client initialized with IAM role (no API keys)
- [ ] Retry logic implemented (exponential backoff, max 3 retries)
- [ ] Output schema validation implemented (reject malformed responses)
- [ ] Human review UI built and functional
- [ ] Provider digital signature workflow implemented
- [ ] FHIR AuditEvent created for every AI interaction
- [ ] Verification flags displayed prominently in review UI
- [ ] Confidence scores visible to provider
- [ ] "AI-generated" label on all AI-produced content

### Post-Deployment Validation

- [ ] End-to-end test: audio -> transcript -> SOAP note -> codes -> review -> sign
- [ ] Cross-tenant isolation verified (Tenant A's encounter cannot invoke Bedrock with Tenant B's context)
- [ ] Rate limiting tested (burst of 100 requests does not cause failures)
- [ ] Failover tested (Bedrock unavailable -> encounters queued, provider notified)
- [ ] Invocation logs reviewed by compliance officer
- [ ] Cost per encounter matches projections (within 20%)

---

## 10. Future Considerations (Phase 2+)

### Additional AI Capabilities

| Capability | Model | Phase | Description |
|---|---|---|---|
| Prior authorization letter generation | Claude Sonnet | Phase 2 | Generate appeal letters from denial reasons + clinical evidence |
| Patient communication (AVS) | Claude Haiku | Phase 2 | Generate patient-friendly visit summaries |
| Clinical decision support | Claude Sonnet | Phase 3 | Flag potential drug interactions, missing screenings |
| Population health insights | Claude Sonnet | Phase 3 | Analyze aggregated (de-identified) data for trends |
| Real-time ambient listening | Whisper + Claude | Phase 3 | Live transcription during encounter (requires different architecture) |

### Bedrock Agents (Phase 2+)

In Phase 1, we use direct `InvokeModel` calls orchestrated by LangGraph. In Phase 2, consider migrating to Bedrock Agents for:

- **Knowledge bases:** Connect Claude to a vector store of medical guidelines (UpToDate, clinical pathways) via Bedrock Knowledge Bases
- **Action groups:** Define allowed actions (FHIR read, FHIR write, eligibility check) that Claude can invoke autonomously within guardrails
- **Guardrails for Bedrock:** Use Bedrock Guardrails to enforce content policies (no off-topic responses, no harmful medical advice)

Terraform resources for Bedrock Agents:

```hcl
# Phase 2 -- not deployed in Phase 1
resource "aws_bedrockagent_agent" "clinical_assistant" {
  agent_name         = "medos-clinical-assistant"
  foundation_model   = "anthropic.claude-sonnet-4-6-20250514"
  instruction        = file("${path.module}/prompts/clinical-assistant.txt")
  idle_session_ttl   = 600  # 10 minutes
}

resource "aws_bedrockagent_knowledge_base" "medical_guidelines" {
  name        = "medos-medical-guidelines"
  role_arn    = aws_iam_role.kb_role.arn
  description = "Medical guidelines and clinical pathways for decision support"

  knowledge_base_configuration {
    type = "VECTOR"
    vector_knowledge_base_configuration {
      embedding_model_arn = "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0"
    }
  }

  storage_configuration {
    type = "OPENSEARCH_SERVERLESS"
    opensearch_serverless_configuration {
      collection_arn    = aws_opensearchserverless_collection.kb.arn
      vector_index_name = "medical-guidelines"
      field_mapping {
        vector_field   = "embedding"
        text_field     = "text"
        metadata_field = "metadata"
      }
    }
  }
}
```

---

## Related Documents

- [[terraform-module-plan]] -- Complete Terraform module specifications including the `bedrock` module
- [[EPIC-004-ai-clinical-documentation]] -- Epic with tasks for building the AI Scribe pipeline
- [[AWS-HIPAA-Infrastructure]] -- HIPAA-eligible services list confirming Bedrock coverage
- [[ADR-001-fhir-native-data-model]] -- FHIR DocumentReference structure for storing AI-generated notes
- [[ADR-002-multi-tenancy-schema-per-tenant]] -- Tenant isolation for AI interaction audit logs
- [[PHASE-1-EXECUTION-PLAN]] -- Sprint 2 tasks for AI clinical documentation
- [[HIPAA-Deep-Dive]] -- Regulatory requirements for AI in healthcare
- [[HEALTHCARE_OS_MASTERPLAN]] -- Product vision for AI-native healthcare OS
