# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Headless Auth Platform · Created: 2026-05-26

## Philosophy

Every identity concept — tenant, application, identity provider, identity, credential, session, role, consent grant — lives in its own table with strict foreign key relationships. The data model is designed for a headless auth platform where security invariants (credential storage, session lifecycle, MFA enforcement) must be expressed through database-level constraints rather than application-level assumptions.

The normalized design is particularly well-suited for identity management because the domain has deeply interrelated entities with independent lifecycles. An identity has multiple credentials (password, passkeys, TOTP secrets, social connections), each of which can be enrolled, verified, rotated, or revoked independently. Sessions reference identities and the authentication method used, enabling session-level MFA enforcement. Roles and permissions form a classic RBAC graph that is best expressed through relational tables and foreign keys.

Credential storage is the most security-critical aspect of the model. Passwords use bcrypt/Argon2 hashes in a dedicated column; WebAuthn credentials store the public key and attestation data; TOTP secrets are encrypted at rest. Each credential type has type-specific fields alongside a common lifecycle (status, created_at, last_used_at). This polymorphic-per-row approach keeps credential queries fast via the type discriminator index.

**Best for:** Teams building a production identity engine with strict security requirements, multi-credential support, RBAC authorisation, and compliance needs that demand fine-grained audit trails and relational integrity.

**Trade-offs:**
- Pro: Full referential integrity across identities, credentials, sessions, roles, and applications
- Pro: Each credential type has its own lifecycle — passkeys, passwords, and TOTP are independently manageable
- Pro: Session table supports both JWT and server-side session validation patterns
- Pro: RBAC with permission-level granularity through junction tables
- Con: Login flow requires multiple queries (identity lookup → credential verification → MFA check → session creation)
- Con: 14 tables increases migration complexity
- Con: Login_attempts table grows rapidly under credential stuffing attacks
- Con: Consent grants as separate table adds JOIN overhead for token issuance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenID Connect Core 1.0 | Applications table stores OIDC client configuration; ID tokens carry custom claims from identities |
| OAuth 2.0 (RFC 6749) | Applications table models OAuth2 clients; consent_grants track scope authorisation |
| OAuth 2.0 PKCE (RFC 7636) | Applications store pkce_required flag; authorization_code_challenges in verification_tokens |
| RFC 9700 (OAuth Security BCP) | PKCE mandatory, exact redirect URI matching, implicit grant disabled |
| SAML 2.0 | Identity providers table stores SAML metadata, certificates, and SSO endpoints |
| WebAuthn Level 2 (W3C) | Credentials table stores public key, credential_id, attestation, sign_count for passkey verification |
| FIDO2 / CTAP2 | Credentials model supports resident credentials and user verification enforcement |
| JWT (RFC 7519) | Sessions store JWT jti for revocation; refresh_token_hash for rotation |
| SCIM 2.0 (RFC 7643/7644) | Identity providers support SCIM provisioning with attribute mapping |
| NIST SP 800-63-4 | MFA enforcement per AAL level; credential types mapped to AAL requirements |
| OWASP Authentication Cheat Sheet | Password hashing (bcrypt/Argon2), account lockout, session fixation prevention |
| GDPR | Identities track consent; audit_log records all data access; right to erasure supported |

---

## Tenant & Application Management

### tenants

```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','cancelled')),
  -- Security policies
  mfa_policy TEXT NOT NULL DEFAULT 'optional'
    CHECK (mfa_policy IN ('disabled','optional','required','adaptive')),
  password_policy JSONB NOT NULL DEFAULT '{}',
  -- Example password_policy:
  -- {
  --   "min_length": 12, "require_uppercase": true,
  --   "require_number": true, "require_special": true,
  --   "bcrypt_cost": 12, "algorithm": "bcrypt",
  --   "max_age_days": null, "history_count": 0
  -- }
  session_lifetime_seconds INTEGER NOT NULL DEFAULT 86400,
  refresh_token_lifetime_seconds INTEGER NOT NULL DEFAULT 2592000,
  refresh_token_rotation BOOLEAN NOT NULL DEFAULT true,
  max_failed_login_attempts INTEGER NOT NULL DEFAULT 10,
  lockout_duration_seconds INTEGER NOT NULL DEFAULT 900,
  -- Branding
  branding JSONB NOT NULL DEFAULT '{}',
  -- JWKS
  jwks_private_key_ref TEXT NOT NULL,
  jwks_algorithm TEXT NOT NULL DEFAULT 'RS256'
    CHECK (jwks_algorithm IN ('RS256','ES256','EdDSA')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### applications

```sql
CREATE TABLE applications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  client_id TEXT NOT NULL UNIQUE,
  client_secret_hash TEXT,
  application_type TEXT NOT NULL
    CHECK (application_type IN ('web','spa','native','machine_to_machine','saml_sp')),
  -- OAuth2 / OIDC configuration
  redirect_uris TEXT[] NOT NULL DEFAULT '{}',
  post_logout_redirect_uris TEXT[],
  allowed_origins TEXT[],
  grant_types TEXT[] NOT NULL DEFAULT '{authorization_code}',
  response_types TEXT[] NOT NULL DEFAULT '{code}',
  scopes TEXT[] NOT NULL DEFAULT '{openid,profile,email}',
  pkce_required BOOLEAN NOT NULL DEFAULT true,
  -- Token configuration
  access_token_lifetime_seconds INTEGER NOT NULL DEFAULT 3600,
  id_token_lifetime_seconds INTEGER NOT NULL DEFAULT 3600,
  refresh_token_enabled BOOLEAN NOT NULL DEFAULT true,
  -- Custom claims
  custom_claims JSONB NOT NULL DEFAULT '{}',
  -- SAML SP configuration (when application_type = 'saml_sp')
  saml_acs_url TEXT,
  saml_entity_id TEXT,
  saml_name_id_format TEXT
    CHECK (saml_name_id_format IN ('email','persistent','transient','unspecified')),
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_applications_tenant ON applications(tenant_id);
CREATE INDEX idx_applications_client ON applications(client_id);
```

---

## Identity Provider Federation

### identity_providers

```sql
CREATE TABLE identity_providers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  provider_type TEXT NOT NULL
    CHECK (provider_type IN (
      'social_google','social_github','social_apple','social_microsoft',
      'social_facebook','social_linkedin','social_discord','social_custom',
      'oidc','saml','ldap'
    )),
  is_active BOOLEAN NOT NULL DEFAULT true,
  -- Social / OIDC configuration
  client_id TEXT,
  client_secret_ref TEXT,
  authorization_url TEXT,
  token_url TEXT,
  userinfo_url TEXT,
  issuer TEXT,
  scopes TEXT[],
  -- SAML IdP configuration
  saml_metadata_url TEXT,
  saml_sso_url TEXT,
  saml_slo_url TEXT,
  saml_certificate TEXT,
  saml_entity_id TEXT,
  saml_name_id_format TEXT,
  -- LDAP configuration
  ldap_url TEXT,
  ldap_bind_dn TEXT,
  ldap_bind_password_ref TEXT,
  ldap_base_dn TEXT,
  ldap_user_filter TEXT,
  ldap_attribute_mapping JSONB,
  -- SCIM provisioning
  scim_enabled BOOLEAN NOT NULL DEFAULT false,
  scim_endpoint TEXT,
  scim_bearer_token_ref TEXT,
  -- Domain verification for JIT provisioning
  verified_domains TEXT[],
  jit_provisioning BOOLEAN NOT NULL DEFAULT false,
  -- Attribute mapping
  attribute_mapping JSONB NOT NULL DEFAULT '{}',
  -- Example attribute_mapping:
  -- {
  --   "email": "email",
  --   "name": "name",
  --   "given_name": "given_name",
  --   "family_name": "family_name",
  --   "picture": "picture"
  -- }
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_identity_providers_tenant ON identity_providers(tenant_id);
CREATE INDEX idx_identity_providers_domains ON identity_providers USING GIN (verified_domains)
  WHERE verified_domains IS NOT NULL;
```

---

## Identity & Credentials

### identities

```sql
CREATE TABLE identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
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
  -- Metadata
  custom_claims JSONB NOT NULL DEFAULT '{}',
  metadata JSONB NOT NULL DEFAULT '{}',
  -- External identity (for federated/SCIM-provisioned users)
  external_id TEXT,
  provisioned_by UUID REFERENCES identity_providers(id),
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
CREATE INDEX idx_identities_status ON identities(tenant_id, status)
  WHERE status != 'active';
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
  -- Password fields
  password_hash TEXT,
  password_algorithm TEXT
    CHECK (password_algorithm IN ('bcrypt','argon2id','scrypt','pbkdf2')),
  password_needs_rehash BOOLEAN NOT NULL DEFAULT false,
  -- WebAuthn / Passkey fields (W3C WebAuthn Level 2)
  webauthn_credential_id BYTEA,
  webauthn_public_key BYTEA,
  webauthn_sign_count BIGINT,
  webauthn_attestation_type TEXT,
  webauthn_transports TEXT[],
  webauthn_aaguid TEXT,
  webauthn_device_name TEXT,
  webauthn_is_resident BOOLEAN,
  webauthn_user_verified BOOLEAN,
  -- TOTP fields
  totp_secret_encrypted TEXT,
  totp_algorithm TEXT DEFAULT 'SHA1',
  totp_digits INTEGER DEFAULT 6,
  totp_period INTEGER DEFAULT 30,
  totp_verified BOOLEAN DEFAULT false,
  -- Social connection fields
  social_provider_id UUID REFERENCES identity_providers(id),
  social_provider_user_id TEXT,
  social_access_token_encrypted TEXT,
  social_refresh_token_encrypted TEXT,
  social_token_expires_at TIMESTAMPTZ,
  -- Common lifecycle
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','pending_verification','revoked','expired')),
  last_used_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credentials_identity ON credentials(identity_id);
CREATE INDEX idx_credentials_type ON credentials(identity_id, credential_type);
CREATE INDEX idx_credentials_webauthn ON credentials(webauthn_credential_id)
  WHERE webauthn_credential_id IS NOT NULL;
CREATE INDEX idx_credentials_social ON credentials(social_provider_id, social_provider_user_id)
  WHERE social_provider_user_id IS NOT NULL;
```

---

## Sessions & Tokens

### sessions

```sql
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL,
  application_id UUID REFERENCES applications(id),
  -- Token tracking
  jti TEXT NOT NULL UNIQUE,
  refresh_token_hash TEXT,
  refresh_token_family TEXT,
  -- Authentication context
  auth_method TEXT NOT NULL
    CHECK (auth_method IN (
      'password','webauthn','social','saml','oidc','ldap',
      'email_otp','sms_otp','magic_link','token_exchange'
    )),
  mfa_completed BOOLEAN NOT NULL DEFAULT false,
  mfa_method TEXT,
  aal INTEGER NOT NULL DEFAULT 1
    CHECK (aal IN (1, 2, 3)),
  -- Session metadata
  ip_address INET,
  user_agent TEXT,
  device_fingerprint TEXT,
  geo_country TEXT,
  geo_city TEXT,
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

### consent_grants

```sql
CREATE TABLE consent_grants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  application_id UUID NOT NULL REFERENCES applications(id),
  tenant_id UUID NOT NULL,
  scopes TEXT[] NOT NULL,
  granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ,
  revoked_at TIMESTAMPTZ,
  UNIQUE(identity_id, application_id)
);

CREATE INDEX idx_consent_grants_identity ON consent_grants(identity_id);
```

---

## RBAC

### roles

```sql
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  description TEXT,
  is_default BOOLEAN NOT NULL DEFAULT false,
  permissions TEXT[] NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, name)
);

CREATE INDEX idx_roles_tenant ON roles(tenant_id);
```

### role_assignments

```sql
CREATE TABLE role_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id) ON DELETE CASCADE,
  role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL,
  assigned_by UUID,
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ,
  UNIQUE(identity_id, role_id)
);

CREATE INDEX idx_role_assignments_identity ON role_assignments(identity_id);
CREATE INDEX idx_role_assignments_role ON role_assignments(role_id);
```

---

## Verification & Security

### verification_tokens

```sql
CREATE TABLE verification_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID REFERENCES identities(id),
  tenant_id UUID NOT NULL,
  token_type TEXT NOT NULL
    CHECK (token_type IN (
      'email_verification','password_reset','email_otp',
      'sms_otp','magic_link','mfa_challenge',
      'authorization_code','device_code'
    )),
  token_hash TEXT NOT NULL,
  -- PKCE (for authorization_code type)
  code_challenge TEXT,
  code_challenge_method TEXT DEFAULT 'S256',
  -- Context
  redirect_uri TEXT,
  application_id UUID,
  scopes TEXT[],
  nonce TEXT,
  -- Lifecycle
  expires_at TIMESTAMPTZ NOT NULL,
  used_at TIMESTAMPTZ,
  attempts INTEGER NOT NULL DEFAULT 0,
  max_attempts INTEGER NOT NULL DEFAULT 3,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_verification_tokens_hash ON verification_tokens(token_hash);
CREATE INDEX idx_verification_tokens_expiry ON verification_tokens(expires_at)
  WHERE used_at IS NULL;
```

### login_attempts

```sql
CREATE TABLE login_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  identity_id UUID,
  email TEXT,
  ip_address INET NOT NULL,
  user_agent TEXT,
  device_fingerprint TEXT,
  geo_country TEXT,
  geo_city TEXT,
  auth_method TEXT NOT NULL,
  success BOOLEAN NOT NULL,
  failure_reason TEXT,
  -- Risk scoring
  risk_score NUMERIC(3,2),
  risk_signals JSONB,
  -- Example risk_signals:
  -- {
  --   "new_device": true, "unusual_location": false,
  --   "velocity_exceeded": false, "known_vpn": true,
  --   "credential_stuffing_pattern": false
  -- }
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_login_attempts_identity ON login_attempts(identity_id, created_at DESC);
CREATE INDEX idx_login_attempts_ip ON login_attempts(ip_address, created_at DESC);
CREATE INDEX idx_login_attempts_email ON login_attempts(tenant_id, email, created_at DESC);
```

---

## Operations

### webhooks

```sql
CREATE TABLE webhooks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  url TEXT NOT NULL,
  event_types TEXT[] NOT NULL,
  signing_secret TEXT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT true,
  failure_count INTEGER NOT NULL DEFAULT 0,
  last_failure_at TIMESTAMPTZ,
  last_success_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_tenant ON webhooks(tenant_id);
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
  entity_id UUID NOT NULL,
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
  actor_id UUID,
  actor_type TEXT NOT NULL
    CHECK (actor_type IN ('identity','admin','system','api_key','scim','ai')),
  action TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id UUID NOT NULL,
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
| Tenant & Applications | 2 | tenants, applications |
| Federation | 1 | identity_providers (social, SAML, OIDC, LDAP, SCIM) |
| Identity & Credentials | 2 | identities, credentials (polymorphic per type) |
| Sessions & Consent | 2 | sessions, consent_grants |
| RBAC | 2 | roles, role_assignments |
| Verification & Security | 2 | verification_tokens, login_attempts (partitioned) |
| Operations | 3 | webhooks, ai_suggestions, audit_log (partitioned) |
| **Total** | **14** | 2 partitioned tables |

---

## Key Design Decisions

1. **Credentials as a polymorphic table** — Passwords, WebAuthn/passkey credentials, TOTP secrets, and social connections all share one table with a type discriminator. Each row stores only the fields relevant to its type (nullable columns for other types). This enables "find all credentials for identity X" as a single query and supports the multi-credential model where a user has a password AND a passkey AND a TOTP enrollment simultaneously.

2. **WebAuthn fields aligned with W3C WebAuthn Level 2** — The credentials table stores credential_id, public_key, sign_count, attestation_type, transports, aaguid, and is_resident. The sign_count field is updated on every successful assertion to detect cloned authenticators. The is_resident flag distinguishes discoverable credentials (passkeys) from server-side credentials, enabling the passkey-first flow.

3. **Password migration via needs_rehash flag** — The password_needs_rehash boolean enables transparent credential migration from legacy systems. When importing bcrypt hashes from an old system, set password_algorithm to the legacy algorithm and needs_rehash to true. On next successful login, the platform re-hashes with the tenant's current algorithm and clears the flag.

4. **Sessions with AAL tracking** — Each session records its Authentication Assurance Level (AAL) per NIST SP 800-63-4. AAL1 = single factor (password or passkey). AAL2 = two factors (password + TOTP). AAL3 = phishing-resistant hardware authenticator. Step-up authentication creates a new session at a higher AAL without invalidating the existing session.

5. **Refresh token rotation via family tracking** — The refresh_token_family column groups a chain of rotated refresh tokens. If a previously-used refresh token from the same family is presented (replay attack), all sessions in that family are revoked. This implements the refresh token rotation pattern recommended by RFC 9700.

6. **Login attempts as a partitioned security table** — Every login attempt (successful or failed) is recorded with device fingerprint, geolocation, and AI risk scoring. This table supports rate limiting (count recent failures per IP/email), credential stuffing detection (high-velocity failures across identities), and anomaly detection (login from unusual location). Partitioned by time for retention management.

7. **Identity providers as a polymorphic table** — Social, OIDC, SAML, and LDAP configurations share one table with type-specific nullable columns. This avoids creating separate tables for each federation protocol while keeping all IdP configuration in one queryable place. The verified_domains array enables domain-based JIT provisioning for enterprise SAML IdPs.

8. **Consent grants separate from sessions** — OAuth2 consent (which scopes an identity granted to which application) is tracked independently of sessions. This enables consent persistence: a user doesn't need to re-consent on every login if their grant is still valid. Consent revocation invalidates the grant without affecting active sessions.

9. **Verification tokens serve multiple purposes** — Email verification, password reset, OTP challenges, magic links, authorization codes, and device codes all share one table. The type discriminator routes token validation to the correct handler. PKCE fields (code_challenge, code_challenge_method) are only populated for authorization_code type, per RFC 7636.

10. **RBAC with permissions as TEXT array** — Roles store permissions as a TEXT array (e.g., `['users:read', 'users:write', 'billing:manage']`) rather than in a separate permissions table with a junction table. This trades queryability ("which roles have permission X?") for simplicity — permission checks are fast array containment queries, and the permission namespace is defined by convention rather than by a foreign key.
