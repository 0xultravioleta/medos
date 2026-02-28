---
type: domain-knowledge
date: "2026-02-27"
tags:
  - domain
  - standards
  - x12
  - edi
  - module-c
  - module-d
category: standards
confidence: high
sources: []
---

# X12 EDI Deep Dive

> If FHIR is the future of healthcare data exchange, X12 EDI is the present -- and the past 30 years. Every claim submitted, every eligibility check, every prior authorization, every remittance advice in the U.S. healthcare system flows through X12 EDI transactions. As a developer building MedOS, you need to understand EDI because it is the backbone of revenue cycle operations, and it will coexist with FHIR for at least another decade. This document explains EDI from scratch, shows real transaction examples with annotations, and maps the path from X12 to FHIR.

---

## 1. What is X12 EDI

EDI stands for **Electronic Data Interchange** -- a standardized format for exchanging business documents between computer systems. X12 is the specific EDI standard developed by the Accredited Standards Committee X12 (ASC X12), an ANSI-accredited body.

Think of X12 EDI as a **very old, very strict, pipe-delimited text format** for healthcare transactions. If JSON is a flexible, human-readable format and XML is a verbose but structured format, X12 EDI is a **positionally-significant, delimiter-separated, cryptically-encoded** format that prioritizes compactness over readability.

Here is what a tiny piece of EDI looks like:

```
ISA*00*          *00*          *ZZ*SENDER         *ZZ*RECEIVER       *260227*1430*^*00501*000000001*0*P*:~
```

That single line (the ISA segment) contains 16 fields separated by `*` (element separator), terminated by `~` (segment terminator), with `:` as the component separator and `^` as the repetition separator. Every character position matters. The spaces in fields like `SENDER         ` are **required padding** to fill fixed-width fields.

### Key Concepts for Developers

| Concept | Definition | Analogy |
|---------|-----------|---------|
| **Transaction Set** | A complete business document (claim, eligibility check, etc.) | Like a JSON schema/endpoint |
| **Segment** | A single line of data, identified by a 2-3 character code | Like a JSON object |
| **Element** | A field within a segment, separated by `*` | Like a JSON property |
| **Component** | A sub-field within an element, separated by `:` | Like a nested JSON property |
| **Loop** | A repeating group of segments | Like a JSON array of objects |
| **Envelope** | ISA/IEA (interchange), GS/GE (functional group), ST/SE (transaction set) | Like HTTP headers wrapping a body |

---

## 2. Why EDI Still Exists

The Health Insurance Portability and Accountability Act (HIPAA) of 1996 mandated specific X12 transaction sets for healthcare administrative transactions. The relevant HIPAA rule is the **Transaction and Code Sets Rule (45 CFR Part 162)**, which requires all covered entities (providers, payers, clearinghouses) to use these specific X12 versions:

| Transaction | X12 Set | HIPAA Version | Purpose |
|-------------|---------|---------------|---------|
| Eligibility Inquiry | 270 | 005010X279A1 | "Is this patient covered?" |
| Eligibility Response | 271 | 005010X279A1 | "Yes, here's their coverage" |
| Claim Submission (Professional) | 837P | 005010X222A1 | Bill for physician services |
| Claim Submission (Institutional) | 837I | 005010X223A3 | Bill for hospital services |
| Claim Status Inquiry | 276 | 005010X212 | "What's happening with my claim?" |
| Claim Status Response | 277 | 005010X212 | "Here's the status" |
| Remittance Advice | 835 | 005010X221A1 | "Here's what we paid and why" |
| Prior Auth Request/Response | 278 | 005010X217 | Request and respond to PA |
| Enrollment | 834 | 005010X220A1 | Add/remove members from plans |
| Premium Payment | 820 | 005010X218 | Premium billing |

Because HIPAA **mandates** these specific formats, every payer, every provider system, and every clearinghouse in the U.S. must support them. FHIR is emerging as a complement (and eventual successor for some transactions), but X12 EDI will remain the legal requirement for claims and remittances for the foreseeable future.

The current mandated version is **005010** (adopted 2012). There is no mandated successor version yet, though the industry is gradually adopting FHIR for newer use cases like prior authorization (see [[Prior-Authorization-Deep-Dive]]).

---

## 3. Transaction Sets We Need

### 270/271 -- Eligibility Inquiry and Response

The 270 asks: "Is patient John Smith covered under plan XYZ, and what are his benefits for service category ABC?"

The 271 responds with: coverage status, copay amounts, deductible status, coinsurance, coverage dates, and benefit limits.

**Key 270 Segments:**

| Segment | Purpose | Example |
|---------|---------|---------|
| `BHT` | Beginning of hierarchical transaction | Transaction purpose, reference ID, date |
| `HL` | Hierarchical level | Information source -> receiver -> subscriber -> dependent |
| `NM1` | Individual or organizational name | Patient name, payer name, provider name |
| `DTP` | Date/time reference | Service date, eligibility date |
| `EQ` | Eligibility inquiry | Service type code (e.g., 30 = health benefit plan coverage) |
| `REF` | Reference identification | Member ID, group number |

**Key 271 Response Segments:**

| Segment | Purpose | Example |
|---------|---------|---------|
| `EB` | Eligibility/benefit information | Coverage level, service type, amount |
| `MSG` | Message text | Free-text benefit descriptions |
| `DTP` | Date ranges | Benefit period, plan dates |
| `III` | Additional inquiry info | Procedure codes, place of service |

### 276/277 -- Claim Status Inquiry and Response

The 276 asks: "What is the status of claim #12345 for patient Jane Doe?"

The 277 responds with: claim status category (accepted, rejected, pending), status codes, and adjudication details.

**Key Segments:**

| Segment | Purpose |
|---------|---------|
| `BHT` | Transaction set purpose and reference |
| `HL` | Hierarchical levels (source, provider, subscriber, claim) |
| `TRN` | Trace number (your tracking reference) |
| `STC` | Status information (category code, status code, entity code, effective date) |
| `REF` | Payer claim control number, clearinghouse trace number |
| `DTP` | Service dates, status date |
| `AMT` | Claim amounts (charged, paid) |

### 278 -- Prior Authorization Request and Response

The 278 handles the electronic submission of prior authorization requests and the payer's response. This is the X12 equivalent of the Da Vinci PAS FHIR workflow described in [[Prior-Authorization-Deep-Dive]].

**Key Segments:**

| Segment | Purpose |
|---------|---------|
| `BHT` | Transaction purpose (request vs. response) |
| `HL` | Utilization management org, requester, subscriber, dependent |
| `UM` | Health care services review information (request category, certification type, service type) |
| `HCR` | Health care services review (action code: certified, denied, pended, modified) |
| `SV1/SV2` | Professional/institutional service line (CPT/HCPCS codes) |
| `DTP` | Proposed service dates, certification effective dates |
| `HI` | Diagnosis codes (ICD-10) |
| `REF` | Prior auth number, referral number |

### 837P -- Professional Claims

The 837P is the workhorse of outpatient billing. It represents a CMS-1500 claim form in electronic format. This is what physician practices submit for office visits, procedures, and professional services.

**Key Segment Groups:**

| Loop | Segments | Purpose |
|------|----------|---------|
| Header | `BHT`, `REF` | Transaction metadata |
| 1000A | `NM1`, `PER` | Submitter information |
| 1000B | `NM1` | Receiver information |
| 2000A | `HL`, `PRV` | Billing provider hierarchical level |
| 2010AA | `NM1`, `N3`, `N4`, `REF` | Billing provider name and address |
| 2000B | `HL`, `SBR` | Subscriber hierarchical level |
| 2010BA | `NM1`, `DMG` | Subscriber name, demographics |
| 2010BB | `NM1` | Payer name |
| 2300 | `CLM`, `DTP`, `HI`, `REF` | Claim information (total charge, dates, diagnosis codes) |
| 2400 | `LX`, `SV1`, `DTP` | Service lines (CPT codes, charges, dates, modifiers) |

### 837I -- Institutional Claims

The 837I represents a UB-04 claim form for hospital/facility billing. It includes additional segments for admission/discharge information, revenue codes, and facility details.

### 835 -- Electronic Remittance Advice

The 835 is the payer's explanation of payment. It tells the provider exactly what was paid, what was denied, and why -- for every claim and every line item.

**Key Segments:**

| Segment | Purpose |
|---------|---------|
| `BPR` | Financial information (payment amount, payment method, bank routing) |
| `TRN` | Check/EFT trace number |
| `CLP` | Claim-level payment information (claim ID, status, charged, paid, patient responsibility) |
| `SVC` | Service-level payment (CPT code, charged, paid) |
| `CAS` | Claim adjustment segments (adjustment reason codes, amounts) |
| `PLB` | Provider-level balance (recoupments, interest, adjustments) |

The 835 is critical for the [[Revenue-Cycle-Deep-Dive]] because it drives automated payment posting and denial management.

---

## 4. EDI Anatomy -- Envelopes, Segments, Elements

### The Envelope Structure

Every X12 transaction is wrapped in three layers of envelopes:

```
ISA ... ~                          <- Interchange Header (outer envelope)
  GS ... ~                        <- Functional Group Header (middle envelope)
    ST ... ~                      <- Transaction Set Header (inner envelope)
      [actual transaction data]
    SE ... ~                      <- Transaction Set Trailer
  GE ... ~                        <- Functional Group Trailer
IEA ... ~                         <- Interchange Trailer
```

- **ISA/IEA** (Interchange): Identifies sender and receiver, contains security info and control numbers. One interchange can contain multiple functional groups.
- **GS/GE** (Functional Group): Groups transactions of the same type (e.g., all 270s together). Contains version identifier.
- **ST/SE** (Transaction Set): Wraps a single transaction. SE contains a segment count for validation.

### Real 270 Transaction Example (Annotated)

```
ISA*00*          *00*          *ZZ*MEDOSPRACTICE  *ZZ*AETNAEOB      *260227*1430*^*00501*000000123*0*P*:~
GS*HS*MEDOS*AETNA*20260227*1430*123*X*005010X279A1~
ST*270*0001*005010X279A1~
BHT*0022*13*REQ20260227001*20260227*1430~
HL*1**20*1~
NM1*PR*2*AETNA*****PI*12345~
HL*2*1*21*1~
NM1*1P*1*SMITH*ROBERT****XX*1234567890~
REF*TJ*123456789~
HL*3*2*22*0~
TRN*1*TRACE20260227001*9MEDOS~
NM1*IL*1*DOE*JANE****MI*XYZ123456789~
DMG*D8*19850315*F~
DTP*291*D8*20260227~
EQ*30~
SE*14*0001~
GE*1*123~
IEA*1*000000123~
```

**Line-by-line breakdown:**

| Line | Segment | Meaning |
|------|---------|---------|
| 1 | `ISA` | Interchange header: sender=MEDOSPRACTICE, receiver=AETNAEOB, date=2026-02-27, time=14:30, version=00501, control#=000000123, Production mode |
| 2 | `GS` | Functional group: type=HS (eligibility), sender=MEDOS, receiver=AETNA, version=005010X279A1 |
| 3 | `ST` | Transaction set 270 begins, control#=0001 |
| 4 | `BHT` | Beginning: purpose=request(13), reference=REQ20260227001, date/time |
| 5 | `HL*1**20*1` | Hierarchy level 1: Information Source (payer), has children |
| 6 | `NM1*PR*2*AETNA*****PI*12345` | Payer name: AETNA, entity type=organization(2), ID type=PI(Payor ID), ID=12345 |
| 7 | `HL*2*1*21*1` | Hierarchy level 2: Information Receiver (provider), parent=1, has children |
| 8 | `NM1*1P*1*SMITH*ROBERT****XX*1234567890` | Provider: Dr. Robert Smith, NPI=1234567890 |
| 9 | `REF*TJ*123456789` | Provider tax ID |
| 10 | `HL*3*2*22*0` | Hierarchy level 3: Subscriber (patient), parent=2, no children |
| 11 | `TRN*1*TRACE20260227001*9MEDOS` | Trace number for tracking this inquiry |
| 12 | `NM1*IL*1*DOE*JANE****MI*XYZ123456789` | Patient: Jane Doe, Member ID=XYZ123456789 |
| 13 | `DMG*D8*19850315*F` | Demographics: DOB=1985-03-15, Female |
| 14 | `DTP*291*D8*20260227` | Service date: 2026-02-27 |
| 15 | `EQ*30` | Eligibility inquiry: service type 30 (health benefit plan coverage) |
| 16 | `SE*14*0001` | Transaction set trailer: 14 segments, control#=0001 |
| 17 | `GE*1*123` | Functional group trailer: 1 transaction set |
| 18 | `IEA*1*000000123` | Interchange trailer: 1 functional group |

### Real 837P Transaction Example (Annotated)

```
ISA*00*          *00*          *ZZ*MEDOSPRACTICE  *ZZ*CLRHOUSE      *260227*1500*^*00501*000000456*0*P*:~
GS*HC*MEDOS*CLRHOUSE*20260227*1500*456*X*005010X222A1~
ST*837*0001*005010X222A1~
BHT*0019*00*CLAIM20260227001*20260227*1500*CH~
NM1*41*2*MEDOS FAMILY PRACTICE*****46*123456789~
PER*IC*BILLING DEPT*TE*5551234567~
NM1*40*2*CLEARINGHOUSE INC*****46*987654321~
HL*1**20*1~
NM1*85*2*MEDOS FAMILY PRACTICE*****XX*1234567890~
N3*123 MAIN STREET~
N4*ANYTOWN*TX*75001~
REF*EI*123456789~
HL*2*1*22*0~
SBR*P*18*GROUP123******CI~
NM1*IL*1*DOE*JANE****MI*XYZ123456789~
N3*456 OAK AVENUE~
N4*ANYTOWN*TX*75002~
DMG*D8*19850315*F~
NM1*PR*2*AETNA*****PI*12345~
N3*PO BOX 98765~
N4*HARTFORD*CT*06101~
CLM*CLAIM001*250***22:B:1*Y*A*Y*Y~
HI*ABK:M79.3~
HI*ABF:Z79.899~
LX*1~
SV1*HC:99213:25*150*UN*1***1~
DTP*472*D8*20260227~
LX*2~
SV1*HC:20610*100*UN*1***1~
DTP*472*D8*20260227~
SE*27*0001~
GE*1*456~
IEA*1*000000456~
```

**Key segments explained:**

| Segment | Meaning |
|---------|---------|
| `BHT*0019*00*...*CH` | Claim transaction, original claim (00), claim type=chargeable (CH) |
| `NM1*85*...*XX*1234567890` | Billing provider with NPI |
| `SBR*P*18*GROUP123******CI` | Subscriber: Primary payer, self relationship(18), group#=GROUP123, commercial insurance(CI) |
| `CLM*CLAIM001*250***22:B:1*Y*A*Y*Y` | Claim ID=CLAIM001, total charge=$250, place of service=22 (outpatient hospital), frequency=B(resubmission), type=1(original) |
| `HI*ABK:M79.3` | Principal diagnosis: ICD-10 M79.3 (Panniculitis, unspecified) |
| `HI*ABF:Z79.899` | Additional diagnosis: ICD-10 Z79.899 |
| `SV1*HC:99213:25*150*UN*1` | Service line 1: CPT 99213 (E/M visit) with modifier 25, charge=$150, unit=1 |
| `SV1*HC:20610*100*UN*1` | Service line 2: CPT 20610 (joint injection), charge=$100, unit=1 |
| `DTP*472*D8*20260227` | Service date: 2026-02-27 |

---

## 5. X12 to FHIR Mapping

Understanding how X12 transactions map to FHIR resources is essential for MedOS, which needs to operate in both worlds. See [[FHIR-R4-Deep-Dive]] for FHIR resource details.

| X12 Transaction | FHIR Resource(s) | Notes |
|----------------|-------------------|-------|
| 270 (Eligibility Inquiry) | CoverageEligibilityRequest | Patient, Coverage, Organization references |
| 271 (Eligibility Response) | CoverageEligibilityResponse | Benefits, coverage details, cost-to-beneficiary |
| 276 (Claim Status Inquiry) | Uses Task or custom operation | No direct FHIR equivalent; Da Vinci CDex may apply |
| 277 (Claim Status Response) | Uses Task or ExplanationOfBenefit | Status communicated via resource status fields |
| 278 (Prior Auth) | Claim (use=preauthorization) / ClaimResponse | Da Vinci PAS IG -- see [[Prior-Authorization-Deep-Dive]] |
| 837P (Professional Claim) | Claim (type=professional) | Claim, Patient, Practitioner, Organization, Coverage |
| 837I (Institutional Claim) | Claim (type=institutional) | Adds Encounter, Location |
| 835 (Remittance) | ExplanationOfBenefit (EOB) | ClaimResponse + PaymentReconciliation for payment details |

### Mapping Example: 837P CLM Segment to FHIR Claim

```
X12:  CLM*CLAIM001*250***22:B:1*Y*A*Y*Y
```

Maps to FHIR:

```json
{
  "resourceType": "Claim",
  "identifier": [{
    "value": "CLAIM001"
  }],
  "total": {
    "value": 250.00,
    "currency": "USD"
  },
  "item": [{
    "locationCodeableConcept": {
      "coding": [{
        "system": "https://www.cms.gov/Medicare/Coding/place-of-service-codes",
        "code": "22",
        "display": "Outpatient Hospital"
      }]
    }
  }],
  "billablePeriod": {
    "start": "2026-02-27"
  }
}
```

### Mapping Example: SV1 Segment to FHIR Claim.item

```
X12:  SV1*HC:99213:25*150*UN*1***1
```

Maps to FHIR:

```json
{
  "sequence": 1,
  "productOrService": {
    "coding": [{
      "system": "http://www.ama-assn.org/go/cpt",
      "code": "99213"
    }]
  },
  "modifier": [{
    "coding": [{
      "system": "http://www.ama-assn.org/go/cpt",
      "code": "25",
      "display": "Significant, Separately Identifiable E/M"
    }]
  }],
  "unitPrice": {
    "value": 150.00,
    "currency": "USD"
  },
  "quantity": {
    "value": 1
  }
}
```

---

## 6. Clearinghouse Architecture

A clearinghouse is a middleman that sits between providers and payers, handling the validation, translation, and routing of EDI transactions.

### What a Clearinghouse Does

```
[Provider EHR/PMS]
       |
       | (Raw 837P, 270, 278, etc.)
       v
[Clearinghouse]
  1. VALIDATION
     - Syntax: Are segments in correct order? Required fields present?
     - Business rules: Is the NPI valid? Is the payer ID correct?
     - SNIP levels 1-7 (increasingly strict validation)
  2. TRANSLATION
     - Payer-specific formatting requirements
     - Code set mapping (if needed)
     - Companion guide compliance
  3. ROUTING
     - Map payer ID to correct endpoint
     - Handle connectivity (AS2, SFTP, API, direct connect)
     - Manage enrollment/trading partner agreements
  4. ACKNOWLEDGMENT
     - TA1 (Interchange Acknowledgment): Was the envelope valid?
     - 999 (Functional Acknowledgment): Was the transaction syntactically valid?
     - 277CA (Claim Acknowledgment): Was the claim accepted for adjudication?
       |
       v
[Payer Adjudication System]
       |
       | (835, 271, 277, 278 response)
       v
[Clearinghouse]
  - Route response back to provider
  - Translate if needed
       |
       v
[Provider EHR/PMS]
```

### Major Clearinghouses

| Clearinghouse | Parent Company | Market Share |
|---------------|---------------|-------------|
| Change Healthcare | Optum/UHG | ~40% |
| Availity | Joint venture (multiple payers) | ~25% |
| Trizetto | Cognizant | ~15% |
| Waystar | EQT Partners | ~10% |
| Office Ally | Independent | ~5% (small practices) |

### SNIP Validation Levels

| Level | What It Checks |
|-------|---------------|
| Type 1 | EDI syntax integrity (segments, delimiters, envelopes) |
| Type 2 | HIPAA implementation guide requirements |
| Type 3 | Balancing (claim totals match line item totals) |
| Type 4 | Inter-segment situational rules |
| Type 5 | External code set validation (ICD-10, CPT, NPI) |
| Type 6 | Payer-specific requirements |
| Type 7 | Real-time claim adjudication/estimation |

---

## 7. Building Our Own Clearinghouse

MedOS has the option to build clearinghouse functionality directly into the platform. This is a significant undertaking but offers major advantages.

### Technical Requirements

1. **HIPAA Compliance**: Must meet all HIPAA Administrative Simplification requirements
2. **X12 Parser/Generator**: Full 005010 support for all mandated transaction sets
3. **Connectivity**: AS2 (most common), SFTP, HTTPS/API connections to payers
4. **Payer Enrollment**: Trading partner agreements with each payer (hundreds)
5. **Validation Engine**: SNIP levels 1-5 minimum
6. **Companion Guide Library**: Payer-specific formatting rules
7. **Acknowledgment Processing**: TA1, 999, 277CA handling
8. **CAQH CORE Certification**: Industry certification for clearinghouse operations
9. **SOC 2 Type II**: Security certification required by payers

### Advantages of Building In-House

| Advantage | Impact |
|-----------|--------|
| No per-transaction fees | Save $0.25-$1.00 per transaction (at scale, this is millions) |
| Real-time processing | No batch delays from third-party clearinghouses |
| Data ownership | Full access to transaction data for analytics and AI |
| Tighter integration | Direct EHR-to-payer pipeline |
| Competitive moat | Significant barrier to entry for competitors |
| FHIR+EDI translation | Unified layer that handles both standards |

### Build vs Buy Decision

For early-stage MedOS, the pragmatic approach is:

1. **Phase 1**: Partner with existing clearinghouse (Availity or Change Healthcare API)
2. **Phase 2**: Build internal X12 parser/generator for analytics and translation
3. **Phase 3**: Establish direct payer connections for top 10 payers (covers ~80% of volume)
4. **Phase 4**: Full clearinghouse functionality with CAQH CORE certification

See [[HEALTHCARE_OS_MASTERPLAN]] for how this fits into the overall platform roadmap.

---

## 8. Common EDI Errors

### Rejection vs. Denial

- **Rejection**: Transaction failed validation (syntax, missing fields, bad codes). Never reached the payer's adjudication system. Fix and resubmit.
- **Denial**: Transaction was accepted and adjudicated but the payer refused to pay. Requires appeal or corrected claim.

### Common Rejection Codes (999/277CA)

| Code | Description | Common Cause | Fix |
|------|-------------|-------------|-----|
| AK905=5 | Implementation guide not supported | Wrong version in GS segment | Use correct 005010 version string |
| AK402=4 | Data element too long | Field exceeds max length | Truncate to spec length |
| AK402=7 | Invalid character | Non-ASCII characters in data | Sanitize input data |
| IK304=3 | Required segment missing | Missing NM1 or REF segment | Add required segment |
| A7:65 | Invalid subscriber ID | Member ID doesn't match payer records | Verify member ID with eligibility check first |
| A7:72 | Invalid diagnosis code | ICD-10 code not valid for service date | Check code set effective dates |
| A8:04 | Invalid NPI | Provider NPI not enrolled with payer | Complete payer enrollment |

### Common Claim Denial Codes (CARC/RARC from 835)

| CARC | Description | Action |
|------|-------------|--------|
| CO-4 | Procedure code inconsistent with modifier | Review modifier usage |
| CO-16 | Claim/service lacks information | Check for missing data elements |
| CO-18 | Duplicate claim | Check for prior submission |
| CO-29 | Filing limit exceeded | Timely filing issue -- may need to appeal |
| CO-50 | Not medically necessary | Appeal with clinical documentation |
| CO-97 | Payment adjusted (bundled) | Review CCI edits for bundling rules |
| PR-1 | Deductible amount | Patient responsibility -- bill patient |
| PR-2 | Coinsurance amount | Patient responsibility -- bill patient |
| PR-3 | Copay amount | Patient responsibility -- bill patient |
| OA-23 | Impact of prior payment | Coordination of benefits issue |

CARC = Claim Adjustment Reason Code. RARC = Remittance Advice Remark Code. CO = Contractual Obligation. PR = Patient Responsibility. OA = Other Adjustment.

---

## 9. Testing EDI

### Companion Guides

Every payer publishes a **companion guide** that specifies their payer-specific requirements on top of the HIPAA implementation guide. These documents detail:

- Required and situational segments/elements
- Payer-specific code values
- Submission methods and endpoints
- Enrollment requirements
- Testing procedures

Always read the companion guide before submitting to a new payer.

### Test Environments

| Environment | Purpose |
|-------------|---------|
| **Clearinghouse sandbox** | Test connectivity and basic validation (most clearinghouses offer this) |
| **Payer test endpoints** | Some payers (UHC, Aetna, Humana) provide test endpoints for EDI submission |
| **CMS testing** | CEDI (Common Electronic Data Interchange) for Medicare Part A/B testing |
| **Local validator** | Validate syntax and implementation guide compliance before submission |

### Testing Checklist

1. Validate against HIPAA implementation guide (SNIP levels 1-4)
2. Validate against payer companion guide (SNIP levels 5-6)
3. Submit test transaction to clearinghouse sandbox
4. Verify TA1 and 999 acknowledgments are clean
5. Submit to payer test endpoint (if available)
6. Verify 277CA acceptance
7. For 837: verify 835 response contains expected adjudication
8. For 270: verify 271 response contains expected benefit data
9. For 278: verify PA response matches expected decision

---

## 10. Libraries and Tools

### Python Libraries for X12 Parsing

#### `pyx12`

The most established Python library for X12 parsing and validation.

```python
# Install
# pip install pyx12

from pyx12.x12context import X12ContextReader
import io

edi_data = """ISA*00*          *00*          *ZZ*SENDER         *ZZ*RECEIVER       *260227*1430*^*00501*000000001*0*P*:~
GS*HS*MEDOS*AETNA*20260227*1430*1*X*005010X279A1~
ST*270*0001*005010X279A1~
BHT*0022*13*REQ001*20260227*1430~
HL*1**20*1~
NM1*PR*2*AETNA*****PI*12345~
HL*2*1*21*1~
NM1*1P*1*SMITH*ROBERT****XX*1234567890~
HL*3*2*22*0~
NM1*IL*1*DOE*JANE****MI*XYZ123456789~
EQ*30~
SE*11*0001~
GE*1*1~
IEA*1*000000001~"""

# Parse EDI
src = io.StringIO(edi_data)
reader = X12ContextReader(src)
for segment in reader.iter_segments():
    print(f"{segment.id}: {segment.elements}")
```

#### `tig` (Transaction Implementation Guide)

A lighter-weight parser for segment-level access:

```python
def parse_x12(edi_string: str) -> list[dict]:
    """Parse X12 EDI into a list of segment dictionaries."""
    segments = []
    # Detect delimiters from ISA segment
    element_sep = edi_string[3]  # Usually '*'
    segment_term = edi_string[105]  # Usually '~'

    raw_segments = edi_string.split(segment_term)
    for raw in raw_segments:
        raw = raw.strip()
        if not raw:
            continue
        elements = raw.split(element_sep)
        segment_id = elements[0]
        segments.append({
            "id": segment_id,
            "elements": elements[1:]
        })
    return segments

def extract_patient_from_270(segments: list[dict]) -> dict:
    """Extract patient information from a parsed 270."""
    patient = {}
    for seg in segments:
        if seg["id"] == "NM1" and seg["elements"][0] == "IL":
            patient["last_name"] = seg["elements"][2]
            patient["first_name"] = seg["elements"][3]
            patient["member_id"] = seg["elements"][8] if len(seg["elements"]) > 8 else None
        if seg["id"] == "DMG":
            patient["dob"] = seg["elements"][1]
            patient["gender"] = seg["elements"][2]
    return patient
```

#### Custom 837P Builder

```python
from datetime import datetime

class EDI837PBuilder:
    """Build an 837P (Professional Claim) transaction."""

    def __init__(self, sender_id: str, receiver_id: str):
        self.sender_id = sender_id.ljust(15)
        self.receiver_id = receiver_id.ljust(15)
        self.segments: list[str] = []
        self.control_number = datetime.now().strftime("%y%m%d%H%M%S")

    def build_isa(self) -> str:
        now = datetime.now()
        return (
            f"ISA*00*{' '*10}*00*{' '*10}"
            f"*ZZ*{self.sender_id}*ZZ*{self.receiver_id}"
            f"*{now.strftime('%y%m%d')}*{now.strftime('%H%M')}"
            f"*^*00501*{self.control_number}*0*P*:~"
        )

    def build_gs(self, group_control: str = "1") -> str:
        now = datetime.now()
        return (
            f"GS*HC*{self.sender_id.strip()}*{self.receiver_id.strip()}"
            f"*{now.strftime('%Y%m%d')}*{now.strftime('%H%M')}"
            f"*{group_control}*X*005010X222A1~"
        )

    def add_claim(
        self,
        patient_last: str,
        patient_first: str,
        member_id: str,
        dob: str,
        gender: str,
        payer_name: str,
        payer_id: str,
        provider_npi: str,
        provider_name: str,
        provider_tax_id: str,
        diagnosis_codes: list[str],
        service_lines: list[dict],
        claim_id: str = None
    ) -> None:
        """Add a claim to the transaction.

        service_lines: [{"cpt": "99213", "charge": 150.00,
                         "units": 1, "modifiers": ["25"],
                         "date": "20260227"}]
        """
        if claim_id is None:
            claim_id = f"CLM{datetime.now().strftime('%Y%m%d%H%M%S')}"

        total_charge = sum(line["charge"] for line in service_lines)

        # Claim segment
        self.segments.append(f"CLM*{claim_id}*{total_charge:.2f}***22:B:1*Y*A*Y*Y~")

        # Diagnosis codes
        for i, dx in enumerate(diagnosis_codes):
            qualifier = "ABK" if i == 0 else "ABF"
            self.segments.append(f"HI*{qualifier}:{dx}~")

        # Service lines
        for idx, line in enumerate(service_lines, 1):
            cpt = line["cpt"]
            modifiers = ":".join(line.get("modifiers", []))
            if modifiers:
                cpt = f"{cpt}:{modifiers}"
            self.segments.append(f"LX*{idx}~")
            self.segments.append(
                f"SV1*HC:{cpt}*{line['charge']:.2f}"
                f"*UN*{line['units']}***1~"
            )
            self.segments.append(f"DTP*472*D8*{line['date']}~")

    def generate(self) -> str:
        """Generate the complete 837P transaction."""
        output = []
        output.append(self.build_isa())
        output.append(self.build_gs())
        output.append(f"ST*837*0001*005010X222A1~")
        output.append(f"BHT*0019*00*{self.control_number}*"
                      f"{datetime.now().strftime('%Y%m%d')}*"
                      f"{datetime.now().strftime('%H%M')}*CH~")

        # Add all claim segments
        for seg in self.segments:
            output.append(seg)

        seg_count = len(output) - 1  # Exclude ISA, GS; include SE
        output.append(f"SE*{seg_count}*0001~")
        output.append(f"GE*1*1~")
        output.append(f"IEA*1*{self.control_number}~")

        return "\n".join(output)
```

### X12 to FHIR Conversion

```python
from typing import Any

def x12_270_to_fhir_eligibility_request(segments: list[dict]) -> dict[str, Any]:
    """Convert parsed X12 270 segments to a FHIR CoverageEligibilityRequest."""
    patient = extract_patient_from_270(segments)

    # Extract payer info
    payer = {}
    for seg in segments:
        if seg["id"] == "NM1" and seg["elements"][0] == "PR":
            payer["name"] = seg["elements"][2]
            payer["id"] = seg["elements"][8] if len(seg["elements"]) > 8 else None

    # Extract service date
    service_date = None
    for seg in segments:
        if seg["id"] == "DTP" and seg["elements"][0] == "291":
            raw = seg["elements"][2]
            service_date = f"{raw[:4]}-{raw[4:6]}-{raw[6:8]}"

    return {
        "resourceType": "CoverageEligibilityRequest",
        "status": "active",
        "purpose": ["benefits"],
        "patient": {
            "display": f"{patient.get('first_name', '')} {patient.get('last_name', '')}",
            "identifier": {
                "value": patient.get("member_id")
            }
        },
        "servicedDate": service_date,
        "insurer": {
            "display": payer.get("name"),
            "identifier": {
                "value": payer.get("id")
            }
        },
        "created": service_date
    }
```

### Useful Tools

| Tool | Purpose | URL/Package |
|------|---------|------------|
| `pyx12` | Python X12 parser with validation | `pip install pyx12` |
| `stedi` | Cloud-based EDI platform with APIs | stedi.com |
| `Edifact/X12 VS Code Extension` | Syntax highlighting for EDI files | VS Code marketplace |
| `x12-parser` (npm) | Node.js X12 parser | `npm install x12-parser` |
| `Claim.MD` | Online EDI testing/submission | claim.md |
| `WEDI` | EDI industry resources and guides | wedi.org |

---

## Key Connections

- [[Prior-Authorization-Deep-Dive]] -- X12 278 is the EDI transaction for prior auth
- [[Revenue-Cycle-Deep-Dive]] -- EDI transactions power the entire revenue cycle
- [[FHIR-R4-Deep-Dive]] -- FHIR is the emerging complement/successor to X12 for some transactions
- [[HEALTHCARE_OS_MASTERPLAN]] -- Module C (claims/billing) and Module D (AI/analytics) architecture
