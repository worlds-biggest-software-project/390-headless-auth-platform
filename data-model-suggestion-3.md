# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Headless Auth Platform · Created: 2026-05-26

## Philosophy

Every state change in the identity engine — identity registered, credential enrolled, session created, MFA verified, role assigned, consent granted — is captured as an immutable event in a single append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimised materialised views serve the login hot path, session validation, and compliance dashboards.

This architecture is a natural fit for identity platforms because security auditing is not an afterthought — it IS the source of truth. ZITADEL already uses event sourcing for its identity platform, and the approach has proven valuable for compliance (SOC 2, ISO 27001, HIPAA) where auditors require complete, immutable records of every authentication decision, credential change, and access grant. An event-sourced auth platform turns audit logging from a side effect into the foundation: every login attempt, MFA challenge, session creation, and password change is a first-class event.

The CQRS pattern separates the write path (append identity and credential events) from the read path (login checks `rm_identity_credentials`, session validation reads `rm_active_sessions`). Temporal queries become native: "what credentials did identity X have on date Y?" is answered by replaying events to that point. This is critical for post-incident investigation of account takeover.

**Best for:** Regulated environments (SOC 2, HIPAA, PCI-DSS) where complete audit trails, temporal identity queries, and cryptographic proof of authentication decisions are non-negotiable requirements.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every login, credential change, and access decision is a first-class event
- Pro: Temporal queries native: "what MFA was enrolled when the breach occurred?"
- Pro: Compliance auditing is trivial — the event store IS the audit log
- Pro: Read models can be rebuilt from events if a projection bug is found
- Con: Login flow depends on read model freshness (milliseconds in practice)
- Con: High event volume during credential stuffing attacks (every attempt is an event)
- Con: Credential hashes must be stored in read models for fast verification
- Con: Event schema evolution requires careful versioning — old events must remain replayable

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope follows CloudEvents spec: ce_source, ce_type, ce_specversion, ce_time |
| OpenID Connect Core 1.0 | Application lifecycle events track OIDC client configuration changes |
| OAuth 2.0 (RFC 6749) | Authorization code and token exchange events track the full OAuth flow |
| RFC 9700 | PKCE and security BCP compliance tracked in authorization events |
| SAML 2.0 | IdP configuration events track SAML metadata and certificate changes |
| WebAuthn Level 2 | Credential events store WebAuthn registration and assertion data |
| FIDO2 / CTAP2 | Passkey lifecycle events track sign_count for cloned authenticator detection |
| JWT (RFC 7519) | Session events track JTI for revocation; token exchange events |
| SCIM 2.0 | Provisioning events track SCIM-initiated identity changes |
| NIST SP 800-63-4 | AAL tracked in session events; credential events mapped to AAL requirements |
| OWASP | Login failure events enable credential stuffing and account lockout detection |
| GDPR | Identity events track consent; right-to-erasure events with compliance proof |

---

## Event Store Infrastructure

### event_store

```sql
CREATE TABLE event_store (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stream_type TEXT NOT NULL
    CHECK (stream_type IN (
      'tenant','application','identity_provider','identity',
      'credential','session','role','consent',
      'verification','login','ai','config'
    )),
  stream_id UUID NOT NULL,
  version BIGINT NOT NULL,
  event_type TEXT NOT NULL,
  event_data JSONB NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}',
  -- CloudEvents 1.0 envelope
  ce_source TEXT NOT NULL DEFAULT '/headless-auth',
  ce_specversion TEXT NOT NULL DEFAULT '1.0',
  ce_type TEXT NOT NULL,
  ce_time TIMESTAMPTZ NOT NULL DEFAULT now(),
  -- Actor
  actor_id UUID,
  actor_type TEXT NOT NULL
    CHECK (actor_type IN (
      'identity','admin','system','api_key','scim',
      'social_provider','saml_idp','ldap','risk_engine','ai'
    )),
  tenant_id UUID NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(stream_type, stream_id, version)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, created_at DESC);
CREATE INDEX idx_events_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_events_ce_type ON event_store(ce_type, ce_time DESC);
```

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
  stream_type TEXT NOT NULL,
  stream_id UUID NOT NULL,
  version BIGINT NOT NULL,
  snapshot_data JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (stream_type, stream_id)
);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
  projection_name TEXT PRIMARY KEY,
  last_event_id UUID NOT NULL,
  last_event_time TIMESTAMPTZ NOT NULL,
  events_processed BIGINT NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Catalogue

### Identity Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| identity.registered | email, name, registration_method (email/social/saml/scim) | identity, social_provider, saml_idp, scim |
| identity.email_verified | email | identity, system |
| identity.phone_verified | phone | identity, system |
| identity.profile_updated | changed_fields | identity, admin, scim |
| identity.suspended | reason | admin, system, risk_engine |
| identity.reactivated | — | admin |
| identity.locked | failed_attempts, lockout_duration_seconds | system |
| identity.unlocked | method (auto_expiry/admin) | system, admin |
| identity.deleted | deletion_type (soft/hard), gdpr_request | admin, identity |
| identity.metadata_updated | key, old_value, new_value | admin, api_key, scim |

### Credential Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| credential.password_set | algorithm, bcrypt_cost, needs_rehash | identity, admin, system |
| credential.password_changed | old_algorithm, new_algorithm | identity |
| credential.password_rehashed | old_algorithm, new_algorithm | system |
| credential.password_reset_requested | delivery_method (email) | identity |
| credential.password_reset_completed | — | identity |
| credential.webauthn_registered | credential_id, aaguid, device_name, transports, is_resident, attestation_type | identity |
| credential.webauthn_used | credential_id, sign_count, user_verified | identity |
| credential.webauthn_sign_count_anomaly | credential_id, expected_count, actual_count | system |
| credential.webauthn_revoked | credential_id, reason | identity, admin |
| credential.totp_enrolled | algorithm, digits, period | identity |
| credential.totp_verified | — | identity |
| credential.totp_used | — | identity |
| credential.totp_revoked | reason | identity, admin |
| credential.social_linked | provider_type, provider_user_id | identity, social_provider |
| credential.social_unlinked | provider_type, provider_user_id | identity |
| credential.social_token_refreshed | provider_type, new_expiry | system |
| credential.compromised_detected | source (haveibeenpwned/internal), risk_level | risk_engine, ai |

### Session Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| session.created | jti, application_id, auth_method, aal, ip_address, user_agent, geo_country | identity |
| session.mfa_completed | mfa_method, aal_upgraded_to | identity |
| session.step_up_completed | previous_aal, new_aal, triggered_by | identity |
| session.refreshed | old_jti, new_jti, rotation_detected | system |
| session.activity_recorded | ip_address, endpoint | system |
| session.anomaly_detected | anomaly_type, risk_score, signals | risk_engine, ai |
| session.revoked | reason (user_logout/admin_action/security/token_replay), revoked_family | identity, admin, system |
| session.expired | — | system |

### Login Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| login.attempted | email, ip_address, auth_method, user_agent, geo_country | identity |
| login.succeeded | identity_id, auth_method, mfa_completed, aal, session_id | identity |
| login.failed | email, failure_reason, ip_address | identity |
| login.risk_evaluated | risk_score, signals (new_device, unusual_location, velocity, vpn, breached_password) | risk_engine |
| login.mfa_challenge_issued | method (totp/sms/email/webauthn), delivery_target | system |
| login.mfa_challenge_verified | method, attempts | identity |
| login.mfa_challenge_failed | method, attempts, max_attempts | identity |
| login.account_locked | failed_attempts, lockout_duration_seconds | system |
| login.credential_stuffing_detected | ip_address, attempt_count, time_window_seconds | risk_engine, ai |
| login.brute_force_detected | identity_id, attempt_count, time_window_seconds | risk_engine |

### Application Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| application.created | name, client_id, type, redirect_uris, grant_types, scopes | admin |
| application.updated | changed_fields | admin |
| application.secret_rotated | old_prefix, new_prefix | admin |
| application.deactivated | reason | admin |
| application.saml_configured | acs_url, entity_id, name_id_format | admin |

### Identity Provider Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| idp.added | name, type, config | admin |
| idp.updated | changed_fields | admin |
| idp.saml_certificate_updated | old_expiry, new_expiry | admin |
| idp.domain_verified | domain | admin, system |
| idp.scim_configured | endpoint | admin |
| idp.scim_sync_completed | users_created, users_updated, users_deprovisioned | scim |
| idp.deactivated | reason | admin |

### Consent Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| consent.granted | application_id, scopes | identity |
| consent.updated | application_id, old_scopes, new_scopes | identity |
| consent.revoked | application_id, reason | identity, admin |

### Role Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| role.created | name, permissions | admin |
| role.updated | name, old_permissions, new_permissions | admin |
| role.assigned | identity_id, role_name | admin, scim |
| role.unassigned | identity_id, role_name | admin, scim |
| role.deleted | name | admin |

### AI Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| ai.suggestion_generated | suggestion_type, entity_type, entity_id, confidence | ai |
| ai.suggestion_accepted | suggestion_id, applied_changes | admin |
| ai.suggestion_dismissed | suggestion_id, reason | admin |
| ai.risk_model_updated | model_version, training_metrics | ai |
| ai.anomaly_detected | entity_type, entity_id, anomaly_type, severity | ai |
| ai.credential_compromise_detected | identity_id, source, risk_level | ai |
| ai.phishing_attempt_blocked | identity_id, recovery_type, signals | ai |

---

## Read Models

### rm_identity_credentials

Login hot path — identity lookup with all active credentials.

```sql
CREATE TABLE rm_identity_credentials (
  identity_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  email TEXT,
  email_verified BOOLEAN NOT NULL DEFAULT false,
  phone TEXT,
  phone_verified BOOLEAN NOT NULL DEFAULT false,
  name TEXT,
  status TEXT NOT NULL,
  locked_until TIMESTAMPTZ,
  failed_login_attempts INTEGER NOT NULL DEFAULT 0,
  -- MFA state
  mfa_enabled BOOLEAN NOT NULL DEFAULT false,
  mfa_preferred_method TEXT,
  -- Password credential (for fast verification)
  password_hash TEXT,
  password_algorithm TEXT,
  password_needs_rehash BOOLEAN,
  -- WebAuthn credentials
  webauthn_credentials JSONB NOT NULL DEFAULT '[]',
  -- Example webauthn_credentials:
  -- [
  --   {
  --     "credential_id": "base64...", "public_key": "base64...",
  --     "sign_count": 42, "device_name": "iPhone Passkey",
  --     "is_resident": true, "transports": ["internal","hybrid"]
  --   }
  -- ]
  -- TOTP state
  totp_enrolled BOOLEAN NOT NULL DEFAULT false,
  totp_secret_encrypted TEXT,
  totp_config JSONB,
  -- Social connections
  social_connections JSONB NOT NULL DEFAULT '[]',
  -- RBAC
  roles JSONB NOT NULL DEFAULT '[]',
  permissions TEXT[] NOT NULL DEFAULT '{}',
  -- Consent grants
  consent_grants JSONB NOT NULL DEFAULT '{}',
  -- Custom claims
  custom_claims JSONB NOT NULL DEFAULT '{}',
  metadata JSONB NOT NULL DEFAULT '{}',
  -- Timestamps
  last_login_at TIMESTAMPTZ,
  password_changed_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_rm_identity_email ON rm_identity_credentials(tenant_id, email)
  WHERE email IS NOT NULL;
CREATE INDEX idx_rm_identity_tenant ON rm_identity_credentials(tenant_id);
CREATE INDEX idx_rm_identity_webauthn ON rm_identity_credentials USING GIN (webauthn_credentials);
```

### rm_active_sessions

Session validation and management.

```sql
CREATE TABLE rm_active_sessions (
  session_id UUID PRIMARY KEY,
  identity_id UUID NOT NULL,
  tenant_id UUID NOT NULL,
  application_id TEXT,
  jti TEXT NOT NULL UNIQUE,
  refresh_token_hash TEXT,
  refresh_token_family TEXT,
  -- Auth context
  auth_method TEXT NOT NULL,
  mfa_completed BOOLEAN NOT NULL DEFAULT false,
  mfa_method TEXT,
  aal INTEGER NOT NULL DEFAULT 1,
  -- Client context
  ip_address INET,
  user_agent TEXT,
  device_fingerprint TEXT,
  geo_country TEXT,
  -- Lifecycle
  status TEXT NOT NULL DEFAULT 'active',
  expires_at TIMESTAMPTZ NOT NULL,
  last_activity_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_sessions_identity ON rm_active_sessions(identity_id, status)
  WHERE status = 'active';
CREATE INDEX idx_rm_sessions_refresh ON rm_active_sessions(refresh_token_hash)
  WHERE refresh_token_hash IS NOT NULL;
CREATE INDEX idx_rm_sessions_family ON rm_active_sessions(refresh_token_family);
CREATE INDEX idx_rm_sessions_expiry ON rm_active_sessions(expires_at)
  WHERE status = 'active';
```

### rm_tenant_config

Pre-computed tenant configuration for login flows.

```sql
CREATE TABLE rm_tenant_config (
  tenant_id UUID PRIMARY KEY,
  slug TEXT NOT NULL UNIQUE,
  status TEXT NOT NULL,
  -- Applications
  applications JSONB NOT NULL DEFAULT '[]',
  -- Identity providers
  identity_providers JSONB NOT NULL DEFAULT '[]',
  -- Security policies
  security_config JSONB NOT NULL DEFAULT '{}',
  -- RBAC definitions
  roles JSONB NOT NULL DEFAULT '[]',
  -- JWKS
  jwks_config JSONB NOT NULL DEFAULT '{}',
  -- Webhooks
  webhooks JSONB NOT NULL DEFAULT '[]',
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### rm_login_analytics

Time-bucketed login metrics for security dashboards.

```sql
CREATE TABLE rm_login_analytics (
  tenant_id UUID NOT NULL,
  bucket TIMESTAMPTZ NOT NULL,
  granularity TEXT NOT NULL DEFAULT '1h'
    CHECK (granularity IN ('1m','5m','1h','1d')),
  -- Login counts
  login_attempts INTEGER NOT NULL DEFAULT 0,
  login_successes INTEGER NOT NULL DEFAULT 0,
  login_failures INTEGER NOT NULL DEFAULT 0,
  -- Auth method breakdown
  method_stats JSONB NOT NULL DEFAULT '{}',
  -- Example method_stats:
  -- {
  --   "password": {"attempts": 500, "successes": 450, "failures": 50},
  --   "webauthn": {"attempts": 200, "successes": 198, "failures": 2},
  --   "social": {"attempts": 150, "successes": 148, "failures": 2}
  -- }
  -- MFA stats
  mfa_challenges_issued INTEGER NOT NULL DEFAULT 0,
  mfa_challenges_passed INTEGER NOT NULL DEFAULT 0,
  mfa_challenges_failed INTEGER NOT NULL DEFAULT 0,
  -- Security events
  accounts_locked INTEGER NOT NULL DEFAULT 0,
  credential_stuffing_events INTEGER NOT NULL DEFAULT 0,
  brute_force_events INTEGER NOT NULL DEFAULT 0,
  compromised_credentials_detected INTEGER NOT NULL DEFAULT 0,
  -- Risk distribution
  high_risk_logins INTEGER NOT NULL DEFAULT 0,
  medium_risk_logins INTEGER NOT NULL DEFAULT 0,
  low_risk_logins INTEGER NOT NULL DEFAULT 0,
  -- Unique identities
  unique_identities INTEGER NOT NULL DEFAULT 0,
  new_registrations INTEGER NOT NULL DEFAULT 0,
  -- Top failure reasons
  top_failure_reasons JSONB NOT NULL DEFAULT '[]',
  -- Top IPs
  top_failed_ips JSONB NOT NULL DEFAULT '[]',
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, bucket, granularity)
);

CREATE INDEX idx_rm_login_analytics_tenant ON rm_login_analytics(tenant_id, bucket DESC);
```

### rm_security_posture

Per-tenant security posture summary.

```sql
CREATE TABLE rm_security_posture (
  tenant_id UUID PRIMARY KEY,
  -- Identity stats
  total_identities INTEGER NOT NULL DEFAULT 0,
  active_identities INTEGER NOT NULL DEFAULT 0,
  locked_identities INTEGER NOT NULL DEFAULT 0,
  -- Credential stats
  identities_with_password INTEGER NOT NULL DEFAULT 0,
  identities_with_passkey INTEGER NOT NULL DEFAULT 0,
  identities_with_totp INTEGER NOT NULL DEFAULT 0,
  identities_with_social INTEGER NOT NULL DEFAULT 0,
  -- MFA adoption
  mfa_enabled_count INTEGER NOT NULL DEFAULT 0,
  mfa_adoption_rate NUMERIC(5,4),
  passkey_adoption_rate NUMERIC(5,4),
  -- Password health
  passwords_needing_rehash INTEGER NOT NULL DEFAULT 0,
  compromised_passwords_detected INTEGER NOT NULL DEFAULT 0,
  -- Session stats
  active_sessions INTEGER NOT NULL DEFAULT 0,
  sessions_at_aal1 INTEGER NOT NULL DEFAULT 0,
  sessions_at_aal2 INTEGER NOT NULL DEFAULT 0,
  sessions_at_aal3 INTEGER NOT NULL DEFAULT 0,
  -- Risk metrics (24h)
  login_failure_rate_24h NUMERIC(5,4),
  account_lockouts_24h INTEGER NOT NULL DEFAULT 0,
  credential_stuffing_events_24h INTEGER NOT NULL DEFAULT 0,
  high_risk_logins_24h INTEGER NOT NULL DEFAULT 0,
  -- Compliance
  audit_events_24h INTEGER NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_identity_credentials, rm_active_sessions, rm_tenant_config, rm_login_analytics, rm_security_posture |
| **Total** | **8** | 1 partitioned table (event_store) |

---

## Key Design Decisions

1. **Authentication events as the source of truth** — Every login attempt, MFA challenge, and session creation is an immutable event. The event store IS the audit log — there is no separate audit_log table because every event already captures who did what, when, from where, and why. SOC 2 and HIPAA auditors can query the event store directly for compliance evidence.

2. **Credential lifecycle as granular events** — Password changes, WebAuthn registrations, TOTP enrollments, and social connections each generate specific events. The webauthn.used event records sign_count, enabling cloned authenticator detection by comparing sequential sign counts. The compromised_detected event enables real-time notification when a password appears in a known breach list.

3. **Login flow as a multi-event sequence** — A single login generates 3-5 events: login.attempted → login.risk_evaluated → login.mfa_challenge_issued → login.mfa_challenge_verified → login.succeeded → session.created. This granularity enables post-incident analysis: "the risk engine evaluated the login as low-risk, but the device was actually compromised — we need to retrain the model."

4. **rm_identity_credentials as the login hot path** — The identity lookup and credential verification during login reads rm_identity_credentials — a single row containing the identity's email, status, password hash, WebAuthn credentials, TOTP state, and social connections. This is projected from identity and credential stream events. The password hash is stored in the read model for fast bcrypt/Argon2 verification.

5. **Session events with AAL tracking** — Session events record the Authentication Assurance Level per NIST SP 800-63-4. Step-up authentication generates a session.step_up_completed event that upgrades the AAL. The rm_active_sessions read model reflects the current AAL for each session, enabling resource servers to enforce AAL-gated access control.

6. **Refresh token rotation as events** — Token refresh generates a session.refreshed event with old_jti and new_jti. If a previously-used JTI appears in a refresh request (rotation_detected = true), all sessions in that refresh_token_family are revoked via session.revoked events. This implements the replay detection pattern from RFC 9700.

7. **Risk scoring as first-class events** — The risk engine emits login.risk_evaluated events with structured signals (new_device, unusual_location, velocity, VPN detection, breached password). These events feed the rm_login_analytics read model for security dashboards and provide the training data for adaptive MFA: the AI stream emits suggestions to adjust MFA requirements based on observed risk patterns.

8. **12 stream types covering the full identity domain** — tenant, application, identity_provider, identity, credential, session, role, consent, verification, login, ai, config. The login stream is the highest-volume; the credential stream contains the most security-sensitive events. Separating login events from session events enables independent analysis of authentication attempts vs. authenticated activity.

9. **Security posture as a real-time read model** — rm_security_posture provides a dashboard-ready summary of tenant security health: MFA adoption rate, passkey adoption, compromised passwords, active sessions by AAL, and risk metrics. This enables the compliance-first configuration feature — security teams can see at a glance whether their tenant meets SOC 2 or HIPAA requirements.

10. **Actor types include federation sources** — Beyond identity/admin/system/ai, actor types include social_provider (for OAuth callback events), saml_idp (for SAML assertion events), ldap (for LDAP bind events), scim (for provisioning events), and risk_engine (for anomaly detection events). This fine-grained attribution enables per-source debugging and compliance reporting: "how many users were provisioned via SCIM vs. self-registration?"
