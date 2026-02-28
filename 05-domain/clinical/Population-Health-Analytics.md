---
type: domain-knowledge
date: "2026-02-27"
tags:
  - domain
  - clinical
  - population-health
  - analytics
  - module-e
category: clinical
confidence: high
sources: []
---

# Population Health Analytics

> This document covers population health management strategies, risk adjustment, quality measurement, care gap identification, and analytics architecture for [[HEALTHCARE_OS_MASTERPLAN]]. Population health analytics (Module E) is the intelligence layer that transforms clinical and claims data into actionable insights for providers, practice administrators, and payer contract negotiations.

---

## 1. Population Health Overview

Population health management (PHM) is the systematic approach to managing the health outcomes of a defined population. Unlike traditional fee-for-service medicine, which focuses on individual encounters, PHM considers the entire patient panel and seeks to optimize outcomes across the population while managing total cost of care.

**Why it matters for the Healthcare OS:**

- **Value-based care is the future**: CMS has committed to having all Medicare beneficiaries in accountable care relationships by 2030. Practices without population health capabilities will be unable to participate in the most favorable payment models.
- **Revenue opportunity**: Risk adjustment accuracy (HCC coding), quality measure performance (HEDIS/MIPS/Stars), and care gap closure directly drive revenue. A practice that improves its RAF score by 5% on a 1,000-member Medicare Advantage panel can generate $500K-$1M in additional annual revenue.
- **Competitive moat**: Network-wide population health analytics create a data advantage that individual practices cannot replicate. Aggregate benchmarking, referral optimization, and payer behavior analysis are only possible with multi-practice data. This is a core thesis of the Healthcare OS.
- **Clinical outcomes**: Proactive identification of high-risk patients, care gap closure, and SDOH intervention genuinely improve health outcomes. This is not just a financial exercise.

Population health analytics connects to [[Revenue-Cycle-Deep-Dive]] (financial impact), [[FHIR-R4-Deep-Dive]] (data model), [[Clinical-Workflows-Overview]] (provider workflows), and [[Patient-Engagement-Patterns]] (patient outreach).

---

## 2. HCC Risk Adjustment (CMS-HCC v28)

### 2.1 What It Is and Why It Matters

The CMS Hierarchical Condition Category (HCC) model is used to adjust Medicare Advantage (MA) capitation payments based on the expected healthcare costs of each beneficiary. Plans and providers that document conditions more completely receive higher capitation payments that reflect the true cost of caring for their population.

This is not about upcoding. It is about accurate and complete documentation of conditions that are actually present. CMS estimates that 10-15% of diagnoses present in a given year are not documented, leading to underpayment and inadequate resource allocation.

### 2.2 How Risk Scores Work

Each Medicare Advantage beneficiary receives a Risk Adjustment Factor (RAF) score calculated from two components:

1. **Demographic factors**: Age, sex, dual-eligible status (Medicare + Medicaid), disability status, institutional status. These establish a baseline RAF.
2. **Diagnosis factors**: ICD-10 codes documented during face-to-face encounters in the measurement year are mapped to HCC categories. Each HCC has a coefficient that adds to the RAF score.

**Example RAF calculation:**
- 72-year-old female, community, non-dual: baseline 0.35
- HCC 19 (Diabetes with complications): +0.302
- HCC 85 (Congestive Heart Failure): +0.323
- HCC 111 (Chronic Obstructive Pulmonary Disease): +0.335
- **Total RAF: 1.310**

The RAF score is multiplied by the county base rate to determine capitation payment. A RAF of 1.310 means this patient is expected to cost 31% more than the average beneficiary.

### 2.3 HCC Categories and Hierarchies

CMS-HCC v28 (effective for payment year 2025+) includes approximately 115 HCC categories organized into hierarchies. The "hierarchical" aspect means that when a patient has related conditions of varying severity, only the most severe category counts.

**Example hierarchy:**
- HCC 17: Diabetes with Acute Complications (highest severity)
- HCC 18: Diabetes with Chronic Complications
- HCC 19: Diabetes without Complication (lowest severity)

If a patient has both HCC 17 and HCC 19, only HCC 17 counts. The hierarchy prevents double-counting within related condition groups.

**Major HCC groups with high coefficients:**
- Cancer (HCC 8-12): coefficients 0.15-1.02
- Diabetes (HCC 17-19): coefficients 0.10-0.30
- Vascular disease (HCC 84-86): coefficients 0.15-0.32
- Heart failure (HCC 85): coefficient ~0.32
- Chronic kidney disease (HCC 134-139): coefficients 0.10-0.44
- COPD (HCC 111): coefficient ~0.34

### 2.4 Revenue Impact

The financial impact of accurate HCC coding is substantial:
- **1% increase in average RAF** across a Medicare Advantage panel = approximately $100-$200 per member per year in additional capitation
- **For a 5,000-member panel**: 1% RAF improvement = $500K-$1M additional annual revenue
- **Industry benchmarks**: Well-managed risk adjustment programs capture 3-8% more RAF than average, translating to millions in annual revenue for large panels

### 2.5 Coding Accuracy Requirements

- All HCC-relevant diagnoses must be documented in a **face-to-face encounter** during the measurement year
- Diagnoses must be **recaptured annually** -- a diagnosis coded in 2025 does not carry forward to 2026 automatically
- Documentation must support the diagnosis with specificity (ICD-10 code level)
- Conditions must be **actively managed** or **monitored** -- historical conditions that are resolved should not be coded
- CMS conducts **RADV (Risk Adjustment Data Validation) audits** that can result in payment recovery if documentation does not support coded diagnoses

### 2.6 Suspect Conditions

Suspect conditions are diagnoses that are likely present based on clinical evidence but have not been formally documented. Identifying suspects is a key function of the population health analytics module.

**Sources of suspect identification:**
- **Claims history**: Conditions coded in prior years but not yet recaptured in the current year
- **Medication evidence**: Patient on metformin but no diabetes diagnosis documented
- **Lab results**: A1c of 7.8% but no diabetes HCC captured
- **Clinical notes**: NLP extraction of conditions mentioned but not coded
- **Predictive models**: ML models that identify patients likely to have undocumented conditions based on demographic and clinical patterns

### 2.7 Annual Wellness Visit as Capture Opportunity

The Medicare Annual Wellness Visit (AWV) is the single best opportunity for HCC recapture:
- Face-to-face encounter that satisfies CMS documentation requirements
- Comprehensive review of all active conditions
- Opportunity to address, document, and code every relevant HCC
- Billable as a preventive service (no cost to patient under Medicare)
- AWV completion rate is a key performance metric for population health programs

**AWV optimization**: The analytics module should generate a pre-visit HCC gap report for each AWV, listing suspect conditions and missing recaptures for the provider to address during the visit.

---

## 3. Quality Measures

### 3.1 HEDIS (Healthcare Effectiveness Data and Information Set)

HEDIS is maintained by NCQA and is the most widely used quality measurement set in U.S. healthcare. Medicare Advantage plans, Medicaid managed care plans, and commercial plans use HEDIS measures to evaluate provider performance.

**Key HEDIS measures relevant to primary care:**
- **CDC (Comprehensive Diabetes Care)**: A1c testing, A1c control (<8%), eye exam, kidney screening, blood pressure control
- **BCS (Breast Cancer Screening)**: Mammography for women 50-74
- **COL (Colorectal Cancer Screening)**: Screening for adults 45-75
- **CBP (Controlling High Blood Pressure)**: BP < 140/90
- **CIS (Childhood Immunization Status)**: Combo 10 by age 2
- **AWC (Adolescent Well-Care Visits)**: Annual visit for ages 12-21
- **FUH (Follow-Up After Hospitalization for Mental Illness)**: Within 7 and 30 days

### 3.2 MIPS (Merit-based Incentive Payment System)

MIPS is the CMS quality payment program for traditional Medicare. It adjusts provider payments based on performance across four categories:

1. **Quality (30% of score)**: 6 measures selected from a menu, including one outcome measure. Reported via claims, QCDR, or EHR.
2. **Promoting Interoperability (25% of score)**: EHR usage metrics including e-prescribing, health information exchange, patient portal access, and security risk analysis.
3. **Improvement Activities (15% of score)**: Participation in qualifying activities like care coordination, patient engagement, and population health management.
4. **Cost (30% of score)**: Calculated by CMS from claims data. Total per capita cost, Medicare Spending Per Beneficiary, and episode-based measures.

**Payment adjustments**: MIPS scores range from 0-100. Scores above 75 earn positive adjustments (up to +9%). Scores below the performance threshold receive negative adjustments (up to -9%). This is a zero-sum budget neutrality program.

### 3.3 Star Ratings (Medicare Advantage)

The CMS Star Rating system rates Medicare Advantage plans on a 1-5 star scale. Stars drive revenue through quality bonus payments (QBP):

- **5 stars**: 5% QBP + open enrollment year-round
- **4+ stars**: 5% QBP
- **3.5 stars**: 3.5% QBP (estimated under current benchmarks)
- **Below 3 stars**: No QBP, potential CMS sanctions, enrollment restrictions

**Star Ratings composition** (approximately 45 measures across domains):
- Staying healthy (screenings, tests, vaccines)
- Managing chronic conditions
- Member experience (CAHPS surveys)
- Member complaints and access
- Health plan customer service

**Financial impact**: For a plan with 50,000 members, the difference between 3.5 and 4.5 stars can be $30M-$50M in annual QBP. Providers who contribute to Star improvement are rewarded with better contract terms.

### 3.4 Measure Types

- **Process measures**: Did the recommended action occur? (e.g., Was A1c tested?)
- **Outcome measures**: Was the desired result achieved? (e.g., Is A1c < 8%?)
- **Patient experience measures**: How did the patient perceive the care? (CAHPS)
- **Cost/efficiency measures**: Were resources used appropriately? (Total cost of care)

The industry is shifting from process to outcome measures. The analytics module must track both but prioritize outcome improvement.

---

## 4. Care Gap Identification

### 4.1 What Is a Care Gap

A care gap is a clinically recommended service that has not been delivered to an eligible patient within the appropriate time frame. Care gaps represent both a clinical risk (the patient is not receiving recommended care) and a financial risk (the practice is not meeting quality benchmarks).

### 4.2 Common Care Gaps

| Care Gap | Eligible Population | Recommended Interval | HEDIS Measure |
|----------|-------------------|---------------------|---------------|
| A1c Testing | Diabetics 18-75 | Every 6-12 months | CDC |
| Mammography | Women 50-74 | Every 2 years | BCS |
| Colonoscopy | Adults 45-75 | Every 10 years (or alternatives) | COL |
| Diabetic Eye Exam | Diabetics 18-75 | Annual | CDC |
| Diabetic Kidney Screening | Diabetics 18-75 | Annual | CDC |
| Flu Vaccine | All patients 6mo+ | Annual | FVO |
| Depression Screening | Adults 12+ | Annual | DSF |
| BP Control | Hypertensives 18-85 | Every visit | CBP |
| Statin Therapy | Diabetics 40-75 | Ongoing | SPD |
| Cervical Cancer Screening | Women 21-65 | Every 3 years (Pap) / 5 years (HPV) | CCS |

### 4.3 Automated Identification from FHIR Data

The analytics engine identifies care gaps by querying FHIR resources:

1. **Identify eligible population**: Query Patient resources for demographics, Condition resources for qualifying diagnoses
2. **Check for completed services**: Query Procedure, Observation, DiagnosticReport, and Immunization resources for evidence of completed care
3. **Apply measure logic**: Compare eligible population against completed services using CQL (Clinical Quality Language) or custom rules
4. **Generate gap list**: Patients with open gaps, prioritized by clinical urgency and financial impact

See [[FHIR-R4-Deep-Dive]] for resource mapping details.

### 4.4 Patient Outreach for Gap Closure

Once gaps are identified, the system drives closure through multi-channel outreach:
- **Patient portal notification**: "You are due for your annual mammogram"
- **SMS campaign**: Bulk messaging to patients with open gaps, with scheduling link
- **Provider pre-visit alert**: When a patient with open gaps has an upcoming appointment, alert the provider to address during the visit
- **Automated scheduling**: For gaps that require an appointment, offer self-scheduling

See [[Patient-Engagement-Patterns]] for outreach channel details.

### 4.5 Revenue Impact of Closing Care Gaps

- **MIPS quality score improvement**: Each additional measure met improves the composite score, potentially moving from penalty to bonus territory
- **Star Rating improvement**: Gap closure directly impacts Star measures, driving quality bonus payments
- **Value-based contract performance**: Many shared savings contracts include quality gates. Failure to meet quality thresholds can reduce or eliminate shared savings payments.
- **Billable services**: Many gap closure activities are independently billable (AWV, screenings, labs). Closing a gap generates both quality points and fee-for-service revenue.

---

## 5. Readmission Prediction

### 5.1 Hospital Readmissions Reduction Program (HRRP)

CMS penalizes hospitals with excess readmission rates for six conditions:
- Acute myocardial infarction (AMI)
- Heart failure (HF)
- Pneumonia
- COPD
- Total hip/knee arthroplasty (THA/TKA)
- Coronary artery bypass graft (CABG)

**Penalty structure**: Up to 3% reduction in all Medicare inpatient payments (not just readmission-related payments). In FY2024, approximately 2,200 hospitals received penalties.

While HRRP directly penalizes hospitals, primary care practices bear responsibility for post-discharge care and can differentiate themselves by reducing readmissions for their patients.

### 5.2 LACE+ Index

LACE is a validated readmission risk scoring tool:
- **L**ength of stay (longer stay = higher risk)
- **A**cuity of admission (emergent vs. elective)
- **C**omorbidities (Charlson Comorbidity Index)
- **E**D visits (emergency department utilization in prior 6 months)

LACE+ adds additional factors including lab values and demographics. Scores range from 0-100, with higher scores indicating greater readmission risk.

### 5.3 Social Determinants of Health as Predictive Factors

SDOH factors significantly improve readmission prediction beyond clinical variables alone:
- **Housing instability**: Homeless patients have 2-3x higher readmission rates
- **Food insecurity**: Nutritional deficiency impairs recovery
- **Transportation barriers**: Inability to attend follow-up appointments
- **Social isolation**: No caregiver support at home
- **Health literacy**: Inability to understand discharge instructions
- **Financial hardship**: Cannot afford medications or follow-up care

### 5.4 ML Model Features and Approach

**Feature categories for readmission prediction model:**

1. **Clinical**: Primary diagnosis, comorbidity count, number of medications, lab values at discharge (albumin, creatinine, BNP), vital signs
2. **Utilization**: Prior hospitalizations (6/12 months), ED visits, number of active providers, polypharmacy
3. **Procedural**: Surgical vs. medical admission, ICU stay, length of stay
4. **Social**: Age, insurance type, zip code deprivation index, living situation, SDOH screening results
5. **Discharge**: Discharge disposition (home, SNF, rehab), follow-up scheduled within 7 days, medication reconciliation completed

**Model approach**: Gradient boosted trees (XGBoost/LightGBM) with SHAP explanations for feature importance. Logistic regression as interpretable baseline. Validate on held-out test set with stratification by facility and diagnosis.

### 5.5 Intervention Strategies for High-Risk Patients

- **Transitional Care Management (TCM)**: Post-discharge phone call within 2 business days, face-to-face visit within 7 or 14 days. Billable under CPT 99495/99496.
- **Medication reconciliation**: Pharmacist-led review of discharge medications vs. pre-admission medications
- **Home health referral**: For patients unable to manage self-care
- **Social work referral**: For patients with identified SDOH barriers
- **Remote patient monitoring**: Continuous vital sign monitoring post-discharge. See [[Patient-Engagement-Patterns]] for RPM details.
- **Patient education**: Condition-specific discharge education with teach-back confirmation

---

## 6. Social Determinants of Health (SDOH)

### 6.1 The Five SDOH Domains

The WHO and Healthy People 2030 define five domains:

1. **Economic Stability**: Employment, income, food security, housing stability, medical cost burden
2. **Education Access and Quality**: Literacy, language, educational attainment, vocational training
3. **Social and Community Context**: Social support, civic participation, incarceration history, discrimination
4. **Healthcare Access and Quality**: Insurance coverage, provider availability, health literacy, transportation to care
5. **Neighborhood and Built Environment**: Housing quality, transportation, air/water quality, access to healthy food, crime/violence

### 6.2 Z-Codes in ICD-10 for SDOH Documentation

ICD-10 Z-codes (Z55-Z65) capture SDOH in structured format:
- **Z55**: Problems related to education and literacy
- **Z56**: Problems related to employment and unemployment
- **Z57**: Occupational exposure to risk factors
- **Z59**: Problems related to housing and economic circumstances (Z59.0 = homelessness, Z59.1 = inadequate housing)
- **Z60**: Problems related to social environment
- **Z62**: Problems related to upbringing
- **Z63**: Problems related to primary support group (family)
- **Z65**: Problems related to psychosocial circumstances

**Documentation importance**: Z-codes do not affect HCC risk scores, but they are increasingly used for quality reporting, care coordination, and population health stratification. CMS is moving toward incorporating SDOH into risk adjustment models.

### 6.3 FHIR Resources for SDOH (Gravity Project)

The Gravity Project (HL7) has standardized FHIR profiles for SDOH data exchange:
- **Screening**: QuestionnaireResponse (SDOH screening tool results)
- **Assessment**: Observation (individual SDOH findings, e.g., food insecurity identified)
- **Diagnosis**: Condition (SDOH conditions coded with Z-codes)
- **Goals**: Goal (patient goals related to SDOH, e.g., obtain stable housing)
- **Interventions**: ServiceRequest (referrals to community services), Task (tracking referral completion)

See [[FHIR-R4-Deep-Dive]] for detailed resource specifications.

### 6.4 Screening Tools

- **AHC-HRSN (Accountable Health Communities Health-Related Social Needs)**: CMS-developed, 10-item core screen covering housing, food, transportation, utilities, and safety
- **PRAPARE (Protocol for Responding to and Assessing Patients' Assets, Risks, and Experiences)**: 21-item tool covering demographics, family/home, money/resources, social/emotional health
- **Screening frequency**: At minimum during AWV, new patient visits, and hospital discharge. Ideally at every encounter via digital intake.

---

## 7. Analytics Dashboard Design

### 7.1 Provider Scorecards

Individual provider performance dashboards showing:
- Panel size and demographics
- Quality measure performance vs. benchmarks (HEDIS/MIPS)
- Care gap closure rate (open gaps / eligible patients)
- HCC recapture rate (recaptured / expected recaptures)
- Coding specificity metrics (e.g., unspecified diabetes vs. type 2 with complications)
- Patient satisfaction scores (NPS, survey results)
- Revenue per patient (FFS + value-based)
- Referral patterns and specialist utilization

### 7.2 Practice-Level Benchmarking

- Compare providers within the same practice on standardized metrics
- Identify top performers and replicate their workflows
- Surface outliers for coaching and support
- Normalize for panel complexity (risk-adjusted metrics)

### 7.3 Payer-Specific Performance

- Performance against each payer's quality program requirements
- Projected MIPS score and payment adjustment
- Star Rating contribution analysis
- Shared savings contract performance tracking
- Denial rate by payer, CPT code, and denial reason (connects to [[Revenue-Cycle-Deep-Dive]])

### 7.4 Trending and Forecasting

- Time-series trends for all key metrics (monthly, quarterly, annual)
- Forecasting models for quality scores, RAF scores, and financial performance
- Early warning indicators when trends deviate from targets
- Seasonal patterns (flu season impact on utilization, Q4 push for gap closure)

### 7.5 Cohort Comparison

- Define custom patient cohorts (e.g., diabetics with A1c > 9%, patients with 3+ ED visits)
- Compare outcomes, utilization, and cost across cohorts
- Identify intervention opportunities for high-cost/high-risk cohorts
- Track cohort performance over time after interventions

---

## 8. Network-Wide Intelligence (Our Data Moat)

This is the most strategically important section. Individual practices can buy point solutions for quality reporting and risk adjustment. What they cannot do alone is generate network-wide intelligence. This is the core value proposition of the Healthcare OS as a multi-practice platform.

### 8.1 Cross-Practice Benchmarking

- Compare any metric across all practices in the network
- Identify best-in-class practices for specific measures and facilitate knowledge sharing
- Risk-adjusted comparison (normalize for panel demographics and acuity)
- Benchmarking drives healthy competition and continuous improvement

### 8.2 Referral Pattern Optimization

- Map referral flows between primary care and specialists across the network
- Identify specialists with the best outcomes, shortest wait times, and lowest cost
- Preferred referral networks that keep patients within the ecosystem
- Referral leakage analysis: what percentage of referrals go outside the network?
- Closed-loop referral tracking: was the referral completed? What was the outcome?

### 8.3 Payer Behavior Analysis (Denial Patterns)

This is a unique insight that only a multi-practice network can generate:
- **Denial patterns by payer + procedure**: Which payers deny which CPT codes most frequently? What is the most effective appeal strategy?
- **Authorization requirements**: Which payers require prior auth for which procedures? How long does approval take?
- **Payment velocity**: How quickly does each payer pay? What is the average days in A/R by payer?
- **Contract term comparison**: Anonymous benchmarking of reimbursement rates across practices (same payer, same CPT, different contracted rates)

This intelligence is enormously valuable for contract negotiations. A practice entering renegotiation with a payer can see how their rates compare to network averages. See [[Revenue-Cycle-Deep-Dive]] for revenue cycle integration.

### 8.4 Regional Health Trends

- Aggregate de-identified data across the network to identify emerging health trends
- Disease prevalence tracking by geography
- Seasonal illness patterns
- Chronic disease management effectiveness by region
- Social determinant mapping by zip code
- This data has potential value for public health agencies, researchers, and payers (with appropriate de-identification and consent)

---

## 9. Value-Based Care Contracts

### 9.1 Shared Savings Arrangements

The most common VBC model. The practice shares in savings achieved below a benchmark total cost of care (TCOC) target.

**Structure:**
- Benchmark: Historical TCOC trended forward (e.g., average of prior 3 years + trend)
- Savings: If actual TCOC < benchmark, practice receives a percentage (typically 30-50%)
- Quality gate: Savings are only paid if quality measures meet minimum thresholds
- Risk corridor: Many arrangements are "upside only" (share savings but no downside risk for losses). More advanced arrangements include downside risk with higher savings share.

### 9.2 Capitation Models

Full or partial capitation shifts financial risk to the provider:

- **Full capitation**: Practice receives a fixed PMPM (per member per month) payment to cover all services. Practice bears full risk.
- **Partial/professional capitation**: PMPM covers professional services only (office visits, procedures). Facility and ancillary services remain FFS.
- **Sub-capitation**: Primary care group receives PMPM and sub-contracts with specialists
- **Global budget**: Similar to capitation but set at the practice level rather than per-member

### 9.3 Performance Guarantees

Payers increasingly require performance guarantees in VBC contracts:
- Minimum quality scores on specific measures
- Access standards (appointment availability within X days)
- SDOH screening completion rates
- Patient satisfaction minimums
- Penalties for failure to meet guarantees (reduced savings share or withholds)

### 9.4 Financial Reconciliation

The analytics module must support VBC financial reconciliation:
- Track attributed patient panel per contract
- Calculate TCOC from claims and encounter data
- Project savings/losses in real-time (not waiting for annual reconciliation)
- Model scenario analyses (what-if we close X more care gaps?)
- Reconcile payer settlements against internal calculations
- Audit trail for dispute resolution

---

## 10. AI/ML Models for Population Health

### 10.1 Models to Build

| Model | Type | Input | Output | Business Value |
|-------|------|-------|--------|----------------|
| No-show prediction | Classification | Patient + appointment features | Show/no-show probability | Optimize scheduling, reduce waste |
| Readmission risk | Classification | Clinical + social features | 30-day readmission probability | Target TCM interventions |
| HCC suspect identification | Classification | Claims + clinical data | Probability of undocumented HCC | Revenue capture |
| Care gap prioritization | Ranking | Patient + gap features | Prioritized outreach list | Maximize gap closure efficiency |
| Cost prediction | Regression | Demographics + clinical history | 12-month expected cost | VBC contract modeling |
| ED utilization | Classification | Patient features | High ED utilizer probability | Proactive intervention |
| Medication adherence | Classification | Rx claims + demographics | Non-adherence probability | Targeted outreach |
| Patient churn | Classification | Encounter + satisfaction data | Probability of leaving practice | Retention strategies |

### 10.2 Training Data Requirements

- **Minimum sample size**: 10,000+ patients for most classification models, 50,000+ for cost prediction
- **Historical depth**: 24-36 months of claims and clinical data minimum
- **Label quality**: Outcomes must be verified (actual readmission, confirmed no-show, validated HCC)
- **Feature completeness**: Handle missing data explicitly (imputation, missingness indicators)
- **Bias auditing**: Test for disparate performance across race, age, sex, insurance type, and geography. Healthcare ML models have well-documented bias risks.

### 10.3 Validation Approaches

- **Temporal validation**: Train on historical data, validate on future data (no random split, which leaks temporal patterns)
- **Geographic validation**: Train on some practices, validate on held-out practices
- **Stratified evaluation**: Report metrics (AUC, precision, recall, calibration) by subgroup
- **Clinical validation**: Provider review of model predictions for face validity
- **Prospective monitoring**: Track model performance in production; retrain when performance degrades (concept drift)

### 10.4 Deployment Considerations

- **Explainability**: SHAP values or similar for every prediction. Clinicians will not trust black-box models.
- **Integration**: Models must surface predictions in clinical workflows (EHR alerts, pre-visit reports), not in separate dashboards.
- **Regulatory**: FDA guidance on Clinical Decision Support (CDS) -- ensure models qualify for CDS exemption criteria (intended for clinician use, clinician does not have to follow, transparent logic).
- **Feedback loop**: Capture whether the clinician acted on the prediction and what the outcome was. Use this for continuous model improvement.
- **Refresh cadence**: Retrain monthly for high-volume models, quarterly for others. Monitor calibration drift continuously.

---

## Cross-References

- [[HEALTHCARE_OS_MASTERPLAN]] - System architecture and module dependencies
- [[Revenue-Cycle-Deep-Dive]] - Financial impact of risk adjustment and quality performance
- [[FHIR-R4-Deep-Dive]] - Data model for clinical and claims data
- [[Clinical-Workflows-Overview]] - Provider workflow integration for analytics insights
- [[Patient-Engagement-Patterns]] - Outreach channels for care gap closure and patient communication
- [[MOC-Domain-Knowledge]] - Map of all domain knowledge notes
