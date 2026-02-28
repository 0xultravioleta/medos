---
type: domain-knowledge
date: "2026-02-27"
tags:
  - engineering
  - security
  - auth
  - smart-on-fhir
  - oauth2
  - module-g
category: engineering
confidence: high
sources:
  - https://build.fhir.org/ig/HL7/smart-app-launch/scopes-and-launch-context.html
  - https://build.fhir.org/ig/HL7/smart-app-launch/app-launch.html
  - https://auth0.com/docs/secure/data-privacy-and-compliance
  - https://docs.aws.amazon.com/cognito/latest/developerguide/compliance-validation.html
  - https://github.com/Alvearie/keycloak-extensions-for-fhir
  - https://www.hipaavault.com/resources/2026-hipaa-changes/
  - https://build.fhir.org/auditevent.html
  - https://profiles.ihe.net/ITI/BALP/
---

# Authentication, Authorization & SMART on FHIR

This document covers the complete authentication and authorization strategy for MedOS, including auth provider selection, SMART on FHIR integration, role-based and attribute-based access control, multi-factor authentication, session management, break-the-glass emergency access, API key management, and audit trail implementation. See also [[System-Architecture-Overview]] and [[HIPAA-Deep-Dive]].

---

## 1. Auth Provider Comparison for Healthcare

Selecting an identity provider for a healthcare platform requires evaluating HIPAA compliance capabilities, SMART on FHIR compatibility, MFA support, and operational burden. Below is a detailed comparison of four leading options.

### Auth0 (Okta)

Auth0 provides a managed identity platform with extensive healthcare adoption. HIPAA BAA is available on Enterprise plans (custom pricing, typically starting around $240/month for 1,000 MAU when negotiated with BAA). Auth0 supports custom OAuth2 flows, making SMART on FHIR scope mapping feasible via Auth0 Actions (serverless hooks in the auth pipeline). Auth0 Actions allow injecting FHIR launch context into tokens, mapping custom scopes, and enforcing ABAC policies at the authorization layer.

Key strengths: mature ecosystem, extensive documentation, pre-built MFA flows (TOTP, SMS, WebAuthn, push notifications), machine-to-machine tokens for system-to-system integration, and Organizations feature for multi-tenancy. The primary drawback is cost at scale and vendor lock-in.

### Clerk

Clerk is a newer identity platform with superior developer experience (drop-in React/Next.js components, session management, user management UI). Clerk offers HIPAA BAA on their Enterprise plan. Clerk's architecture is component-driven, meaning auth UI elements (sign-in, sign-up, user profile) are pre-built and customizable. Clerk supports MFA (TOTP and SMS), webhook-driven event flows, and Organizations for multi-tenancy.

Key weakness for healthcare: SMART on FHIR scope support does not exist natively. Implementing FHIR-specific scopes would require a custom authorization layer sitting between Clerk and the FHIR server. Clerk is younger than Auth0, so the healthcare-specific ecosystem is less mature.

### Keycloak (Self-Hosted)

Keycloak is an open-source identity and access management solution (CNCF project). It is free to use, fully self-hosted, and provides complete control over the auth pipeline. Keycloak has active community extensions for SMART on FHIR, notably [Alvearie keycloak-extensions-for-fhir](https://github.com/Alvearie/keycloak-extensions-for-fhir) and [zedwerks/keycloak-smart-fhir](https://github.com/zedwerks/keycloak-smart-fhir), which add SMART launch context, FHIR scope parsing, and custom token mappers.

HIPAA BAA is not applicable since it is self-hosted; compliance is the operator's responsibility. Keycloak supports all MFA methods (TOTP, WebAuthn, OTP via SMS through custom SPIs), RBAC, fine-grained authorization with policy enforcement, and identity brokering. The operational burden is significant: patching, scaling, high availability, database management, and security hardening are all on your team.

### AWS Cognito

AWS Cognito is HIPAA eligible when covered under an AWS BAA. It is the cheapest option for basic authentication (first 50,000 MAU free in Cognito User Pools, then $0.0055/MAU). Cognito supports MFA (TOTP, SMS), OAuth2/OIDC flows, and Lambda triggers for custom logic. It integrates natively with the AWS ecosystem ([[AWS-HIPAA-Infrastructure]]).

Key weakness: limited customization of the auth UI, no native SMART on FHIR support (would require custom Lambda authorizers), and the user management console is significantly less polished than Auth0 or Clerk. Cognito's advanced authorization features (fine-grained permissions) require Verified Permissions, a separate service.

### Comparison Matrix

| Feature | Auth0 | Clerk | Keycloak | AWS Cognito |
|---|---|---|---|---|
| **HIPAA BAA** | Enterprise plan | Enterprise plan | N/A (self-hosted) | AWS BAA |
| **SMART on FHIR** | Via Actions (custom) | No native support | Community extensions | Via Lambda (custom) |
| **MFA Options** | TOTP, SMS, WebAuthn, Push | TOTP, SMS | TOTP, WebAuthn, SMS (SPI) | TOTP, SMS |
| **Pricing (1K MAU)** | ~$240/mo (Enterprise) | ~$200/mo (Enterprise) | Free (+ infra costs) | ~$5.50/mo |
| **Developer Experience** | Excellent | Best-in-class | Good (steep learning curve) | Limited |
| **Multi-Tenancy** | Organizations feature | Organizations feature | Realms | User Pool groups |
| **Healthcare Adoption** | High | Low-Medium | Medium | Medium |
| **Custom Token Claims** | Auth0 Actions | Webhooks | Protocol Mappers | Lambda Triggers |
| **Self-Hosted Option** | No | No | Yes (only option) | No |
| **Vendor Lock-in Risk** | Medium | Medium | None | High (AWS ecosystem) |
| **SOC 2 Type II** | Yes | Yes | Operator responsibility | Yes (AWS) |

### Recommendation

**Phase 1 (MVP): Auth0 Enterprise** with HIPAA BAA. Auth0 provides the fastest path to production with HIPAA compliance, mature MFA flows, and enough flexibility to implement SMART on FHIR scopes via Actions. The Organizations feature maps cleanly to our multi-tenant architecture ([[ADR-002-multi-tenancy-schema-per-tenant]]).

**Phase 3 (Long-term): Evaluate migration to Keycloak** if we need full SMART on FHIR App Launch compliance for third-party EHR integrations. Keycloak's FHIR extensions are the most complete open-source option, and self-hosting eliminates per-MAU costs at scale.

**Migration insurance**: Abstract the auth provider behind a service interface (`AuthService`) so switching providers requires changing only the adapter layer, not application code.

---

## 2. SMART on FHIR -- Complete Guide

### What is SMART on FHIR?

SMART on FHIR (Substitutable Medical Apps, Reusable Technologies on FHIR) is a set of open specifications built on top of OAuth 2.0, OpenID Connect, and HL7 FHIR. It defines a standardized way for third-party applications to authenticate with healthcare systems, request specific data access permissions, and interact with FHIR-based clinical data. See [[FHIR-R4-Deep-Dive]] for the underlying data model.

SMART on FHIR solves a critical problem: how do you let a third-party app (e.g., a diabetes management tool) securely access a patient's data from any EHR, without each EHR implementing a proprietary API? The answer is a standardized authorization framework layered on top of FHIR's RESTful API.

### SMART App Launch Framework

SMART defines two launch modes:

**EHR Launch**: The app is launched from within the EHR's user interface (e.g., a provider clicks "Open Diabetes Dashboard" inside Epic). The EHR passes a `launch` parameter to the app, which the app exchanges for context (current patient, encounter, etc.).

**Standalone Launch**: The app launches independently (e.g., a patient opens a mobile app). The app must discover the FHIR server's authorization endpoints, authenticate the user, and request the necessary scopes. No EHR context is provided automatically; the app must select a patient or rely on the authenticated user's identity.

### Authorization Scopes

SMART on FHIR v2 defines scopes using the syntax:

```
<context>/<resource>.<permissions>
```

**Context** defines whose data is being accessed:
- `patient/` -- Access limited to a single patient's data (the patient in context)
- `user/` -- Access limited by the authenticated user's permissions
- `system/` -- Backend service access (no user in the loop)

**Resource** is any FHIR resource type (e.g., `Patient`, `Observation`, `MedicationRequest`) or `*` for all resources.

**Permissions** use the CRUDS notation (SMART v2):
- `c` = Create
- `r` = Read
- `u` = Update
- `d` = Delete
- `s` = Search

Examples:
```
patient/Observation.rs      -- Read and search observations for the patient in context
user/Patient.cruds          -- Full CRUD + search on Patient resources (limited by user's access)
system/MedicationRequest.r  -- Backend service can read medication requests
patient/*.rs                -- Read and search all resource types for patient in context
launch/patient              -- Request patient context at launch time
launch/encounter            -- Request encounter context at launch time
openid fhirUser             -- Request OpenID Connect identity with FHIR user reference
```

For backwards compatibility with SMART v1, the legacy scope syntax is also supported:
```
patient/Observation.read    -- v1 syntax (maps to patient/Observation.rs in v2)
patient/Observation.write   -- v1 syntax (maps to patient/Observation.cud in v2)
```

### SMART v2 Granular Scopes with Filtering

SMART v2 introduced query-based scope filtering, allowing scopes to be narrowed by search parameters:

```
patient/Observation.rs?category=vital-signs    -- Only vital signs observations
patient/Condition.rs?clinical-status=active     -- Only active conditions
user/Observation.rs?category=laboratory         -- Only lab results for the user
```

### Complete Authorization Flow

The following 10-step flow covers the EHR Launch sequence:

```
Step 1: App Registration
    App registers with the EHR's authorization server.
    Provides: redirect_uri, requested scopes, app name, logo, JWKS URL.

Step 2: SMART Configuration Discovery
    App fetches: GET {fhir_base_url}/.well-known/smart-configuration
    Response includes: authorization_endpoint, token_endpoint,
    capabilities, scopes_supported, code_challenge_methods_supported.

Step 3: EHR Launch (EHR Launch mode only)
    EHR launches app with: GET {app_url}?iss={fhir_base_url}&launch={opaque_token}

Step 4: Authorization Request
    App redirects browser to authorization_endpoint:
    GET {authorization_endpoint}?
        response_type=code&
        client_id={client_id}&
        redirect_uri={redirect_uri}&
        scope=launch patient/Observation.rs openid fhirUser&
        state={random_state}&
        aud={fhir_base_url}&
        launch={launch_token}&           (EHR launch only)
        code_challenge={challenge}&       (PKCE)
        code_challenge_method=S256

Step 5: User Authentication
    Authorization server authenticates the user (if not already authenticated).
    May use MFA, SSO, or existing EHR session.

Step 6: User Authorization (Consent)
    Authorization server presents the requested scopes to the user.
    User approves or denies access.

Step 7: Authorization Code Grant
    Authorization server redirects to redirect_uri with code:
    GET {redirect_uri}?code={authorization_code}&state={state}

Step 8: Token Exchange
    App exchanges authorization code for tokens:
    POST {token_endpoint}
    Content-Type: application/x-www-form-urlencoded

    grant_type=authorization_code&
    code={authorization_code}&
    redirect_uri={redirect_uri}&
    client_id={client_id}&
    code_verifier={verifier}       (PKCE)

Step 9: Token Response (with FHIR context)
    {
        "access_token": "eyJ...",
        "token_type": "Bearer",
        "expires_in": 3600,
        "scope": "launch patient/Observation.rs openid fhirUser",
        "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g...",
        "id_token": "eyJ...",
        "patient": "Patient/12345",          // FHIR patient context
        "encounter": "Encounter/67890",      // FHIR encounter context
        "fhirUser": "Practitioner/98765",    // Authenticated user's FHIR identity
        "need_patient_banner": true,
        "smart_style_url": "https://ehr.example.com/styles/smart.json"
    }

Step 10: FHIR API Access
    App uses access_token to call FHIR API:
    GET {fhir_base_url}/Observation?patient=Patient/12345&category=vital-signs
    Authorization: Bearer {access_token}
```

### Token Refresh and Expiration Strategy

Access tokens should be short-lived (15-60 minutes). Refresh tokens should be longer-lived (8-24 hours) and support rotation (each use returns a new refresh token, invalidating the old one). For offline access (e.g., background data sync), request the `offline_access` scope; this grants a refresh token that does not expire with the user's session but should still have an absolute expiration (e.g., 90 days).

```python
# Token refresh implementation
import httpx
from datetime import datetime, timedelta

class SmartTokenManager:
    def __init__(self, token_endpoint: str, client_id: str):
        self.token_endpoint = token_endpoint
        self.client_id = client_id
        self._access_token: str | None = None
        self._refresh_token: str | None = None
        self._expires_at: datetime | None = None

    @property
    def is_expired(self) -> bool:
        if self._expires_at is None:
            return True
        # Refresh 60 seconds before actual expiration
        return datetime.utcnow() >= (self._expires_at - timedelta(seconds=60))

    async def get_access_token(self) -> str:
        if self.is_expired and self._refresh_token:
            await self._refresh()
        return self._access_token

    async def _refresh(self):
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_endpoint,
                data={
                    "grant_type": "refresh_token",
                    "refresh_token": self._refresh_token,
                    "client_id": self.client_id,
                },
            )
            response.raise_for_status()
            token_data = response.json()
            self._access_token = token_data["access_token"]
            self._refresh_token = token_data.get("refresh_token", self._refresh_token)
            self._expires_at = datetime.utcnow() + timedelta(
                seconds=token_data["expires_in"]
            )
```

---

## 3. RBAC + ABAC Hybrid Model

MedOS implements a hybrid model combining Role-Based Access Control (RBAC) for coarse-grained permissions with Attribute-Based Access Control (ABAC) for fine-grained, context-aware enforcement. This aligns with the multi-tenant architecture described in [[ADR-002-multi-tenancy-schema-per-tenant]].

### Role Definitions

| Role | Description | Scope |
|---|---|---|
| **Super Admin** | System-wide platform administration | All tenants, all resources |
| **Practice Admin** | Practice-level management | Single tenant, all resources within practice |
| **Physician** | Full clinical access to assigned patients | Single tenant, assigned patients |
| **Nurse / MA** | Clinical access with limited order authority | Single tenant, assigned patients, limited write |
| **Billing Staff** | Financial and insurance data access | Single tenant, financial resources only |
| **Front Desk** | Demographics and scheduling | Single tenant, demographics + scheduling only |
| **Patient** | Own health data access | Own data only (patient portal) |

### Permission Matrix

| Resource | Super Admin | Practice Admin | Physician | Nurse/MA | Billing | Front Desk | Patient |
|---|---|---|---|---|---|---|---|
| Patient Demographics | CRUDS | CRUDS | RS | RS | RS | CRUDS | R (own) |
| Clinical Notes | CRUDS | RS | CRUDS | RS | -- | -- | R (own) |
| Lab Results | CRUDS | RS | CRUDS | RS | -- | -- | R (own) |
| Medications | CRUDS | RS | CRUDS | R | -- | -- | R (own) |
| Orders | CRUDS | RS | CRUDS | CRS | -- | -- | -- |
| Billing/Claims | CRUDS | CRUDS | R | -- | CRUDS | -- | R (own) |
| Scheduling | CRUDS | CRUDS | RS | RS | -- | CRUDS | CRS (own) |
| Audit Logs | RS | RS | -- | -- | -- | -- | -- |
| User Management | CRUDS | CRUDS | -- | -- | -- | -- | -- |
| Practice Settings | CRUDS | CRUDS | -- | -- | -- | -- | -- |

Legend: C=Create, R=Read, U=Update, D=Delete, S=Search, -- = No Access

### ABAC Policies

ABAC adds attribute-based conditions evaluated at runtime:

**Tenant Isolation**: Every request is scoped to the user's tenant. A user in Tenant A cannot access any data in Tenant B, regardless of their role. This is enforced at the database query level (schema-per-tenant) and at the API gateway level.

**Provider-Patient Relationship**: Even within a tenant, a physician can only access full clinical data for patients they are assigned to (via CareTeam or active encounter). Unassigned patients show only summary/demographic data.

**Time-Based Access**: Covering providers receive temporary access that expires at the end of their coverage period. Residents rotating through a clinic get access tied to their rotation dates.

**Location-Based Access**: Admin functions (user management, practice settings) are restricted to known IP ranges or require VPN. Clinical access from unfamiliar locations triggers step-up MFA.

### Policy Engine Implementation

```python
from functools import wraps
from enum import Enum
from typing import Callable
from fastapi import HTTPException, Request, Depends


class Permission(str, Enum):
    CREATE = "create"
    READ = "read"
    UPDATE = "update"
    DELETE = "delete"
    SEARCH = "search"


class Resource(str, Enum):
    PATIENT = "Patient"
    OBSERVATION = "Observation"
    MEDICATION_REQUEST = "MedicationRequest"
    CLAIM = "Claim"
    APPOINTMENT = "Appointment"
    AUDIT_EVENT = "AuditEvent"
    CLINICAL_NOTE = "ClinicalNote"


class PolicyDecision(str, Enum):
    ALLOW = "allow"
    DENY = "deny"


# --- RBAC Permission Registry ---
ROLE_PERMISSIONS: dict[str, dict[str, set[str]]] = {
    "super_admin": {
        "*": {"create", "read", "update", "delete", "search"},
    },
    "practice_admin": {
        "Patient": {"create", "read", "update", "delete", "search"},
        "Appointment": {"create", "read", "update", "delete", "search"},
        "Claim": {"create", "read", "update", "delete", "search"},
        "AuditEvent": {"read", "search"},
        "ClinicalNote": {"read", "search"},
    },
    "physician": {
        "Patient": {"read", "search"},
        "Observation": {"create", "read", "update", "delete", "search"},
        "MedicationRequest": {"create", "read", "update", "delete", "search"},
        "ClinicalNote": {"create", "read", "update", "delete", "search"},
        "Claim": {"read"},
        "Appointment": {"read", "search"},
    },
    "nurse": {
        "Patient": {"read", "search"},
        "Observation": {"create", "read", "search"},
        "MedicationRequest": {"read"},
        "ClinicalNote": {"read", "search"},
        "Appointment": {"read", "search"},
    },
    "billing": {
        "Patient": {"read", "search"},
        "Claim": {"create", "read", "update", "delete", "search"},
    },
    "front_desk": {
        "Patient": {"create", "read", "update", "delete", "search"},
        "Appointment": {"create", "read", "update", "delete", "search"},
    },
    "patient": {
        "Patient": {"read"},
        "Observation": {"read"},
        "MedicationRequest": {"read"},
        "ClinicalNote": {"read"},
        "Claim": {"read"},
        "Appointment": {"create", "read", "search"},
    },
}


def check_rbac(role: str, resource: str, permission: str) -> bool:
    """Check if a role has a specific permission on a resource."""
    perms = ROLE_PERMISSIONS.get(role, {})
    # Check wildcard first (super_admin)
    if "*" in perms and permission in perms["*"]:
        return True
    return permission in perms.get(resource, set())


# --- ABAC Policy Functions ---

async def check_tenant_isolation(request: Request, resource_tenant_id: str) -> bool:
    """Ensure user can only access resources within their own tenant."""
    user_tenant_id = request.state.user.tenant_id
    return user_tenant_id == resource_tenant_id


async def check_provider_patient_relationship(
    request: Request, patient_id: str, db
) -> bool:
    """Check if the provider has an active care relationship with the patient."""
    user = request.state.user
    if user.role in ("super_admin", "practice_admin"):
        return True
    if user.role == "patient":
        return user.fhir_patient_id == patient_id
    # Check CareTeam membership or active encounter
    query = """
        SELECT EXISTS(
            SELECT 1 FROM care_team_members
            WHERE provider_id = $1 AND patient_id = $2
            AND (end_date IS NULL OR end_date > NOW())
        )
    """
    return await db.fetchval(query, user.provider_id, patient_id)


async def check_time_based_access(
    request: Request, resource_id: str, db
) -> bool:
    """Check if temporary access grants are still valid."""
    user = request.state.user
    query = """
        SELECT EXISTS(
            SELECT 1 FROM temporary_access_grants
            WHERE grantee_id = $1
            AND (resource_id = $2 OR resource_id = '*')
            AND starts_at <= NOW()
            AND expires_at > NOW()
            AND revoked = FALSE
        )
    """
    return await db.fetchval(query, user.id, resource_id)


# --- Policy Engine Decorator ---

def require_permission(resource: Resource, permission: Permission):
    """FastAPI dependency that enforces RBAC + ABAC policies."""
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(request: Request, *args, **kwargs):
            user = request.state.user

            # Step 1: RBAC check
            if not check_rbac(user.role, resource.value, permission.value):
                raise HTTPException(
                    status_code=403,
                    detail=f"Role '{user.role}' lacks '{permission.value}' "
                           f"permission on '{resource.value}'",
                )

            # Step 2: Tenant isolation (always enforced)
            resource_tenant_id = kwargs.get("tenant_id") or request.path_params.get(
                "tenant_id"
            )
            if resource_tenant_id:
                if not await check_tenant_isolation(request, resource_tenant_id):
                    raise HTTPException(status_code=403, detail="Tenant access denied")

            # Step 3: Break-the-glass check (handled separately)
            # Step 4: Log the access attempt (always)
            await log_access_attempt(request, resource, permission, "allowed")

            return await func(request, *args, **kwargs)

        return wrapper
    return decorator


async def log_access_attempt(
    request: Request, resource: Resource, permission: Permission, outcome: str
):
    """Log every access attempt for audit trail."""
    # Implementation in Section 8 below
    pass


# --- Usage Example ---
# @app.get("/api/v1/patients/{patient_id}/observations")
# @require_permission(Resource.OBSERVATION, Permission.READ)
# async def get_patient_observations(request: Request, patient_id: str):
#     ...
```

---

## 4. Multi-Factor Authentication

### HIPAA 2025/2026 MFA Mandate

The proposed HIPAA Security Rule amendments (NPRM published January 2025, final rule expected mid-2026) eliminate the distinction between "required" and "addressable" safeguards. MFA is now mandatory for all access to electronic Protected Health Information (ePHI) -- not just remote access but internal access as well. This is the most significant HIPAA Security Rule change in nearly 20 years.

### MFA Methods (Ranked by Security)

1. **WebAuthn / Passkeys (Preferred)**: Hardware-bound credentials (YubiKey, device biometrics). Phishing-resistant, no shared secrets. FIDO2/WebAuthn is the gold standard.
2. **TOTP (Google Authenticator, Authy)**: Time-based one-time passwords. Good security, widely supported. Requires user to have a smartphone or hardware token.
3. **SMS OTP (Backup Only)**: Vulnerable to SIM swapping and SS7 attacks. Acceptable only as a fallback when other methods are unavailable. NIST SP 800-63B rates SMS as "restricted" for authentication.

### Adaptive MFA (Risk-Based)

Not every login requires the same friction. Adaptive MFA evaluates risk signals and escalates authentication requirements dynamically:

```python
# Risk scoring for adaptive MFA
def calculate_auth_risk(request: Request, user) -> int:
    """Returns risk score 0-100. Higher = more authentication required."""
    score = 0

    # New device (not in user's registered device list)
    device_fingerprint = request.headers.get("X-Device-Fingerprint")
    if device_fingerprint not in user.registered_devices:
        score += 30

    # Unusual location (IP geolocation differs from norm)
    ip_location = geolocate(request.client.host)
    if ip_location not in user.usual_locations:
        score += 25

    # Unusual time (outside normal working hours)
    if not is_within_working_hours(user):
        score += 15

    # Sensitive action (admin function, break-the-glass, bulk export)
    if request.url.path in SENSITIVE_ENDPOINTS:
        score += 20

    # Failed recent attempts
    recent_failures = get_recent_failed_logins(user.id, minutes=30)
    score += min(recent_failures * 10, 30)

    return min(score, 100)


# MFA enforcement based on risk score
# Score 0-20:   Session cookie sufficient (no MFA prompt)
# Score 21-50:  Require TOTP or WebAuthn
# Score 51-75:  Require WebAuthn specifically (phishing-resistant)
# Score 76-100: Require WebAuthn + manager approval for admin actions
```

### Auth0 MFA Configuration

```javascript
// Auth0 Action: Post-Login - Enforce MFA
exports.onExecutePostLogin = async (event, api) => {
  const isDoctorOrAdmin = ["physician", "practice_admin", "super_admin"].includes(
    event.user.app_metadata?.role
  );

  // Always require MFA for clinical and admin roles
  if (isDoctorOrAdmin && !event.authentication?.methods?.find(m => m.name === "mfa")) {
    api.authentication.challengeWithAny([
      { type: "webauthn-roaming" },
      { type: "webauthn-platform" },
      { type: "otp" },
    ]);
  }

  // For all users: require MFA on new device
  const isNewDevice = !event.user.app_metadata?.known_devices?.includes(
    event.request.ip
  );
  if (isNewDevice && !event.authentication?.methods?.find(m => m.name === "mfa")) {
    api.authentication.challengeWithAny([
      { type: "otp" },
      { type: "webauthn-platform" },
    ]);
  }
};
```

---

## 5. Session Management

### Redis-Backed Sessions

All sessions are stored in Redis with structured keys. This enables fast lookup, tenant-scoped session listing, and efficient invalidation.

```python
import redis.asyncio as redis
import json
import uuid
from datetime import datetime, timedelta

SESSION_TTL_SECONDS = 1800  # 30 minutes (HIPAA auto-logoff)
MAX_CONCURRENT_SESSIONS = 3

class SessionManager:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def create_session(self, user_id: str, tenant_id: str, metadata: dict) -> str:
        session_id = str(uuid.uuid4())
        session_data = {
            "user_id": user_id,
            "tenant_id": tenant_id,
            "created_at": datetime.utcnow().isoformat(),
            "last_activity": datetime.utcnow().isoformat(),
            "ip_address": metadata.get("ip_address"),
            "user_agent": metadata.get("user_agent"),
            "mfa_verified": metadata.get("mfa_verified", False),
        }

        # Enforce concurrent session limit
        user_sessions_key = f"user_sessions:{user_id}"
        existing_sessions = await self.redis.smembers(user_sessions_key)
        if len(existing_sessions) >= MAX_CONCURRENT_SESSIONS:
            # Evict oldest session
            oldest = sorted(existing_sessions)[0]
            await self.destroy_session(oldest.decode(), user_id)

        # Store session
        session_key = f"session:{session_id}"
        await self.redis.setex(session_key, SESSION_TTL_SECONDS, json.dumps(session_data))
        await self.redis.sadd(user_sessions_key, session_id)
        await self.redis.expire(user_sessions_key, SESSION_TTL_SECONDS * 2)

        return session_id

    async def validate_session(self, session_id: str) -> dict | None:
        session_key = f"session:{session_id}"
        data = await self.redis.get(session_key)
        if not data:
            return None
        session = json.loads(data)
        # Refresh TTL on activity (sliding window)
        session["last_activity"] = datetime.utcnow().isoformat()
        await self.redis.setex(session_key, SESSION_TTL_SECONDS, json.dumps(session))
        return session

    async def destroy_session(self, session_id: str, user_id: str):
        await self.redis.delete(f"session:{session_id}")
        await self.redis.srem(f"user_sessions:{user_id}", session_id)

    async def invalidate_all_sessions(self, user_id: str):
        """Called on password change, account compromise, or role change."""
        user_sessions_key = f"user_sessions:{user_id}"
        session_ids = await self.redis.smembers(user_sessions_key)
        if session_ids:
            session_keys = [f"session:{sid.decode()}" for sid in session_ids]
            await self.redis.delete(*session_keys)
            await self.redis.delete(user_sessions_key)
```

### Session Timeout Requirements

- **Inactivity timeout**: 15-30 minutes (configurable per practice, minimum 15 minutes per HIPAA)
- **Absolute timeout**: 12 hours (session expires regardless of activity)
- **Sensitive action re-auth**: Require password/MFA for changing security settings, viewing full SSN, break-the-glass
- **Password change**: Invalidate all other sessions immediately
- **Role change**: Invalidate all sessions, force re-login to pick up new permissions

---

## 6. Break-the-Glass (Emergency Access)

Break-the-glass (BTG) is an emergency override mechanism allowing providers to access patient data outside their normal authorization scope. This is critical for emergency medicine where the treating provider may have no prior relationship with the patient.

### Trigger Conditions

- Patient arrives unconscious in the emergency department
- Provider not assigned to patient but needs immediate clinical data
- After-hours coverage where normal assignment workflows have not been completed
- System-level emergency (e.g., mass casualty event requiring broad data access)

### Implementation

```python
from datetime import datetime, timedelta
from enum import Enum


class BTGReason(str, Enum):
    EMERGENCY = "emergency"
    COVERING_PROVIDER = "covering_provider"
    CONSULTATION_URGENT = "consultation_urgent"
    MASS_CASUALTY = "mass_casualty"


BTG_DURATION_HOURS = 4  # Auto-expires after 4 hours


class BreakTheGlassService:
    def __init__(self, db, audit_service, notification_service):
        self.db = db
        self.audit = audit_service
        self.notify = notification_service

    async def request_btg_access(
        self,
        provider_id: str,
        patient_id: str,
        reason: BTGReason,
        justification_text: str,
        tenant_id: str,
    ) -> dict:
        """
        Grant emergency access with full audit trail.
        Justification text is mandatory and stored in FHIR AuditEvent.
        """
        if not justification_text or len(justification_text) < 10:
            raise ValueError(
                "Break-the-glass requires a justification of at least 10 characters"
            )

        # Create the BTG grant
        grant = {
            "id": generate_uuid(),
            "provider_id": provider_id,
            "patient_id": patient_id,
            "tenant_id": tenant_id,
            "reason": reason.value,
            "justification": justification_text,
            "granted_at": datetime.utcnow(),
            "expires_at": datetime.utcnow() + timedelta(hours=BTG_DURATION_HOURS),
            "revoked": False,
            "reviewed": False,
            "reviewed_by": None,
            "reviewed_at": None,
        }

        await self.db.execute(
            """
            INSERT INTO break_the_glass_grants
            (id, provider_id, patient_id, tenant_id, reason, justification,
             granted_at, expires_at, revoked, reviewed)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
            """,
            *grant.values(),
        )

        # Create FHIR AuditEvent
        audit_event = {
            "resourceType": "AuditEvent",
            "type": {
                "system": "http://dicom.nema.org/resources/ontology/DCM",
                "code": "110113",
                "display": "Security Alert",
            },
            "subtype": [
                {
                    "system": "https://medos.health/fhir/audit-subtypes",
                    "code": "break-the-glass",
                    "display": "Break-the-Glass Emergency Access",
                }
            ],
            "action": "E",  # Execute
            "recorded": datetime.utcnow().isoformat() + "Z",
            "outcome": "0",  # Success
            "agent": [
                {
                    "who": {"reference": f"Practitioner/{provider_id}"},
                    "requestor": True,
                    "policy": [f"urn:medos:btg:{grant['id']}"],
                }
            ],
            "entity": [
                {
                    "what": {"reference": f"Patient/{patient_id}"},
                    "role": {
                        "system": "http://terminology.hl7.org/CodeSystem/object-role",
                        "code": "1",
                        "display": "Patient",
                    },
                    "detail": [
                        {"type": "reason", "valueString": reason.value},
                        {"type": "justification", "valueString": justification_text},
                    ],
                }
            ],
        }

        await self.audit.log_fhir_audit_event(audit_event)

        # Notify practice admin and privacy officer
        await self.notify.send_btg_alert(
            tenant_id=tenant_id,
            provider_name=await self._get_provider_name(provider_id),
            patient_id=patient_id,
            reason=reason.value,
            justification=justification_text,
        )

        return grant

    async def check_btg_access(self, provider_id: str, patient_id: str) -> bool:
        """Check if an active BTG grant exists."""
        result = await self.db.fetchval(
            """
            SELECT EXISTS(
                SELECT 1 FROM break_the_glass_grants
                WHERE provider_id = $1
                AND patient_id = $2
                AND expires_at > NOW()
                AND revoked = FALSE
            )
            """,
            provider_id,
            patient_id,
        )
        return result

    async def review_btg_grant(
        self, grant_id: str, reviewer_id: str, approved: bool, notes: str
    ):
        """Post-access review by privacy officer or practice admin."""
        await self.db.execute(
            """
            UPDATE break_the_glass_grants
            SET reviewed = TRUE, reviewed_by = $2, reviewed_at = NOW(),
                review_approved = $3, review_notes = $4
            WHERE id = $1
            """,
            grant_id,
            reviewer_id,
            approved,
            notes,
        )

    async def get_pending_reviews(self, tenant_id: str) -> list[dict]:
        """Get all BTG grants that have not been reviewed yet."""
        return await self.db.fetch(
            """
            SELECT * FROM break_the_glass_grants
            WHERE tenant_id = $1 AND reviewed = FALSE
            ORDER BY granted_at DESC
            """,
            tenant_id,
        )

    async def _get_provider_name(self, provider_id: str) -> str:
        return await self.db.fetchval(
            "SELECT name FROM providers WHERE id = $1", provider_id
        )
```

---

## 7. API Key Management for Third-Party Apps

### SMART App Registration Process

Third-party applications register through a developer portal. Each registration produces a `client_id` (and `client_secret` for confidential clients) along with approved scopes. The registration includes:

- Application name, description, and logo
- Redirect URIs (whitelist)
- Requested SMART scopes (reviewed and approved by practice admin)
- Launch mode (EHR launch, standalone, or both)
- Client type (public or confidential)
- JWKS URL (for asymmetric client authentication)

### Client Credentials Flow (System-to-System)

For backend services that need access without a user in the loop (e.g., a lab system pushing results), use the OAuth2 Client Credentials grant with a signed JWT assertion:

```python
import jwt
import time
import httpx

def create_client_assertion(client_id: str, token_endpoint: str, private_key: str) -> str:
    """Create a signed JWT for client authentication (RFC 7523)."""
    now = int(time.time())
    payload = {
        "iss": client_id,
        "sub": client_id,
        "aud": token_endpoint,
        "iat": now,
        "exp": now + 300,  # 5 minute expiration
        "jti": str(uuid.uuid4()),
    }
    return jwt.encode(payload, private_key, algorithm="RS384")


async def get_system_access_token(
    client_id: str,
    token_endpoint: str,
    private_key: str,
    scopes: list[str],
) -> dict:
    """Obtain an access token using client credentials with JWT assertion."""
    assertion = create_client_assertion(client_id, token_endpoint, private_key)

    async with httpx.AsyncClient() as client:
        response = await client.post(
            token_endpoint,
            data={
                "grant_type": "client_credentials",
                "scope": " ".join(scopes),
                "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
                "client_assertion": assertion,
            },
        )
        response.raise_for_status()
        return response.json()
```

### API Key Rotation Strategy

- **Rotation period**: Every 90 days (mandatory), or immediately upon suspected compromise
- **Grace period**: Old key remains valid for 48 hours after rotation to allow clients to update
- **Notification**: Automated email to registered developer 14 days, 7 days, and 1 day before expiration
- **Revocation**: Immediate revocation available via admin API for compromised keys

### Rate Limiting per Client

```python
# Rate limits by client tier
RATE_LIMITS = {
    "tier_free":       {"requests_per_minute": 60,   "requests_per_day": 10_000},
    "tier_standard":   {"requests_per_minute": 300,  "requests_per_day": 100_000},
    "tier_enterprise": {"requests_per_minute": 1000, "requests_per_day": 1_000_000},
}
```

---

## 8. Audit Trail Implementation

Every access to ePHI must be logged in an immutable audit trail. This is a HIPAA requirement (45 CFR 164.312(b)) and is essential for breach investigation, compliance reporting, and detecting unauthorized access patterns. See [[HIPAA-Deep-Dive]] for the full regulatory context.

### FHIR AuditEvent Resource

MedOS logs all access as FHIR AuditEvent resources, following the IHE Basic Audit Log Patterns (BALP) profile:

| Field | Description | Example |
|---|---|---|
| `type` | Category of event | `rest` (RESTful operation), `110113` (security alert) |
| `subtype` | Specific action | `read`, `create`, `break-the-glass` |
| `action` | CRUD action code | `R` (Read), `C` (Create), `U` (Update), `D` (Delete) |
| `recorded` | Timestamp (UTC) | `2026-02-27T14:30:00Z` |
| `outcome` | Success or failure | `0` (Success), `4` (Minor failure), `8` (Serious failure) |
| `agent` | Who performed the action | `Practitioner/12345` with role, IP, device |
| `source` | System that recorded the event | `medos-api-server-01` |
| `entity` | What was accessed | `Patient/67890`, `Observation/11111` |

### Immutable Storage

Audit logs must be append-only and tamper-evident:

- **Primary store**: PostgreSQL append-only table with row-level security preventing UPDATE and DELETE
- **Archive**: Nightly export to S3 with Object Lock (WORM - Write Once Read Many) in Governance mode
- **Retention**: 6 years minimum (HIPAA requirement), 10 years recommended
- **Integrity**: SHA-256 hash chain linking consecutive audit entries

### Complete Audit Middleware

```python
import hashlib
import json
from datetime import datetime
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp


class AuditMiddleware(BaseHTTPMiddleware):
    """Middleware that logs every API request as a FHIR AuditEvent."""

    # Endpoints that access ePHI and require audit logging
    AUDITABLE_PREFIXES = [
        "/api/v1/patients",
        "/api/v1/observations",
        "/api/v1/conditions",
        "/api/v1/medications",
        "/api/v1/encounters",
        "/api/v1/claims",
        "/api/v1/clinical-notes",
    ]

    def __init__(self, app: ASGIApp, db_pool, audit_repo):
        super().__init__(app)
        self.db_pool = db_pool
        self.audit_repo = audit_repo

    async def dispatch(self, request: Request, call_next) -> Response:
        # Skip non-auditable endpoints (health checks, static assets)
        if not any(request.url.path.startswith(p) for p in self.AUDITABLE_PREFIXES):
            return await call_next(request)

        start_time = datetime.utcnow()
        response = None
        outcome = "0"  # Assume success

        try:
            response = await call_next(request)
            if response.status_code >= 400:
                outcome = "4" if response.status_code < 500 else "8"
            return response
        except Exception as exc:
            outcome = "8"  # Serious failure
            raise
        finally:
            await self._create_audit_event(request, response, outcome, start_time)

    async def _create_audit_event(
        self, request: Request, response: Response | None, outcome: str, start_time: datetime
    ):
        user = getattr(request.state, "user", None)
        action_map = {
            "GET": "R",
            "POST": "C",
            "PUT": "U",
            "PATCH": "U",
            "DELETE": "D",
        }

        # Extract patient ID from URL path if present
        patient_id = self._extract_patient_id(request.url.path)

        # Determine the FHIR resource type being accessed
        resource_type = self._extract_resource_type(request.url.path)

        audit_event = {
            "resource_type": "AuditEvent",
            "type_code": "rest",
            "subtype_code": request.method.lower(),
            "action": action_map.get(request.method, "E"),
            "recorded": start_time.isoformat() + "Z",
            "outcome": outcome,
            "outcome_desc": self._outcome_description(outcome),
            # Agent (who)
            "agent_user_id": user.id if user else "anonymous",
            "agent_role": user.role if user else "unknown",
            "agent_tenant_id": user.tenant_id if user else None,
            "agent_ip": request.client.host if request.client else "unknown",
            "agent_user_agent": request.headers.get("user-agent", ""),
            # Source (where)
            "source_site": "medos-api",
            "source_observer": request.headers.get("X-Request-ID", ""),
            # Entity (what)
            "entity_patient_id": patient_id,
            "entity_resource_type": resource_type,
            "entity_query": str(request.query_params) if request.query_params else None,
            # Integrity
            "previous_hash": None,  # Set by repository
        }

        await self.audit_repo.append(audit_event)

    def _extract_patient_id(self, path: str) -> str | None:
        parts = path.split("/")
        try:
            idx = parts.index("patients")
            if idx + 1 < len(parts):
                return parts[idx + 1]
        except ValueError:
            pass
        return None

    def _extract_resource_type(self, path: str) -> str:
        resource_map = {
            "patients": "Patient",
            "observations": "Observation",
            "conditions": "Condition",
            "medications": "MedicationRequest",
            "encounters": "Encounter",
            "claims": "Claim",
            "clinical-notes": "DocumentReference",
        }
        parts = path.split("/")
        for part in parts:
            if part in resource_map:
                return resource_map[part]
        return "Unknown"

    def _outcome_description(self, outcome: str) -> str:
        return {"0": "Success", "4": "Minor failure", "8": "Serious failure"}.get(
            outcome, "Unknown"
        )


class AuditRepository:
    """Append-only audit log repository with hash chain integrity."""

    def __init__(self, db_pool):
        self.db_pool = db_pool

    async def append(self, audit_event: dict):
        async with self.db_pool.acquire() as conn:
            # Get the hash of the previous entry for chain integrity
            previous_hash = await conn.fetchval(
                "SELECT entry_hash FROM audit_log ORDER BY id DESC LIMIT 1"
            )
            audit_event["previous_hash"] = previous_hash or "GENESIS"

            # Compute hash of this entry
            entry_content = json.dumps(audit_event, sort_keys=True, default=str)
            entry_hash = hashlib.sha256(entry_content.encode()).hexdigest()

            await conn.execute(
                """
                INSERT INTO audit_log (
                    resource_type, type_code, subtype_code, action,
                    recorded, outcome, outcome_desc,
                    agent_user_id, agent_role, agent_tenant_id,
                    agent_ip, agent_user_agent,
                    source_site, source_observer,
                    entity_patient_id, entity_resource_type, entity_query,
                    previous_hash, entry_hash
                ) VALUES (
                    $1, $2, $3, $4, $5, $6, $7, $8, $9, $10,
                    $11, $12, $13, $14, $15, $16, $17, $18, $19
                )
                """,
                audit_event["resource_type"],
                audit_event["type_code"],
                audit_event["subtype_code"],
                audit_event["action"],
                audit_event["recorded"],
                audit_event["outcome"],
                audit_event["outcome_desc"],
                audit_event["agent_user_id"],
                audit_event["agent_role"],
                audit_event["agent_tenant_id"],
                audit_event["agent_ip"],
                audit_event["agent_user_agent"],
                audit_event["source_site"],
                audit_event["source_observer"],
                audit_event["entity_patient_id"],
                audit_event["entity_resource_type"],
                audit_event["entity_query"],
                audit_event["previous_hash"],
                entry_hash,
            )
```

### Audit Log Schema (PostgreSQL)

```sql
CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    resource_type       VARCHAR(50) NOT NULL DEFAULT 'AuditEvent',
    type_code           VARCHAR(50) NOT NULL,
    subtype_code        VARCHAR(50),
    action              CHAR(1) NOT NULL,              -- C, R, U, D, E
    recorded            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    outcome             CHAR(1) NOT NULL,              -- 0, 4, 8, 12
    outcome_desc        VARCHAR(255),
    agent_user_id       VARCHAR(255) NOT NULL,
    agent_role          VARCHAR(100),
    agent_tenant_id     VARCHAR(255),
    agent_ip            INET,
    agent_user_agent    TEXT,
    source_site         VARCHAR(255),
    source_observer     VARCHAR(255),
    entity_patient_id   VARCHAR(255),
    entity_resource_type VARCHAR(100),
    entity_query        TEXT,
    previous_hash       VARCHAR(64),
    entry_hash          VARCHAR(64) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Append-only enforcement: prevent UPDATE and DELETE
REVOKE UPDATE, DELETE ON audit_log FROM app_user;

-- Indexes for common query patterns
CREATE INDEX idx_audit_agent_user ON audit_log (agent_user_id, recorded DESC);
CREATE INDEX idx_audit_patient ON audit_log (entity_patient_id, recorded DESC);
CREATE INDEX idx_audit_tenant ON audit_log (agent_tenant_id, recorded DESC);
CREATE INDEX idx_audit_outcome ON audit_log (outcome) WHERE outcome != '0';

-- Partition by month for performance
-- (Implementation depends on PostgreSQL version; use declarative partitioning)
```

### Celebrity Patient Snooping Detection

```python
from collections import defaultdict
from datetime import datetime, timedelta


class SnoopingDetector:
    """
    Detect unauthorized access patterns that suggest 'snooping' --
    accessing celebrity, VIP, or colleague patient records without
    a legitimate clinical reason.
    """

    # Thresholds for anomaly detection
    UNIQUE_ACCESSORS_THRESHOLD = 5      # More than 5 unique users accessing same patient in 24h
    ACCESS_WITHOUT_ENCOUNTER_THRESHOLD = 3  # 3+ accesses without an active encounter
    AFTER_HOURS_THRESHOLD = 3           # 3+ after-hours accesses to same patient

    async def analyze_access_patterns(self, db, tenant_id: str, lookback_hours: int = 24):
        """Run snooping detection analysis. Called by scheduled job."""
        since = datetime.utcnow() - timedelta(hours=lookback_hours)
        alerts = []

        # Pattern 1: Unusually high number of unique users accessing one patient
        high_access_patients = await db.fetch(
            """
            SELECT entity_patient_id, COUNT(DISTINCT agent_user_id) as unique_users,
                   ARRAY_AGG(DISTINCT agent_user_id) as user_ids
            FROM audit_log
            WHERE agent_tenant_id = $1
              AND recorded > $2
              AND entity_patient_id IS NOT NULL
              AND action = 'R'
            GROUP BY entity_patient_id
            HAVING COUNT(DISTINCT agent_user_id) > $3
            """,
            tenant_id, since, self.UNIQUE_ACCESSORS_THRESHOLD,
        )
        for row in high_access_patients:
            alerts.append({
                "type": "high_unique_accessors",
                "patient_id": row["entity_patient_id"],
                "unique_users": row["unique_users"],
                "user_ids": row["user_ids"],
                "severity": "high",
            })

        # Pattern 2: Access without active encounter or care team membership
        unrelated_access = await db.fetch(
            """
            SELECT al.agent_user_id, al.entity_patient_id, COUNT(*) as access_count
            FROM audit_log al
            LEFT JOIN care_team_members ctm
              ON al.agent_user_id = ctm.provider_id
              AND al.entity_patient_id = ctm.patient_id
              AND (ctm.end_date IS NULL OR ctm.end_date > NOW())
            WHERE al.agent_tenant_id = $1
              AND al.recorded > $2
              AND al.entity_patient_id IS NOT NULL
              AND al.action = 'R'
              AND ctm.id IS NULL
              AND al.agent_role IN ('physician', 'nurse')
            GROUP BY al.agent_user_id, al.entity_patient_id
            HAVING COUNT(*) >= $3
            """,
            tenant_id, since, self.ACCESS_WITHOUT_ENCOUNTER_THRESHOLD,
        )
        for row in unrelated_access:
            alerts.append({
                "type": "access_without_relationship",
                "user_id": row["agent_user_id"],
                "patient_id": row["entity_patient_id"],
                "access_count": row["access_count"],
                "severity": "critical",
            })

        return alerts
```

---

## 9. Implementation Plan

### Phase 1 -- MVP (Months 1-3)

**Goal**: Secure authentication with basic authorization, meeting HIPAA minimum requirements.

| Component | Implementation | Details |
|---|---|---|
| Identity Provider | Auth0 Enterprise | HIPAA BAA signed, Organizations enabled |
| Authentication | Email/password + MFA | TOTP mandatory for all clinical roles |
| Authorization | RBAC (7 roles) | Permission matrix enforced via middleware |
| Session Management | Redis-backed | 30-min inactivity timeout, concurrent session limits |
| Audit Logging | PostgreSQL append-only | All ePHI access logged, 6-year retention |
| Multi-Tenancy | Tenant ID in JWT claims | Enforced at middleware + database query level |

**Auth0 Configuration**:
```javascript
// Auth0 Action: Post-Login - Add MedOS claims to token
exports.onExecutePostLogin = async (event, api) => {
  const namespace = "https://medos.health/claims";
  const { role, tenant_id, provider_id, fhir_user } = event.user.app_metadata || {};

  api.accessToken.setCustomClaim(`${namespace}/role`, role);
  api.accessToken.setCustomClaim(`${namespace}/tenant_id`, tenant_id);
  api.accessToken.setCustomClaim(`${namespace}/provider_id`, provider_id);
  api.accessToken.setCustomClaim(`${namespace}/fhir_user`, fhir_user);

  // Set organization context
  if (event.organization) {
    api.accessToken.setCustomClaim(`${namespace}/org_id`, event.organization.id);
    api.accessToken.setCustomClaim(`${namespace}/org_name`, event.organization.name);
  }
};
```

### Phase 2 -- SMART on FHIR + ABAC (Months 4-6)

**Goal**: Full SMART on FHIR scope support, attribute-based policies, break-the-glass.

| Component | Implementation | Details |
|---|---|---|
| SMART Scopes | Custom scope parser + enforcer | v2 CRUDS syntax, query-based filtering |
| ABAC Engine | Policy evaluation middleware | Provider-patient relationship, time-based access |
| Break-the-Glass | BTG service + review workflow | 4-hour grants, mandatory justification, admin alerts |
| FHIR AuditEvent | Structured audit log in FHIR format | IHE BALP profile compliance |
| `.well-known/smart-configuration` | Discovery endpoint | Advertise supported scopes, endpoints, capabilities |

**SMART Configuration Discovery Endpoint**:
```json
{
  "issuer": "https://auth.medos.health",
  "jwks_uri": "https://auth.medos.health/.well-known/jwks.json",
  "authorization_endpoint": "https://auth.medos.health/authorize",
  "token_endpoint": "https://auth.medos.health/oauth/token",
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post",
    "private_key_jwt"
  ],
  "registration_endpoint": "https://auth.medos.health/oauth/register",
  "scopes_supported": [
    "openid", "fhirUser", "launch", "launch/patient", "launch/encounter",
    "offline_access",
    "patient/*.cruds", "patient/Patient.rs", "patient/Observation.rs",
    "patient/Condition.rs", "patient/MedicationRequest.rs",
    "patient/AllergyIntolerance.rs", "patient/Procedure.rs",
    "patient/Immunization.rs", "patient/DiagnosticReport.rs",
    "user/*.cruds", "user/Patient.cruds", "user/Observation.cruds",
    "system/*.cruds", "system/Patient.rs"
  ],
  "response_types_supported": ["code"],
  "capabilities": [
    "launch-ehr",
    "launch-standalone",
    "client-public",
    "client-confidential-symmetric",
    "client-confidential-asymmetric",
    "context-ehr-patient",
    "context-ehr-encounter",
    "context-standalone-patient",
    "sso-openid-connect",
    "permission-v2",
    "permission-offline"
  ],
  "code_challenge_methods_supported": ["S256"]
}
```

### Phase 3 -- Third-Party App Marketplace (Months 7-12)

**Goal**: Open platform for third-party SMART apps with advanced analytics.

| Component | Implementation | Details |
|---|---|---|
| Developer Portal | Self-service app registration | Scope approval workflow, sandbox testing |
| App Marketplace | Curated app directory | Practice admins can enable/disable apps per tenant |
| Client Credentials | System-to-system auth | JWT assertion-based, for lab systems, pharmacy, etc. |
| Rate Limiting | Per-client tiered limits | Free/Standard/Enterprise tiers |
| Snooping Detection | ML-based anomaly detection | Scheduled analysis of access patterns |
| Audit Analytics | Dashboard + compliance reports | HIPAA-required access reports on demand |

### Migration Path (Auth0 to Keycloak)

If migrating to Keycloak becomes necessary:

1. **AuthService interface** already abstracts the provider; implement `KeycloakAuthAdapter`
2. **User migration**: Auth0 Management API exports users (passwords must be re-hashed; trigger password reset flow)
3. **Token format**: Keep the same custom claims namespace (`https://medos.health/claims`)
4. **Parallel run**: Run both providers for 30 days with feature flag routing
5. **SMART extensions**: Deploy [keycloak-extensions-for-fhir](https://github.com/Alvearie/keycloak-extensions-for-fhir) for native FHIR scope support

---

## 10. Security Testing

### Auth Bypass Testing Checklist

| Test | Method | Expected Result |
|---|---|---|
| Missing Authorization header | Send request without `Authorization` header | 401 Unauthorized |
| Expired token | Use a token past its `exp` claim | 401 Unauthorized |
| Token for wrong tenant | Use Tenant A's token against Tenant B's endpoint | 403 Forbidden |
| Modified token payload | Alter JWT claims without re-signing | 401 (signature invalid) |
| Algorithm confusion | Send `alg: none` or `alg: HS256` when RS256 expected | 401 Rejected |
| Scope escalation | Request `patient/*.cruds` when only `patient/Observation.rs` granted | 403 Forbidden |
| Role manipulation | Change `role` claim in token | 401 (signature invalid) |
| IDOR via patient ID | Access `/patients/{other_patient_id}` as patient role | 403 Forbidden |
| BTG without justification | Request break-the-glass with empty justification | 400 Bad Request |
| Expired BTG grant | Access patient data after 4-hour BTG window | 403 Forbidden |

### Token Manipulation Tests

```python
import pytest
import jwt


class TestTokenSecurity:
    """Security tests for JWT token handling."""

    def test_reject_none_algorithm(self, client, valid_token_payload):
        """Tokens with alg:none must be rejected."""
        forged = jwt.encode(valid_token_payload, key="", algorithm="none")
        response = client.get("/api/v1/patients", headers={"Authorization": f"Bearer {forged}"})
        assert response.status_code == 401

    def test_reject_algorithm_confusion(self, client, valid_token_payload, public_key):
        """Tokens signed with HMAC using the public key must be rejected."""
        forged = jwt.encode(valid_token_payload, key=public_key, algorithm="HS256")
        response = client.get("/api/v1/patients", headers={"Authorization": f"Bearer {forged}"})
        assert response.status_code == 401

    def test_reject_expired_token(self, client, expired_token):
        """Expired tokens must be rejected."""
        response = client.get(
            "/api/v1/patients", headers={"Authorization": f"Bearer {expired_token}"}
        )
        assert response.status_code == 401

    def test_reject_future_nbf(self, client, future_nbf_token):
        """Tokens with nbf in the future must be rejected."""
        response = client.get(
            "/api/v1/patients", headers={"Authorization": f"Bearer {future_nbf_token}"}
        )
        assert response.status_code == 401

    def test_reject_wrong_audience(self, client, wrong_audience_token):
        """Tokens with incorrect aud claim must be rejected."""
        response = client.get(
            "/api/v1/patients", headers={"Authorization": f"Bearer {wrong_audience_token}"}
        )
        assert response.status_code == 401

    def test_tenant_isolation(self, client, tenant_a_token):
        """Tenant A token must not access Tenant B resources."""
        response = client.get(
            "/api/v1/tenants/tenant-b/patients",
            headers={"Authorization": f"Bearer {tenant_a_token}"},
        )
        assert response.status_code == 403
```

### Privilege Escalation Tests

```python
class TestPrivilegeEscalation:
    """Tests to verify role boundaries cannot be bypassed."""

    def test_nurse_cannot_delete_patient(self, client, nurse_token):
        response = client.delete(
            "/api/v1/patients/patient-123",
            headers={"Authorization": f"Bearer {nurse_token}"},
        )
        assert response.status_code == 403

    def test_billing_cannot_read_clinical_notes(self, client, billing_token):
        response = client.get(
            "/api/v1/patients/patient-123/clinical-notes",
            headers={"Authorization": f"Bearer {billing_token}"},
        )
        assert response.status_code == 403

    def test_front_desk_cannot_read_observations(self, client, front_desk_token):
        response = client.get(
            "/api/v1/patients/patient-123/observations",
            headers={"Authorization": f"Bearer {front_desk_token}"},
        )
        assert response.status_code == 403

    def test_patient_cannot_access_other_patient(self, client, patient_token_for_123):
        response = client.get(
            "/api/v1/patients/patient-456/observations",
            headers={"Authorization": f"Bearer {patient_token_for_123}"},
        )
        assert response.status_code == 403

    def test_physician_without_relationship_gets_limited_view(
        self, client, physician_token_unassigned
    ):
        """Physician not on care team sees demographics only, not full chart."""
        response = client.get(
            "/api/v1/patients/unassigned-patient/observations",
            headers={"Authorization": f"Bearer {physician_token_unassigned}"},
        )
        assert response.status_code == 403

    def test_practice_admin_cannot_access_other_tenant(
        self, client, practice_admin_tenant_a_token
    ):
        response = client.get(
            "/api/v1/tenants/tenant-b/users",
            headers={"Authorization": f"Bearer {practice_admin_tenant_a_token}"},
        )
        assert response.status_code == 403
```

### OWASP Authentication Testing Applied to Healthcare

Based on the OWASP Testing Guide (WSTG-ATHN), adapted for healthcare:

1. **WSTG-ATHN-01 (Credentials Transport)**: Verify all auth endpoints use TLS 1.2+ only. No HTTP fallback. HSTS enforced.
2. **WSTG-ATHN-02 (Default Credentials)**: No default admin accounts. First-run setup wizard requires unique credentials.
3. **WSTG-ATHN-03 (Account Lockout)**: Lock account after 5 failed attempts for 15 minutes. Notify user via email. Do not reveal whether the account exists (prevent enumeration).
4. **WSTG-ATHN-04 (Auth Schema Bypass)**: Test all API endpoints without authentication. Test with tokens from different tenants. Test with revoked tokens.
5. **WSTG-ATHN-05 (Credential Recovery)**: Password reset uses time-limited token (15 minutes). Reset link is single-use. Old password is not disclosed. Notify user of password change via separate channel.
6. **WSTG-ATHN-06 (Browser Cache)**: Auth pages set `Cache-Control: no-store`. Session tokens not stored in localStorage (use httpOnly cookies or in-memory only).
7. **WSTG-ATHN-07 (Password Policy)**: Minimum 12 characters. Check against breached password databases (Have I Been Pwned API via k-anonymity). No maximum length (up to 128 characters). No arbitrary composition rules (NIST 800-63B guidance).
8. **WSTG-ATHN-08 (Security Questions)**: Not used. Security questions are weak authenticators and are not suitable for healthcare.
9. **WSTG-ATHN-09 (Password Change)**: Require current password. Invalidate all other sessions. Log the event in audit trail.
10. **WSTG-ATHN-10 (MFA Bypass)**: Verify MFA cannot be skipped by manipulating request flow. Verify MFA challenge is required on every new session (not just first login). Test that backup codes are single-use.

---

## Related Notes

- [[ADR-002-multi-tenancy-schema-per-tenant]] -- Tenant isolation strategy at the database level
- [[HIPAA-Deep-Dive]] -- Full HIPAA regulatory requirements and compliance mapping
- [[FHIR-R4-Deep-Dive]] -- FHIR resource model and API specifications
- [[System-Architecture-Overview]] -- Overall MedOS architecture and service boundaries
- [[AWS-HIPAA-Infrastructure]] -- AWS infrastructure configured for HIPAA compliance
