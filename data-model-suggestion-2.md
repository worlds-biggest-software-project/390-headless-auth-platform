# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Headless Auth Platform · Created: 2026-05-26

## Philosophy

The hybrid approach consolidates tenant-level configuration (applications, identity providers, security policies, RBAC roles) into JSONB columns on the tenant row, while keeping security-critical entities (identities, credentials, sessions) relational. The tenant row becomes a self-contained configuration document: reading one row gives the complete security policy, all registered applications, all federated identity providers, and the RBAC role definitions.

This design recognises that auth platform configuration is read-heavy and tenant-scoped: during a login flow, the platform needs the tenant's password policy, MFA requirements, registered applications, and identity provider settings as a single unit. Rather than JOINing across 4+ tables, the login flow reads one tenant row and has the complete policy context. Identities, credentials, and sessions remain relational because they are security-critical and benefit from database-level constraints — unique email enforcement, foreign key cascades on identity deletion, and indexed session lookups for revocation.

The trade-off is that adding a new application or updating an IdP configuration requires a JSONB patch on the tenant row. Since these operations happen infrequently (minutes to hours between changes) and are always initiated by admins, this is acceptable.

**Best for:** Rapid MVP development where tenant self-service configuration and fast login flows matter more than fine-grained cross-tenant queries on application or IdP configuration.

**Trade-offs:**
- Pro: Complete tenant configuration in one row — single read for login flow context
- Pro: Adding new IdP types or application settings requires no schema migration
- Pro: Fewer tables (7) means simpler operational surface
- Pro: Security-critical entities (identities, credentials, sessions) retain full relational integrity
- Con: Cross-tenant queries on specific applications or IdPs require JSONB path queries
- Con: No foreign key enforcement between embedded application IDs and sessions referencing them
- Con: Large JSONB columns on tenants with many applications and IdPs
- Con: Concurrent admin operations on the same tenant require optimistic locking

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenID Connect Core 1.0 | Application objects in tenants.applications_json store OIDC client configuration |
| OAuth 2.0 (RFC 6749) | Application grant types, redirect URIs, PKCE settings embedded in applications_json |
| RFC 9700 | PKCE enforcement, exact redirect URI matching configured per-application |
| SAML 2.0 | IdP objects in tenants.identity_providers_json store SAML metadata and certificates |
| WebAuthn Level 2 | Credentials table stores WebAuthn-specific fields per W3C spec |
| FIDO2 / CTAP2 | Passkey support with resident credential tracking |
| JWT (RFC 7519) | Session jti for revocation; JWKS configuration on tenant |
| SCIM 2.0 | IdP objects support SCIM provisioning configuration |
| NIST SP 800-63-4 | Sessions track AAL; tenant security policies enforce per-AAL requirements |
| OWASP | Password hashing, session management, account lockout in security_json |
| GDPR | Identity consent tracking; audit log for data access compliance |

---

## Tenant Configuration

### tenants

```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','cancelled')),

  -- Applications (OIDC/OAuth2/SAML clients)
  applications_json JSONB NOT NULL DEFAULT '[]',
  -- Example applications_json:
  -- [
  --   {
  --     "id": "uuid", "name": "My Web App",
  --     "client_id": "app_abc123",
  --     "client_secret_hash": "sha256...",
  --     "type": "spa",
  --     "redirect_uris": ["https://app.example.com/callback"],
  --     "post_logout_redirect_uris": ["https://app.example.com"],
  --     "allowed_origins": ["https://app.example.com"],
  --     "grant_types": ["authorization_code"],
  --     "response_types": ["code"],
  --     "scopes": ["openid","profile","email"],
  --     "pkce_required": true,
  --     "access_token_lifetime_seconds": 3600,
  --     "id_token_lifetime_seconds": 3600,
  --     "refresh_token_enabled": true,
  --     "custom_claims": {"org_id": "{{identity.metadata.org_id}}"},
  --     "is_active": true
  --   },
  --   {
  --     "id": "uuid", "name": "Enterprise SAML SP",
  --     "client_id": "app_saml_xyz",
  --     "type": "saml_sp",
  --     "saml": {
  --       "acs_url": "https://enterprise.example.com/saml/acs",
  --       "entity_id": "urn:enterprise:saml:sp",
  --       "name_id_format": "email"
  --     },
  --     "is_active": true
  --   }
  -- ]

  -- Identity providers (social, SAML, OIDC, LDAP)
  identity_providers_json JSONB NOT NULL DEFAULT '[]',
  -- Example identity_providers_json:
  -- [
  --   {
  --     "id": "uuid", "name": "Google",
  --     "type": "social_google",
  --     "client_id": "google-client-id",
  --     "client_secret_ref": "vault://idp/google",
  --     "scopes": ["openid","email","profile"],
  --     "attribute_mapping": {"email": "email", "name": "name"},
  --     "is_active": true
  --   },
  --   {
  --     "id": "uuid", "name": "Okta SAML",
  --     "type": "saml",
  --     "saml": {
  --       "sso_url": "https://company.okta.com/app/sso/saml",
  --       "entity_id": "urn:okta:entity",
  --       "certificate": "MIID...",
  --       "name_id_format": "email"
  --     },
  --     "verified_domains": ["company.com"],
  --     "jit_provisioning": true,
  --     "scim": {
  --       "enabled": true,
  --       "endpoint": "https://auth.example.com/scim/v2",
  --       "bearer_token_ref": "vault://scim/okta"
  --     },
  --     "is_active": true
  --   },
  --   {
  --     "id": "uuid", "name": "Corporate LDAP",
  --     "type": "ldap",
  --     "ldap": {
  --       "url": "ldaps://ldap.company.com:636",
  --       "bind_dn": "cn=svc-auth,dc=company,dc=com",
  --       "bind_password_ref": "vault://ldap/bind",
  --       "base_dn": "ou=users,dc=company,dc=com",
  --       "user_filter": "(uid={{username}})",
  --       "attribute_mapping": {"email": "mail", "name": "displayName"}
  --     },
  --     "is_active": true
  --   }
  -- ]

  -- RBAC roles and permissions
  roles_json JSONB NOT NULL DEFAULT '[]',
  -- Example roles_json:
  -- [
  --   {
  --     "id": "uuid", "name": "admin",
  --     "permissions": ["users:read","users:write","settings:manage","billing:manage"],
  --     "is_default": false
  --   },
  --   {
  --     "id": "uuid", "name": "member",
  --     "permissions": ["users:read"],
  --     "is_default": true
  --   }
  -- ]

  -- Security policies
  security_json JSONB NOT NULL DEFAULT '{}',
  -- Example security_json:
  -- {
  --   "mfa_policy": "required",
  --   "password_policy": {
  --     "min_length": 12, "require_uppercase": true,
  --     "require_number": true, "require_special": true,
  --     "algorithm": "bcrypt", "bcrypt_cost": 12,
  --     "max_age_days": null, "history_count": 0
  --   },
  --   "session": {
  --     "lifetime_seconds": 86400,
  --     "refresh_token_lifetime_seconds": 2592000,
  --     "refresh_token_rotation": true,
  --     "idle_timeout_seconds": 3600
  --   },
  --   "lockout": {
  --     "max_failed_attempts": 10,
  --     "duration_seconds": 900
  --   },
  --   "risk_scoring_enabled": true,
  --   "breached_password_detection": true
  -- }

  -- JWKS configuration
  jwks_json JSONB NOT NULL DEFAULT '{}',
  -- Example jwks_json:
  -- {
  --   "algorithm": "RS256",
  --   "private_key_ref": "vault://jwks/tenant-abc",
  --   "key_rotation_interval_days": 90
  -- }

  -- Webhooks
  webhooks_json JSONB NOT NULL DEFAULT '[]',
  -- Example webhooks_json:
  -- [
  --   {
  --     "id": "uuid", "url": "https://api.example.com/auth/events",
  --     "event_types": ["user.created","user.login","user.mfa_enrolled"],
  --     "signing_secret": "whsec_...",
  --     "is_active": true
  --   }
  -- ]

  -- Branding
  branding_json JSONB NOT NULL DEFAULT '{}',

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_apps ON tenants USING GIN (applications_json);
CREATE INDEX idx_tenants_idps ON tenants USING GIN (identity_providers_json);
```

**Application lookup by client_id:**

```sql
SELECT id, applications_json
FROM tenants
WHERE applications_json @> '[{"client_id": "app_abc123"}]'::jsonb;
```

---

## Identity & Credentials

### identities

```sql
CREATE TABLE identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  email TEXT,
  email_verified BOOLEAN NOT NULL DEFAULT false,
  phone TEXT,
  phone_verified BOOLEAN NOT NULL DEFAULT false,
  name TEXT,
  given_name TEXT,
  family_name TEXT,
  picture_url TEXT,
  -- Status
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','locked','deleted')),
  locked_until TIMESTAMPTZ,
  failed_login_attempts INTEGER NOT NULL DEFAULT 0,
  -- MFA
  mfa_enabled BOOLEAN NOT NULL DEFAULT false,
  mfa_preferred_method TEXT
    CHECK (mfa_preferred_method IN ('totp','sms','email','webauthn')),
  -- RBAC (role IDs reference roles_json on tenant)
  role_ids TEXT[] NOT NULL DEFAULT '{}',
  -- Metadata and claims
  custom_claims JSONB NOT NULL DEFAULT '{}',
  metadata JSONB NOT NULL DEFAULT '{}',
  -- Federation
  external_id TEXT,
  provisioned_by_idp_id TEXT,
  -- Consent grants (embedded per application)
  consent_grants_json JSONB NOT NULL DEFAULT '{}',
  -- Example consent_grants_json:
  -- {
  --   "app_abc123": {
  --     "scopes": ["openid","profile","email"],
  --     "granted_at": "2026-05-26T10:00:00Z"
  --   }
  -- }
  -- Timestamps
  last_login_at TIMESTAMPTZ,
  password_changed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, email)
);

CREATE INDEX idx_identities_tenant ON identities(tenant_id);
CREATE INDEX idx_identities_email ON identities(tenant_id, email)
  WHERE email IS NOT NULL;
CREATE INDEX idx_identities_external ON identities(tenant_id, external_id)
  WHERE external_id IS NOT NULL;
```

### credentials

```sql
CREATE TABLE credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL,
  credential_type TEXT NOT NULL
    CHECK (credential_type IN (
      'password','webauthn','totp','social','email_otp','sms_otp'
    )),

  -- Credential data (polymorphic by type)
  credential_data JSONB NOT NULL,
  -- Example credential_data (password):
  -- {
  --   "hash": "$2b$12$...",
  --   "algorithm": "bcrypt",
  --   "needs_rehash": false
  -- }
  -- Example credential_data (webauthn):
  -- {
  --   "credential_id": "base64...",
  --   "public_key": "base64...",
  --   "sign_count": 42,
  --   "attestation_type": "none",
  --   "transports": ["internal","hybrid"],
  --   "aaguid": "adce0002-...",
  --   "device_name": "iPhone Passkey",
  --   "is_resident": true,
  --   "user_verified": true
  -- }
  -- Example credential_data (totp):
  -- {
  --   "secret_encrypted": "enc:...",
  --   "algorithm": "SHA1",
  --   "digits": 6, "period": 30,
  --   "verified": true
  -- }
  -- Example credential_data (social):
  -- {
  --   "provider_id": "uuid",
  --   "provider_user_id": "google-12345",
  --   "access_token_encrypted": "enc:...",
  --   "refresh_token_encrypted": "enc:...",
  --   "token_expires_at": "2026-06-26T10:00:00Z"
  -- }

  -- Common lifecycle
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','pending_verification','revoked','expired')),
  last_used_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credentials_identity ON credentials(identity_id);
CREATE INDEX idx_credentials_type ON credentials(identity_id, credential_type);
CREATE INDEX idx_credentials_data ON credentials USING GIN (credential_data);
```

### sessions

```sql
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL,
  application_id TEXT,
  jti TEXT NOT NULL UNIQUE,
  refresh_token_hash TEXT,
  refresh_token_family TEXT,

  -- Authentication context
  auth_json JSONB NOT NULL DEFAULT '{}',
  -- Example auth_json:
  -- {
  --   "method": "password",
  --   "mfa_completed": true,
  --   "mfa_method": "totp",
  --   "aal": 2,
  --   "credential_id": "uuid"
  -- }

  -- Client context
  context_json JSONB NOT NULL DEFAULT '{}',
  -- Example context_json:
  -- {
  --   "ip_address": "203.0.113.1",
  --   "user_agent": "Mozilla/5.0...",
  --   "device_fingerprint": "fp_abc123",
  --   "geo_country": "US",
  --   "geo_city": "San Francisco"
  -- }

  -- Lifecycle
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','refreshed','revoked','expired')),
  expires_at TIMESTAMPTZ NOT NULL,
  last_activity_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  revoked_at TIMESTAMPTZ,
  revocation_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_identity ON sessions(identity_id, status)
  WHERE status = 'active';
CREATE INDEX idx_sessions_jti ON sessions(jti);
CREATE INDEX idx_sessions_refresh ON sessions(refresh_token_hash)
  WHERE refresh_token_hash IS NOT NULL;
CREATE INDEX idx_sessions_expiry ON sessions(expires_at)
  WHERE status = 'active';
```

---

## Operations

### login_attempts

```sql
CREATE TABLE login_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  identity_id UUID,
  email TEXT,
  ip_address INET NOT NULL,
  success BOOLEAN NOT NULL,
  auth_method TEXT NOT NULL,
  failure_reason TEXT,
  risk_json JSONB,
  -- Example risk_json:
  -- {
  --   "score": 0.85,
  --   "signals": {
  --     "new_device": true, "unusual_location": false,
  --     "velocity_exceeded": false, "known_vpn": true,
  --     "breached_password": false
  --   }
  -- }
  user_agent TEXT,
  geo_country TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_login_attempts_identity ON login_attempts(identity_id, created_at DESC);
CREATE INDEX idx_login_attempts_ip ON login_attempts(ip_address, created_at DESC);
CREATE INDEX idx_login_attempts_email ON login_attempts(tenant_id, email, created_at DESC);
```

### ai_suggestions

```sql
CREATE TABLE ai_suggestions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  suggestion_type TEXT NOT NULL
    CHECK (suggestion_type IN (
      'adaptive_mfa','anomaly_detection','credential_compromise',
      'phishing_detection','session_anomaly','risk_scoring',
      'policy_recommendation','compliance_gap','password_strength',
      'mfa_adoption'
    )),
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  confidence NUMERIC(3,2) NOT NULL CHECK (confidence BETWEEN 0 AND 1),
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  suggestion_data JSONB NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','accepted','dismissed','expired')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id, status);
```

### audit_log

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  actor_id TEXT,
  actor_type TEXT NOT NULL
    CHECK (actor_type IN ('identity','admin','system','api_key','scim','ai')),
  action TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  changes JSONB,
  ip_address INET,
  user_agent TEXT,
  session_id UUID,
  risk_score NUMERIC(3,2),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id, created_at DESC);
CREATE INDEX idx_audit_log_actor ON audit_log(actor_id, created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant Configuration | 1 | Embeds applications, IdPs, roles, security policies, JWKS, webhooks |
| Identity & Credentials | 2 | identities (with embedded consent/roles), credentials (JSONB credential_data) |
| Sessions | 1 | sessions (with embedded auth/context JSONB) |
| Security | 1 | login_attempts (partitioned) |
| Operations | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **7** | 2 partitioned tables |

---

## Key Design Decisions

1. **Tenant as a self-contained configuration document** — Applications, identity providers, RBAC roles, security policies, JWKS config, and webhooks are all embedded in the tenant row. The login flow reads one tenant row and has the complete policy context: password requirements, MFA enforcement level, available social login providers, and SAML IdP configuration.

2. **Credentials with JSONB credential_data** — Rather than type-specific nullable columns, each credential stores its type-specific data in a credential_data JSONB column. A password credential stores hash/algorithm/needs_rehash; a WebAuthn credential stores credential_id/public_key/sign_count per W3C spec. This keeps the credentials table clean while supporting any future credential type without column additions.

3. **Identities remain relational** — Despite the JSONB-heavy design, identities stay in their own indexed table because email uniqueness enforcement (UNIQUE constraint on tenant_id + email) and foreign key cascades (credential deletion on identity deletion) are too important to embed. The unique constraint prevents duplicate account creation — a critical security invariant.

4. **Sessions remain relational** — Sessions need indexed lookups by jti (for revocation), refresh_token_hash (for rotation), and expiry (for cleanup). The status column enables efficient "active sessions for identity X" queries. Auth and client context are embedded as JSONB because they are written once and read for display only.

5. **Consent grants embedded in identity** — OAuth2 consent records are stored in consent_grants_json on the identity row, keyed by client_id. This avoids a separate table and makes consent checking a single identity read during token issuance. The trade-off: cross-identity consent queries ("which identities consented to app X?") require JSONB path queries.

6. **Role assignments as TEXT array on identity** — Rather than a junction table, role_ids on the identity row references role IDs from the tenant's roles_json. Permission checks are fast: read identity → get role_ids → look up roles in tenant → check permissions. The trade-off: no foreign key enforcement on role references.

7. **Login attempts remain relational and partitioned** — Login attempts are the highest-volume security table and need fast indexed queries by IP, email, and identity for rate limiting and anomaly detection. Risk scoring data is embedded as JSONB. Partitioning enables retention management (e.g., 90-day rolling window).

8. **WebAuthn credential_id lookup** — The GIN index on credential_data enables JSONB containment queries for WebAuthn credential_id lookup during passkey assertion. The query `WHERE credential_data @> '{"credential_id": "base64..."}'` finds the matching credential efficiently.

9. **SAML and OIDC IdP configuration as embedded JSONB** — Enterprise IdP configuration (SAML metadata URL, SSO URL, certificates, SCIM settings) is embedded in the tenant's identity_providers_json. This makes tenant provisioning a single row operation and supports the self-service SSO portal pattern where enterprise customers configure their own IdPs through the admin API.

10. **Security policies as structured JSONB** — Password policy (min length, complexity, algorithm, bcrypt cost), session policy (lifetime, idle timeout, refresh rotation), and lockout policy (max attempts, duration) are stored in security_json. This enables per-tenant security customization without schema changes and supports the compliance template pattern where pre-built profiles (SOC 2, HIPAA, PCI-DSS) set all policies at once.
