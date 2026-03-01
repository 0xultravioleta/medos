---
type: training
date: "2026-02-28"
tags:
  - training
  - admin
  - onboarding
  - phase-1
---

# Administrator Workflow Guide -- MedOS

> Step-by-step guide for practice administrators managing the MedOS platform: practice onboarding, provider and location configuration, user management, payer contracts, and audit log review.

---

## Prerequisites

- MedOS account with Admin role assigned
- Multi-factor authentication (MFA) configured
- Practice information ready: NPI numbers, tax ID, provider credentials, payer contracts, office locations

---

## Workflow Steps

### Step 1: Onboard a New Practice

1. Log in and navigate to **Admin > Practice Setup**.
2. Enter practice details: legal name, NPI (Type 2 organizational), tax ID, primary address, phone, and specialty.
3. Configure practice settings: business hours, appointment slot duration, default encounter types.
4. MedOS creates an isolated tenant schema for the practice with its own encryption keys.
5. Review the confirmation screen showing the practice ID and tenant configuration.

[Screenshot: Practice Setup wizard with fields and confirmation]

**Expected result:** New practice tenant created with isolated database schema and dedicated KMS encryption key.

### Step 2: Configure Providers and Locations

1. Navigate to **Admin > Providers**.
2. Click **Add Provider** and enter: name, NPI (Type 1 individual), specialty, credentials (MD/DO/NP/PA), state license number.
3. Assign the provider to one or more office locations.
4. Set provider-specific settings: default encounter templates, scribe preferences, schedule visibility.
5. Navigate to **Admin > Locations** to manage office addresses, phone numbers, and operating hours.

[Screenshot: Provider configuration form with NPI and credential fields]

**Expected result:** Provider appears in the practice directory and can be assigned to the schedule.

### Step 3: Manage Users

1. Navigate to **Admin > Users**.
2. Click **Invite User** and enter their email address.
3. Assign a role: Provider, Nurse, Medical Assistant, Billing Staff, Front Desk, or Admin.
4. Set permissions scope: which locations, which providers' patients, which modules.
5. The user receives an email invitation to set up their account with MFA.
6. To deactivate a user, click their name and select **Deactivate**. Access is revoked immediately.

[Screenshot: User management list with role assignments and invite button]

**Expected result:** User receives invitation email within minutes. Role-based access is enforced immediately upon activation.

### Step 4: Set Up Payer Contracts

1. Navigate to **Admin > Payer Contracts**.
2. Click **Add Payer** and select from the payer directory or enter custom payer details.
3. Enter contract terms: fee schedule, allowed amounts by CPT code, timely filing limits, prior auth requirements.
4. Configure the clearinghouse payer ID for electronic claims submission.
5. Test connectivity with a sample eligibility check (X12 270/271).

[Screenshot: Payer contract configuration with fee schedule entry]

**Expected result:** Payer configured and ready for eligibility checks and claims submission.

### Step 5: Review Audit Logs

1. Navigate to **Admin > Audit Logs**.
2. The audit log shows all PHI access events across the practice: who accessed which patient's data, when, and from where.
3. Filter by date range, user, patient, action type (read, create, update, delete), or resource type.
4. Break-the-glass access events are highlighted and require review.
5. Export audit logs for compliance reporting (CSV or PDF).

[Screenshot: Audit log viewer with filters and highlighted break-the-glass events]

**Expected result:** Complete audit trail of all PHI access. Every access event includes user, action, resource, timestamp, and IP address.

---

## Common Issues and FAQ

**Q: A user cannot log in after being invited.**
A: Verify the invitation email was received (check spam). Re-send the invitation from the Users panel. Ensure MFA setup was completed.

**Q: A provider's NPI is not being accepted.**
A: Verify the NPI is active at https://npiregistry.cms.hhs.gov/. Ensure you are using the Type 1 (individual) NPI, not the Type 2 (organizational).

**Q: I need to change a user's role.**
A: Navigate to Admin > Users, click the user's name, and update their role. The change takes effect on their next login.

**Q: How often should I review audit logs?**
A: Review break-the-glass events weekly. Conduct a full access audit quarterly per the [[HIPAA-Risk-Assessment]] schedule.

---

## References

- [[Auth-SMART-on-FHIR]] -- Role-based access control and authentication
- [[HIPAA-Risk-Assessment]] -- Access review schedule and compliance requirements
- [[System-Architecture-Overview]] -- Multi-tenancy and tenant isolation architecture
