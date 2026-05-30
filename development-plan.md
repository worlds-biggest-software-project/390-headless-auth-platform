# Headless Auth Platform — Phased Development Plan

> Project: 390-headless-auth-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. The database schema is taken from **data-model-suggestion-1 (Entity-Centric Normalized Relational)** — the best fit for a production identity engine where security invariants (credential storage, session lifecycle, MFA enforcement, refresh-token rotation) must be enforced by database constraints, with audit-first ideas from suggestion 3 folded into the `audit_log` and `login_attempts` tables.

The product is an **API-first, headless, multi-tenant identity engine**: no opinionated UI, every capability exposed as a typed REST endpoint, OpenID-certified-grade OIDC/OAuth2, SAML 2.0, WebAuthn passkeys, MFA, sessions with remote revocation, SCIM, audit logging, and an AI-native adaptive-risk layer. Deployment is self-hosted-first (Docker) with a managed-SaaS path.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | The AI-native differentiators (adaptive MFA, anomaly detection, credential-compromise scoring) are the project's strategic moat and live most naturally in Python's ML/data ecosystem. Mature, audited crypto and protocol libraries exist (`authlib`, `python-jose`, `webauthn`, `python3-saml`, `passlib`). |
| API framework | **FastAPI** | Generates OpenAPI 3.1 automatically (a standards.md requirement for SDK generation), native Pydantic validation for the security-sensitive request surface, async I/O for high-concurrency token endpoints, and dependency-injection that cleanly models per-tenant context and auth scopes. |
| ASGI server | **Uvicorn + Gunicorn** (uvicorn workers) | Production-grade async serving; Gunicorn process manager for multi-core. |
| Database | **PostgreSQL 16** | The chosen data model uses UUIDs, JSONB, array columns, partitioned tables (`login_attempts`, `audit_log`), GIN indexes, and `INET` types — all native to Postgres. No SQLite fallback: the security invariants depend on Postgres-specific constraints and partitioning. |
| Migrations | **Alembic** | Standard SQLAlchemy migration tool; deterministic, reviewable DDL — essential for an auth schema audited under SOC 2. |
| ORM / query layer | **SQLAlchemy 2.0 (async)** | Async sessions matching FastAPI; typed models; raw SQL escape hatch for partition management and hot-path login queries. |
| Cache / ephemeral store | **Redis 7** | Rate-limit counters, OTP/challenge state, JWKS key cache, distributed lock for refresh-token rotation, and revocation lists for fast session-revocation propagation. |
| Task queue | **Celery + Redis broker** | Async workloads: webhook delivery with retries, breached-password list sync, audit-log SIEM streaming, SCIM reconciliation, email/SMS dispatch. |
| JOSE / JWT | **python-jose[cryptography]** + **authlib** | RFC 7519/7515/7517/7518 JWT/JWS/JWK; `authlib` provides battle-tested OAuth2/OIDC authorization-server primitives (PKCE, grant validation). |
| WebAuthn | **py_webauthn** (Duo Labs) | W3C WebAuthn Level 2 / FIDO2 registration and assertion ceremonies, sign-count verification, attestation parsing. |
| SAML | **python3-saml** (OneLogin) | SAML 2.0 SP and IdP-side assertion handling, metadata, HTTP-POST/Redirect bindings. |
| Password hashing | **passlib[argon2,bcrypt]** | Argon2id default (OWASP), bcrypt verification for legacy-hash migration with on-demand re-hash. |
| TOTP | **pyotp** | RFC 6238 TOTP generation/verification. |
| Crypto at rest | **cryptography (Fernet/AES-GCM)** via an envelope-encryption `KeyProvider` | TOTP secrets, social tokens, LDAP bind passwords encrypted at rest; key material referenced by `*_ref` columns resolved through a KMS-pluggable provider. |
| AI / risk scoring | **scikit-learn** (Isolation Forest / logistic baseline) + pluggable LLM provider via **httpx** | Anomaly detection and risk scoring start as classical ML on login-attempt features; an optional LLM provider abstraction enables policy-recommendation suggestions. Keeps inference local and privacy-preserving. |
| HTTP client | **httpx (async)** | Social-IdP token exchange, OIDC discovery, webhook delivery, breached-password API. |
| Validation / settings | **Pydantic v2 + pydantic-settings** | Request/response schemas and 12-factor env-driven config. |
| Testing | **pytest + pytest-asyncio + httpx.AsyncClient + testcontainers** | Unit + integration with real ephemeral Postgres/Redis containers; `respx` to mock external HTTP. |
| Code quality | **ruff** (lint+format), **mypy** (strict) | Fast linting/formatting and static typing on a security-critical codebase. |
| Package manager | **uv** | Fast, lockfile-based, reproducible builds for `pyproject.toml`. |
| Containerisation | **Docker + docker-compose** | Self-hosted-first deployment (mirrors ZITADEL/Ory operating model): app, Postgres, Redis, Celery worker. |
| Secrets / config | **Environment variables + KeyProvider abstraction** | KMS-pluggable (env, AWS KMS, Vault) so private signing keys and encryption keys never live in the DB. |

### Project Structure

```
headless-auth-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── openapi/
│   └── openapi.json                  # exported spec (CI artifact)
├── migrations/                        # Alembic
│   ├── env.py
│   └── versions/
├── src/
│   └── hap/
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory, router registration
│       ├── config.py                  # Pydantic settings
│       ├── db/
│       │   ├── session.py             # async engine/session
│       │   ├── models/                # SQLAlchemy models (one file per entity group)
│       │   │   ├── tenant.py
│       │   │   ├── application.py
│       │   │   ├── identity_provider.py
│       │   │   ├── identity.py
│       │   │   ├── credential.py
│       │   │   ├── session.py
│       │   │   ├── consent.py
│       │   │   ├── rbac.py
│       │   │   ├── verification.py
│       │   │   ├── security.py        # login_attempts
│       │   │   └── operations.py      # webhooks, ai_suggestions, audit_log
│       │   └── partitions.py          # partition creation helpers
│       ├── crypto/
│       │   ├── keys.py                # KeyProvider, JWKS management
│       │   ├── jwt.py                 # sign/verify ID/access tokens
│       │   ├── hashing.py             # password hash/verify/rehash
│       │   └── encryption.py          # at-rest envelope encryption
│       ├── core/
│       │   ├── context.py             # TenantContext, AuthContext DI
│       │   ├── errors.py              # OAuth/OIDC error model
│       │   ├── ratelimit.py           # Redis token-bucket
│       │   └── audit.py               # audit_log writer
│       ├── domain/                    # business logic (framework-agnostic)
│       │   ├── identities.py
│       │   ├── credentials/
│       │   │   ├── password.py
│       │   │   ├── totp.py
│       │   │   ├── webauthn.py
│       │   │   └── otp.py             # email/sms OTP
│       │   ├── sessions.py
│       │   ├── tokens.py              # refresh rotation, revocation
│       │   ├── oauth/                 # authorization server
│       │   │   ├── authorize.py
│       │   │   ├── token.py
│       │   │   ├── introspect.py
│       │   │   ├── revoke.py
│       │   │   └── discovery.py
│       │   ├── saml/
│       │   ├── federation/            # social, OIDC, LDAP IdPs
│       │   ├── scim/
│       │   ├── rbac.py
│       │   ├── risk/                  # AI-native layer
│       │   │   ├── signals.py
│       │   │   ├── scorer.py
│       │   │   └── breached.py
│       │   └── webhooks.py
│       ├── api/
│       │   └── v1/
│       │       ├── oauth.py           # /oauth2/* + /.well-known/*
│       │       ├── auth.py            # login/register/verify/reset flows
│       │       ├── mfa.py
│       │       ├── webauthn.py
│       │       ├── sessions.py
│       │       ├── identities.py      # management API
│       │       ├── tenants.py
│       │       ├── applications.py
│       │       ├── providers.py
│       │       ├── saml.py
│       │       ├── scim.py
│       │       ├── rbac.py
│       │       └── webhooks.py
│       ├── schemas/                   # Pydantic request/response models
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks.py
│       └── integrations/
│           ├── email.py               # SMTP/provider abstraction
│           └── sms.py
└── tests/
    ├── conftest.py                    # testcontainers fixtures (pg, redis)
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure groups by concern (crypto, domain logic, API routes, workers) so each phase adds modules without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Config, Database, Crypto Primitives

### Purpose
Establish the runnable application shell, the full database schema (all 14 tables from data-model-suggestion-1), the cryptographic primitives every later phase depends on (key management, JWT signing, password hashing, at-rest encryption), and the multi-tenant request context. After this phase the service boots, connects to Postgres and Redis, exposes a health check, and can sign/verify a JWT against a tenant's keypair.

### Tasks

#### 1.1 — Project scaffold, settings, and app factory

**What**: Create the `uv`-managed package, Pydantic settings, and a FastAPI app factory with health/readiness endpoints.

**Design**:
- `config.py` settings (env-driven, prefix `HAP_`):
```python
class Settings(BaseSettings):
    database_url: str            # postgresql+asyncpg://...
    redis_url: str               # redis://...
    key_provider: Literal["env","aws_kms","vault"] = "env"
    master_encryption_key: SecretStr        # base64, used by env KeyProvider
    default_jwks_algorithm: Literal["RS256","ES256","EdDSA"] = "RS256"
    access_token_default_ttl: int = 3600
    environment: Literal["dev","staging","prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="HAP_")
```
- `main.py`: `create_app()` builds FastAPI with OpenAPI metadata (title, version, OAuth2 security schemes), registers routers (added per phase), installs exception handlers (Task 1.4) and middleware (request-id, tenant resolution).
- Endpoints: `GET /healthz` (liveness, no DB), `GET /readyz` (checks DB + Redis).

**Testing**:
- `Unit: Settings loads from env with HAP_ prefix → correct typed values, SecretStr masks master key in repr`.
- `Unit: missing DATABASE_URL → ValidationError naming database_url`.
- `Integration: GET /healthz → 200 {"status":"ok"}`.
- `Integration (real pg+redis via testcontainers): GET /readyz → 200; with Redis down → 503`.

#### 1.2 — Database models and migrations (full schema)

**What**: Implement all 14 tables from data-model-suggestion-1 as SQLAlchemy models plus the initial Alembic migration, including partitioned `login_attempts` and `audit_log`.

**Design**:
- Models map 1:1 to the DDL in data-model-suggestion-1: `tenants, applications, identity_providers, identities, credentials, sessions, consent_grants, roles, role_assignments, verification_tokens, login_attempts, webhooks, ai_suggestions, audit_log`.
- CHECK constraints, UNIQUE constraints, FKs, and partial/GIN indexes are declared in the models (`__table_args__`) so they appear in migrations.
- Partitioned tables: the migration creates the parent (`PARTITION BY RANGE (created_at)`) plus an initial monthly partition; `db/partitions.py` exposes `ensure_partition(table, month)` used by a Celery beat task (Phase 9).
- Enums modelled as `CHECK (x IN (...))` text columns (not PG enums) to allow additive values without ALTER TYPE migrations.
- `*_ref` columns (`jwks_private_key_ref`, `client_secret_ref`, `totp_secret_encrypted`, `social_*_encrypted`, `ldap_bind_password_ref`) store opaque references/ciphertext, never plaintext.

**Testing**:
- `Integration (real pg): alembic upgrade head → all 14 tables + partitions exist; downgrade base → clean`.
- `Integration: insert tenant then application with bad FK tenant_id → IntegrityError`.
- `Integration: insert credential with credential_type='bogus' → CHECK violation`.
- `Integration: two identities same (tenant_id, email) → UNIQUE violation`.
- `Integration: login_attempts insert routes to correct monthly partition`.

#### 1.3 — Crypto layer: KeyProvider, JWKS, JWT, hashing, encryption

**What**: Implement signing-key management, JWKS publication primitives, JWT sign/verify, password hashing with rehash, and at-rest envelope encryption.

**Design**:
- `crypto/keys.py`:
```python
class KeyProvider(Protocol):
    def get_private_key(self, ref: str) -> PrivateKey: ...
    def store_private_key(self, key: PrivateKey) -> str: ...   # returns ref
    def get_encryption_key(self) -> bytes: ...

class EnvKeyProvider(KeyProvider):  # default; AWS/Vault implement same Protocol
    ...
def generate_signing_keypair(alg: str) -> tuple[PrivateKey, JWK]
def build_jwks(tenant) -> dict   # {"keys":[{kid,kty,use:"sig",alg,n,e|crv,x,y}]}
```
- `crypto/jwt.py`: `sign_jwt(claims, tenant) -> str` (sets `kid`, `iss`, `iat`, `exp`, `jti`); `verify_jwt(token, jwks) -> Claims` (validates `exp`, `iss`, `aud`, signature). Algorithms restricted to RS256/ES256/EdDSA (RFC 7518); `alg:none` rejected.
- `crypto/hashing.py`: `hash_password(pw, policy) -> (hash, algorithm)`; `verify_password(pw, hash, algorithm) -> bool`; `needs_rehash(algorithm, policy) -> bool` (true for bcrypt when policy=argon2id — supports migration via the `password_needs_rehash` column).
- `crypto/encryption.py`: `encrypt(plaintext) -> ciphertext` / `decrypt(ciphertext)` using AES-GCM with the provider's data key (envelope encryption).

**Testing**:
- `Unit: sign_jwt then verify_jwt → claims round-trip; tampered payload → InvalidSignature`.
- `Unit: verify rejects alg:none and HS* when tenant uses RS256`.
- `Unit: expired exp → ExpiredSignatureError`.
- `Unit: hash_password(argon2id) → verify true; wrong pw → false`.
- `Unit: legacy bcrypt hash verifies true AND needs_rehash → true`.
- `Unit: encrypt→decrypt round-trips; ciphertext != plaintext; tampered ciphertext → decryption error`.
- `Unit: build_jwks emits valid JWK with kid matching signing key`.

#### 1.4 — Multi-tenant context, error model, audit writer

**What**: Tenant resolution dependency, the OAuth/OIDC-compliant error model, and the central audit-log writer.

**Design**:
- `core/context.py`: `TenantContext(tenant_id, tenant)` resolved via FastAPI dependency from `client_id`, host header, or admin API key; raises 404 for unknown/suspended tenants.
- `core/errors.py`: `AuthError(error, error_description, status, error_uri=None)` rendered as RFC 6749 §5.2 JSON `{"error":"invalid_request","error_description":"..."}`; standard codes (`invalid_request, invalid_client, invalid_grant, unauthorized_client, unsupported_grant_type, invalid_scope, access_denied, server_error`). Exception handler maps to HTTP status.
- `core/audit.py`: `record_audit(actor_id, actor_type, action, entity_type, entity_id, changes=None, ip=None, session_id=None, risk_score=None)` → insert into `audit_log`; called by all mutating domain operations.

**Testing**:
- `Unit: raising AuthError("invalid_grant") → JSON body + 400`.
- `Integration: request with unknown client_id → 401 invalid_client`.
- `Integration: suspended tenant → 403`.
- `Integration: record_audit writes row with correct tenant_id and action`.

---

## Phase 2: Identities & Password Credentials

### Purpose
Implement the core identity lifecycle and the password credential type: registration, password hashing/verification with account-lockout, email-verification and password-reset flows backed by `verification_tokens`, and credential migration (bcrypt → argon2id on next login). This is the foundation of every authentication method and the management API.

### Tasks

#### 2.1 — Identity CRUD domain + management API

**What**: Create/read/update/soft-delete identities with claims, scoped to a tenant.

**Design**:
- `domain/identities.py`: `create_identity(ctx, email, attrs) -> Identity` (enforces tenant password/MFA policy defaults), `get_identity`, `update_identity`, `set_status`, `soft_delete` (status='deleted', GDPR erasure scrubs PII columns).
- API (`/v1/identities`, requires admin API key scope `identities:write`/`:read`):
  - `POST /v1/identities` → 201 `IdentityResponse`
  - `GET /v1/identities/{id}` → 200
  - `GET /v1/identities?email=&status=&cursor=` → cursor-paginated list
  - `PATCH /v1/identities/{id}` (claims, name, status) → 200
  - `DELETE /v1/identities/{id}` → 204 (soft delete + erasure)
- `IdentityResponse` excludes credential material; includes `custom_claims`, `mfa_enabled`, `status`.

**Testing**:
- `Unit: create_identity sets tenant defaults; duplicate email same tenant → ConflictError`.
- `Integration: full CRUD cycle via API → correct status codes`.
- `Integration: DELETE → PII columns null, status='deleted', audit row written`.
- `Integration: list pagination cursor is stable across inserts`.

#### 2.2 — Password registration & lockout

**What**: Register a password credential and authenticate against it with policy enforcement and account lockout.

**Design**:
- `domain/credentials/password.py`:
```python
def set_password(ctx, identity_id, password) -> None      # validates tenant password_policy; stores hash+algorithm
def verify_credentials(ctx, email, password) -> VerifyResult
# VerifyResult: {ok: bool, identity_id, needs_rehash, locked, reason}
```
- Lockout: on failure increment `identities.failed_login_attempts`; when `>= tenant.max_failed_login_attempts` set `locked_until = now + lockout_duration_seconds`, status='locked'. Successful verify resets counter. (OWASP A07 mitigation.)
- On success with `needs_rehash` → re-hash with tenant algorithm, update `password_hash`, clear `password_needs_rehash`.
- Every attempt writes a `login_attempts` row (success/failure, reason, ip, user_agent) — feeds Phase 8 risk layer.

**Testing**:
- `Unit: password below min_length → PolicyError listing failed rules`.
- `Unit: N consecutive failures → locked_until set; verify during lockout → locked reason`.
- `Unit: successful login with legacy bcrypt → hash upgraded to argon2id, needs_rehash cleared`.
- `Integration: each verify writes a login_attempts row with correct success flag`.

#### 2.3 — Verification tokens: email verification & password reset

**What**: Issue, deliver, and consume single-use tokens for email verification and password reset.

**Design**:
- `domain/verification.py`: `issue_token(type, identity_id, ttl, max_attempts=3) -> (raw_token, record)` stores only `sha256(raw_token)` in `token_hash`; `consume_token(type, raw_token) -> record` validates not-expired, not-used, attempts<max, marks `used_at`.
- Flows:
  - `POST /v1/auth/verify-email/request` → issues `email_verification`, enqueues email (Phase 9 Celery; synchronous SMTP in dev).
  - `POST /v1/auth/verify-email/confirm {token}` → sets `email_verified=true`.
  - `POST /v1/auth/password-reset/request {email}` → always 202 (no user enumeration).
  - `POST /v1/auth/password-reset/confirm {token, new_password}` → validates token, sets password, revokes all active sessions for that identity.
- Token entropy ≥ 256 bits, URL-safe base64.

**Testing**:
- `Unit: consume expired token → TokenExpiredError`.
- `Unit: reused token → TokenUsedError`.
- `Unit: token_hash stored, raw token never persisted`.
- `Integration: password-reset for unknown email → 202 (no enumeration)`.
- `Integration: successful reset revokes existing sessions`.

---

## Phase 3: OAuth 2.0 / OIDC Authorization Server (Core Value)

### Purpose
The heart of the product: a standards-compliant OAuth 2.0 / OpenID Connect authorization server. Implements the authorization-code flow with mandatory PKCE (RFC 9700), token issuance (ID token + access token + refresh token), discovery, and JWKS publication. After this phase a relying party can complete a full login-to-token round trip.

### Tasks

#### 3.1 — Discovery & JWKS endpoints

**What**: Publish OIDC Discovery, OAuth AS metadata, and JWKS so clients self-configure.

**Design**:
- `GET /.well-known/openid-configuration` (OIDC Discovery 1.0): issuer, authorization_endpoint, token_endpoint, userinfo_endpoint, jwks_uri, introspection_endpoint, revocation_endpoint, `response_types_supported:["code"]`, `grant_types_supported:["authorization_code","refresh_token","client_credentials","urn:ietf:params:oauth:grant-type:device_code"]`, `code_challenge_methods_supported:["S256"]`, `id_token_signing_alg_values_supported` from tenant.
- `GET /.well-known/oauth-authorization-server` (RFC 8414): same data, OAuth framing.
- `GET /oauth2/jwks` → tenant JWKS from `crypto/keys.build_jwks` (includes rotated keys still valid for verification).

**Testing**:
- `Integration: discovery doc validates against OIDC metadata schema; all endpoint URLs absolute and reachable`.
- `Integration: jwks contains current signing kid; alg matches tenant`.
- `Integration: code_challenge_methods_supported == ["S256"] (plain disallowed)`.

#### 3.2 — Authorization endpoint (code flow + PKCE)

**What**: `GET/POST /oauth2/authorize` issuing an authorization code bound to a PKCE challenge and an authenticated session.

**Design**:
- Validates `client_id` (active application), exact-match `redirect_uri` against `applications.redirect_uris` (RFC 9700), `response_type=code`, `scope ⊆ application.scopes`, requires `code_challenge` + `code_challenge_method=S256` (reject if missing → `invalid_request`).
- Requires an authenticated identity (server-side session cookie from Phase 4, or a just-completed login). If unauthenticated → returns `login_required` interaction signal (headless: a JSON 401 telling the client to drive its own login UI, then re-call authorize).
- Consent: if no valid `consent_grants` row for (identity, application) covering requested scopes → returns `consent_required` with scope list; client renders consent UI and re-calls with `consent=granted`.
- On success: create `verification_tokens` row type `authorization_code` storing `code_hash`, `code_challenge`, `redirect_uri`, `application_id`, `scopes`, `nonce`, `identity_id`, 60s TTL; redirect with `code` + `state`.

**Testing**:
- `Unit: missing code_challenge → invalid_request`.
- `Unit: redirect_uri not exact-match → invalid_request, no redirect`.
- `Unit: scope superset of allowed → invalid_scope`.
- `Integration: authenticated + consented → 302 with code & state preserved`.
- `Integration: unauthenticated → login_required; un-consented → consent_required`.

#### 3.3 — Token endpoint (code, refresh, client_credentials) + ID token

**What**: `POST /oauth2/token` exchanging codes/refresh tokens for tokens, with PKCE verification and refresh-token rotation.

**Design**:
- Client auth: `client_secret_basic`/`client_secret_post` for confidential clients (verify `client_secret_hash`); public clients (SPA/native) authenticate via PKCE only.
- `grant_type=authorization_code`: look up code by hash, verify not used/expired, verify `S256(code_verifier)==code_challenge`, verify `redirect_uri` matches, mark code used. Issue:
  - **ID token** (JWT): `iss, sub=identity_id, aud=client_id, iat, exp, nonce, auth_time, acr (aal), amr (auth methods)` + tenant/app `custom_claims` + standard profile claims per granted scopes.
  - **access token** (JWT): `iss, sub, aud, scope, exp, jti, client_id`.
  - **refresh token** (opaque, random ≥256-bit): store `refresh_token_hash` + new `refresh_token_family` on the created `sessions` row.
- `grant_type=refresh_token`: validate hash; **rotation** — issue new refresh token in same family, mark session 'refreshed'. **Replay detection**: if a refresh token whose hash matches an already-rotated token in a family is presented, revoke the entire family (RFC 9700).
- `grant_type=client_credentials`: machine-to-machine; no ID token; access token with app scopes.
- Redis distributed lock per refresh-token family prevents rotation race conditions.

**Testing**:
- `Unit: wrong code_verifier → invalid_grant`.
- `Unit: reused authorization code → invalid_grant`.
- `Integration: code→tokens; ID token verifies against /oauth2/jwks; nonce echoed`.
- `Integration: refresh rotates token, old hash invalid, new works`.
- `Integration: replay old refresh token → entire family revoked, all sessions revoked`.
- `Integration: client_credentials → access token, no id_token`.
- `E2E: discovery → authorize → token → userinfo full round trip succeeds`.

#### 3.4 — UserInfo, introspection, revocation

**What**: RFC-compliant `/userinfo`, `/introspect` (RFC 7662), `/revoke` (RFC 7009).

**Design**:
- `GET /oauth2/userinfo` (Bearer access token): returns claims for granted scopes (`openid→sub`, `profile→name/given/family/picture`, `email→email/email_verified`).
- `POST /oauth2/introspect` (client-authenticated): returns `{active, scope, client_id, sub, exp, token_type}`; inactive/revoked/expired → `{active:false}`.
- `POST /oauth2/revoke`: revokes access or refresh token; refresh revocation revokes its session + family; adds access-token `jti` to Redis revocation set until natural expiry.

**Testing**:
- `Integration: userinfo with email scope → email present; without → absent`.
- `Integration: introspect active token → active:true; revoked → active:false`.
- `Integration: revoke refresh token → session revoked, subsequent refresh fails`.
- `Integration: introspect with invalid client auth → 401`.

---

## Phase 4: Sessions, Step-Up & AAL

### Purpose
Add server-side session management (the Hanko/Clerk hybrid pattern) on top of token issuance: session creation at login, remote revocation, session listing, and Authentication Assurance Level (AAL per NIST SP 800-63-4) tracking that later enables step-up authentication. This decouples "is this user logged in" from "does this token grant scope X".

### Tasks

#### 4.1 — Session lifecycle & server-side session cookie

**What**: Create, validate, and revoke sessions; issue a server-side session cookie used by the authorize endpoint.

**Design**:
- `domain/sessions.py`: `create_session(ctx, identity_id, auth_method, aal, request_meta) -> Session` writes `sessions` row (jti, expires_at from tenant lifetime, ip/user_agent/device_fingerprint/geo); returns a session cookie (signed, httpOnly, Secure, SameSite=Lax) carrying the session id.
- `validate_session(cookie) -> Session|None`: checks status='active', not expired; updates `last_activity_at`; checks Redis revocation set first (fast path).
- `revoke_session(session_id, reason)`: status='revoked', `revoked_at`, add jti to Redis revocation set; propagates to bound refresh-token family.
- `list_sessions(identity_id)`: active sessions with device/geo metadata for "manage your devices" UIs.

**Testing**:
- `Unit: create_session sets expires_at = now + tenant.session_lifetime`.
- `Integration: validate_session after revoke → None (Redis fast-path hit)`.
- `Integration: expired session → validate returns None and marks 'expired'`.
- `Integration: list_sessions returns only active, with geo/device fields`.

#### 4.2 — Session management API

**What**: REST endpoints for current-user and admin session management.

**Design**:
- `GET /v1/sessions` (Bearer/cookie) → caller's active sessions.
- `DELETE /v1/sessions/{id}` → revoke one (must own it or admin scope).
- `DELETE /v1/sessions` → revoke all of caller's sessions (logout-everywhere).
- `GET /v1/identities/{id}/sessions` (admin `sessions:read`) → any identity's sessions.
- `POST /v1/identities/{id}/sessions:revoke-all` (admin) → security response action.

**Testing**:
- `Integration: user revokes another user's session → 403`.
- `Integration: logout-everywhere revokes all, tokens for those sessions stop introspecting active`.
- `Integration: admin revoke-all for compromised identity → all sessions revoked + audit row`.

#### 4.3 — Step-up authentication & AAL enforcement

**What**: Allow applications to require a minimum AAL for an action and drive a step-up flow.

**Design**:
- Access tokens carry `acr` = `aal{1,2,3}`. Resource servers / the authorize endpoint can require `acr_values=aal2`.
- If session AAL < required, authorize returns `interaction_required` with `required_acr`; client runs an MFA challenge (Phase 5/6); on success `sessions.aal` is upgraded and `mfa_completed=true` without creating a new session.
- `domain/sessions.py: elevate_aal(session_id, new_aal, mfa_method)`.
- AAL mapping: AAL1 single factor; AAL2 password+TOTP/OTP or passkey w/ user verification; AAL3 passkey with hardware attestation + UV.

**Testing**:
- `Unit: required aal2 on aal1 session → interaction_required`.
- `Integration: complete TOTP step-up → session aal=2, mfa_completed=true, same session id`.
- `Integration: token issued after step-up carries acr=aal2`.

---

## Phase 5: MFA — TOTP, Email OTP, SMS OTP

### Purpose
Deliver the MVP multi-factor methods: TOTP enrolment/verification, and email/SMS OTP. These plug into the AAL/step-up machinery from Phase 4 and the login flow from Phase 2, satisfying the NIST AAL2 requirement.

### Tasks

#### 5.1 — TOTP enrolment & verification

**What**: Enrol a TOTP authenticator (RFC 6238) and verify codes.

**Design**:
- `domain/credentials/totp.py`: `begin_totp_enrolment(identity_id) -> {secret, otpauth_uri, qr_payload}` — generates secret, stores `totp_secret_encrypted` (at-rest encryption), `totp_verified=false`; `confirm_totp_enrolment(identity_id, code)` verifies a code within ±1 window, sets `totp_verified=true`, `identities.mfa_enabled=true`.
- `verify_totp(identity_id, code) -> bool` with replay protection (Redis stores last-accepted counter per credential to block code reuse).
- API: `POST /v1/mfa/totp/enroll`, `POST /v1/mfa/totp/confirm`, `POST /v1/mfa/totp/verify`.

**Testing**:
- `Unit: confirm with valid code → totp_verified true; invalid → unchanged`.
- `Unit: verify within ±30s window passes; outside fails`.
- `Unit: replay same code in window → rejected after first use`.
- `Integration: secret stored encrypted, never returned after enrolment confirm`.

#### 5.2 — Email & SMS OTP

**What**: One-time-passcode challenges over email and SMS for MFA and passwordless login.

**Design**:
- `domain/credentials/otp.py`: `issue_otp(identity_id, channel) -> challenge_id` generates 6-digit code, stores `verification_tokens` (`email_otp`/`sms_otp`) with `token_hash`, 5-min TTL, `max_attempts=3`; enqueues delivery via `integrations/email.py` or `sms.py`.
- `verify_otp(challenge_id, code) -> bool` increments attempts, locks after max.
- Rate limit per identity/channel via Redis (e.g., max 3 issues / 10 min).
- API: `POST /v1/mfa/otp/send {channel}`, `POST /v1/mfa/otp/verify {challenge_id, code}`.

**Testing**:
- `Unit: 4th wrong attempt → ChallengeLockedError`.
- `Unit: issuing beyond rate limit → RateLimitError`.
- `Integration (mocked email): send → message contains code; verify correct → ok`.
- `Integration: expired challenge → TokenExpiredError`.

---

## Phase 6: WebAuthn / Passkeys (FIDO2)

### Purpose
First-class passkey support per W3C WebAuthn Level 2 / FIDO2 — the table-stakes differentiator from research.md. Implements registration and authentication ceremonies, sign-count cloned-authenticator detection, and passwordless-by-default login. Passkeys with user verification satisfy AAL2; hardware attestation paths satisfy AAL3.

### Tasks

#### 6.1 — Passkey registration ceremony

**What**: Two-step WebAuthn registration storing a credential per data-model-suggestion-1.

**Design**:
- `domain/credentials/webauthn.py` using `py_webauthn`:
  - `begin_registration(identity_id) -> PublicKeyCredentialCreationOptions` (challenge stored in Redis keyed by identity, RP ID = tenant domain, `authenticatorSelection.userVerification="preferred"`, `residentKey="required"` for passkeys).
  - `finish_registration(identity_id, attestation_response)` verifies attestation, stores `credentials` row: `webauthn_credential_id, webauthn_public_key, webauthn_sign_count, webauthn_aaguid, webauthn_transports, webauthn_is_resident, webauthn_user_verified, webauthn_attestation_type`, `webauthn_device_name`.
- API: `POST /v1/webauthn/register/begin`, `POST /v1/webauthn/register/finish`.

**Testing**:
- `Unit (fixture attestation): finish_registration stores credential_id + public_key, sign_count seeded`.
- `Unit: replayed/expired challenge → CeremonyError`.
- `Integration: resident-key registration → is_resident=true`.

#### 6.2 — Passkey authentication ceremony + sign-count check

**What**: Discoverable-credential (usernameless) and known-user assertion ceremonies.

**Design**:
- `begin_authentication(identity_id|None) -> PublicKeyCredentialRequestOptions` (allowCredentials empty for usernameless passkey login; challenge in Redis).
- `finish_authentication(assertion_response) -> identity_id` verifies signature against stored public key; **enforces `new_sign_count > stored_sign_count`** (cloned-authenticator detection) and updates it; sets resulting session AAL=2 if `user_verified`.
- API: `POST /v1/webauthn/login/begin`, `POST /v1/webauthn/login/finish` → creates session (Phase 4) on success.

**Testing**:
- `Unit (fixture assertion): valid signature + increasing sign_count → identity resolved, count updated`.
- `Unit: sign_count <= stored → CloneDetectedError, credential flagged`.
- `Unit: usernameless flow resolves identity from credential_id`.
- `Integration: successful passkey login → session aal=2, login_attempts row method='webauthn'`.

---

## Phase 7: Federation — Social, OIDC, SAML 2.0

### Purpose
Connect to external identity providers: social/OIDC login via OAuth2, and SAML 2.0 SSO with per-tenant IdP config, domain verification, and JIT provisioning — the enterprise-SSO requirement. The platform acts as SP/RP toward upstream IdPs while remaining the AS/IdP toward relying parties.

### Tasks

#### 7.1 — Social & generic OIDC login (upstream)

**What**: Login via Google/GitHub/Apple/Microsoft and any OIDC IdP configured in `identity_providers`.

**Design**:
- `domain/federation/oidc.py`: `start_federated_login(provider_id, state) -> redirect_url` (builds upstream authorize URL with PKCE + state in Redis); `complete_federated_login(provider_id, code, state) -> Identity` exchanges code (httpx), fetches userinfo, applies `attribute_mapping`, then **account linking**: find existing identity by verified email or `(social_provider_id, social_provider_user_id)`; create+link a `social` credential or, if `jit_provisioning`, create a new identity.
- Upstream tokens stored encrypted (`social_access_token_encrypted`, `social_refresh_token_encrypted`).
- API: `GET /v1/providers/{id}/login`, `GET /v1/providers/{id}/callback`.

**Testing**:
- `Integration (respx-mocked Google): callback → identity created, social credential linked`.
- `Unit: state mismatch/CSRF → SecurityError`.
- `Unit: existing verified-email identity → linked, not duplicated`.
- `Integration: jit disabled + unknown user → registration_required`.

#### 7.2 — SAML 2.0 SP (inbound enterprise SSO)

**What**: Accept SAML assertions from enterprise IdPs configured per tenant.

**Design**:
- `domain/saml/` using `python3-saml`: SP metadata at `GET /v1/saml/{provider_id}/metadata`; `POST /v1/saml/{provider_id}/acs` (Assertion Consumer Service, HTTP-POST binding) validates signature against `saml_certificate`, checks audience/recipient/`NotOnOrAfter`, extracts NameID + attributes per mapping, performs JIT provisioning bound to `verified_domains`, creates a session.
- SP-initiated login: `GET /v1/saml/{provider_id}/login` → AuthnRequest (HTTP-Redirect). SLO endpoint for logout propagation.
- Replay protection: assertion `ID` cached in Redis until `NotOnOrAfter`.

**Testing**:
- `Unit (fixture signed assertion): valid → attributes extracted, session created`.
- `Unit: tampered/unsigned assertion → SamlValidationError`.
- `Unit: expired NotOnOrAfter → rejected`.
- `Unit: replayed assertion ID → rejected`.
- `Integration: JIT only for emails in verified_domains`.

#### 7.3 — SAML / OIDC outbound (platform as IdP) + custom claims in assertions

**What**: Let the platform issue SAML assertions to SP-type applications (`application_type='saml_sp'`).

**Design**:
- For `saml_sp` applications: issue signed SAML 2.0 assertions to `saml_acs_url` with `saml_name_id_format`, mapping identity claims + `applications.custom_claims` to SAML attributes; assertion signed with tenant key.
- Mirrors the OIDC custom-claims behaviour so both protocols expose the same claim set.

**Testing**:
- `Unit: issued assertion validates against tenant cert; NameID format honoured`.
- `Unit: custom_claims appear as SAML attributes`.
- `Integration: SP-initiated → AuthnRequest parsed → signed assertion POSTed to ACS`.

---

## Phase 8: AI-Native Risk Layer (Differentiator)

### Purpose
Implement the strategic moat from research.md/features.md: adaptive risk-based MFA, login anomaly detection, credential-compromise checks, and session-anomaly step-up — features concentrated in proprietary platforms with no open alternative. Built on the `login_attempts` and `ai_suggestions` tables, classical ML, and a privacy-preserving local-first design.

### Tasks

#### 8.1 — Risk signal extraction & scoring

**What**: Compute a per-attempt risk score from contextual signals.

**Design**:
- `domain/risk/signals.py`: from a login attempt + history derive `RiskSignals{new_device, unusual_location, velocity_exceeded, known_vpn, impossible_travel, credential_stuffing_pattern}` using `login_attempts` history (per identity/IP) and device fingerprint.
- `domain/risk/scorer.py`: `score(signals, history) -> float in [0,1]`. Baseline = weighted heuristic; optional `IsolationForest` trained per tenant on historical successful logins (`fit_tenant(tenant_id)` nightly Celery job). Score + signals written to `login_attempts.risk_score`/`risk_signals`.
- `tenant.mfa_policy='adaptive'`: thresholds → `allow (<0.4)`, `challenge MFA (0.4–0.8)`, `block (>0.8)`.

**Testing**:
- `Unit: login from new country shortly after another → impossible_travel true`.
- `Unit: many failures across identities from one IP → credential_stuffing_pattern true`.
- `Unit: adaptive policy at score 0.9 → block; 0.6 → challenge; 0.1 → allow`.
- `Integration: scorer writes risk_score + signals to login_attempts`.

#### 8.2 — Credential-compromise (breached password) check

**What**: Block or flag known-breached passwords without leaking them.

**Design**:
- `domain/risk/breached.py`: k-anonymity range query (HaveIBeenPwned-style: send first 5 chars of SHA-1, match suffix locally) via httpx; optional self-hosted offline list. Checked on registration, password change, and (configurably) login. On hit at login → force password reset; at set-password → `PolicyError("breached_password")`.
- Nightly Celery task refreshes the offline list when configured.

**Testing**:
- `Unit (mocked range API): known breached password → compromised true; only hash prefix sent`.
- `Unit: set_password with breached pw → PolicyError`.
- `Integration: login with later-breached pw → force-reset flag set`.

#### 8.3 — AI suggestions & session-anomaly step-up

**What**: Surface security recommendations and trigger step-up on anomalous active sessions.

**Design**:
- `domain/risk/` writes `ai_suggestions` rows (`adaptive_mfa, anomaly_detection, credential_compromise, session_anomaly, mfa_adoption, compliance_gap`) with `confidence`, `entity_type/id`, `suggestion_data`.
- API: `GET /v1/ai/suggestions?status=pending`, `POST /v1/ai/suggestions/{id}:accept|:dismiss`.
- Session-anomaly: a background check compares an active session's current request context to its origin; large deviation → write `session_anomaly` suggestion and (if adaptive) demote AAL forcing step-up on next sensitive action.

**Testing**:
- `Unit: high-risk login emits adaptive_mfa suggestion with confidence`.
- `Integration: accept suggestion → status='accepted', resolved_at set, audit row`.
- `Integration: session context shift → session_anomaly suggestion + AAL demotion`.

---

## Phase 9: Operations — Webhooks, SCIM, Audit Streaming, Email/SMS Delivery

### Purpose
Make the platform operable in production and integrable with enterprise systems: reliable webhook delivery for lifecycle events, SCIM 2.0 user provisioning, real-time audit-log streaming to SIEM, and asynchronous email/SMS delivery — all on Celery.

### Tasks

#### 9.1 — Webhooks with signed, retried delivery

**What**: Deliver lifecycle events to tenant-configured endpoints with HMAC signatures and retries.

**Design**:
- `domain/webhooks.py`: event types `identity.created/updated/deleted, session.created/revoked, mfa.enrolled, credential.compromised, login.blocked`. `emit(event_type, tenant_id, payload)` enqueues a Celery task per matching `webhooks` row.
- Delivery task: POST JSON with `X-HAP-Signature: t=<ts>,v1=<hmac_sha256(secret, ts.body)>`; exponential backoff (5 attempts); increments `failure_count`, sets `last_failure_at`/`last_success_at`; auto-disable after threshold.
- API: `POST/GET/DELETE /v1/webhooks`.

**Testing**:
- `Unit: signature header computed correctly; verifiable with signing_secret`.
- `Integration (mocked endpoint 500): retries with backoff, failure_count increments`.
- `Integration: identity.created emits to subscribed webhook only`.

#### 9.2 — SCIM 2.0 provisioning

**What**: RFC 7643/7644 `/scim/v2` endpoints for enterprise IdP-driven provisioning.

**Design**:
- `domain/scim/` + `api/v1/scim.py`: `GET/POST/PUT/PATCH/DELETE /scim/v2/Users`, `/scim/v2/Groups`, `/scim/v2/ServiceProviderConfig`, `/scim/v2/Schemas`. Bearer auth via `identity_providers.scim_bearer_token_ref`. User resource maps SCIM core schema ↔ `identities` (+ `external_id`, `provisioned_by`); Groups ↔ `roles`/`role_assignments`. PATCH supports RFC 7644 add/remove/replace operations.

**Testing**:
- `Integration: POST /scim/v2/Users → identity created with external_id, provisioned_by set`.
- `Integration: PATCH replace active=false → identity status suspended`.
- `Integration: DELETE /scim/v2/Users/{id} → soft delete`.
- `Unit: SCIM filter eq parsing → correct query`.
- `Integration: invalid SCIM bearer token → 401`.

#### 9.3 — Audit-log SIEM streaming, partition maintenance, email/SMS

**What**: Stream audit events to SIEM, maintain time-partitions, and back the delivery integrations.

**Design**:
- Celery beat: nightly `ensure_partition` for next month on `audit_log`/`login_attempts`; retention drop of partitions older than tenant policy.
- Audit streaming task: forward new `audit_log` rows to a configured sink (HTTP/webhook, syslog, or S3 batch) in a SIEM-friendly JSON shape; at-least-once with a cursor in Redis.
- `integrations/email.py` (SMTP + pluggable provider) and `sms.py` (pluggable gateway, Twilio-style) used by Phases 2/5/9.

**Testing**:
- `Integration: beat task creates next-month partition; old partition dropped per policy`.
- `Integration (mocked sink): new audit rows streamed once, cursor advances`.
- `Unit: email provider selection from config; SMS gateway ret/error handling`.

---

## Phase 10: RBAC, Multi-Tenant Admin, Hardening & Packaging

### Purpose
Complete authorization (RBAC), tenant/application/admin management APIs, security hardening (global rate limiting, RFC 9700 conformance audit), the exported OpenAPI spec + SDK generation, and the Docker deployment artifacts. After this phase the platform is deployable and self-serviceable.

### Tasks

#### 10.1 — RBAC: roles, assignments, permission claims

**What**: Manage roles/permissions and inject them into tokens.

**Design**:
- `domain/rbac.py`: CRUD `roles` (permissions TEXT[]), `assign_role`/`revoke_role` (`role_assignments`), `effective_permissions(identity_id)`. ID/access tokens optionally include `roles` and `permissions` claims (per application config).
- API: `/v1/rbac/roles` CRUD, `/v1/identities/{id}/roles` (assign/revoke).
- `default` roles auto-assigned on identity creation.

**Testing**:
- `Unit: effective_permissions unions assigned roles, dedup`.
- `Integration: assign role → access token carries permissions claim`.
- `Integration: expired role_assignment excluded from effective permissions`.

#### 10.2 — Tenant, application & admin-key management APIs

**What**: Self-service provisioning of tenants, applications, and identity providers.

**Design**:
- `/v1/tenants` (platform-admin), `/v1/applications` (tenant-admin: generates `client_id`, hashes `client_secret`, validates redirect URIs), `/v1/providers` (configure social/OIDC/SAML/LDAP/SCIM IdPs, domain verification challenge + check), admin API-key issuance with scopes.
- On tenant create: generate signing keypair via KeyProvider, store `jwks_private_key_ref`, seed default role and password policy.

**Testing**:
- `Integration: create tenant → JWKS ref stored, discovery doc resolves`.
- `Integration: create application → client_id unique, secret returned once then only hash stored`.
- `Integration: provider domain verification → verified_domains populated after challenge`.

#### 10.3 — Global rate limiting, security headers & RFC 9700 conformance

**What**: Cross-cutting hardening on all auth endpoints.

**Design**:
- `core/ratelimit.py`: Redis token-bucket middleware keyed by client_id+IP+endpoint class; stricter limits on `/oauth2/token`, OTP, password verify. Returns 429 + `Retry-After`.
- Security headers middleware; enforce HTTPS-only cookies; reject query-string bearer tokens (RFC 9700/6750); exact redirect-URI matching audit; disable implicit/ROPC grants globally.
- A conformance test suite asserting RFC 9700 mandates.

**Testing**:
- `Integration: exceed token-endpoint limit → 429 with Retry-After`.
- `Integration: bearer token in query string → 401`.
- `Integration: attempt response_type=token (implicit) → unsupported_response_type`.
- `Conformance: PKCE required, S256-only, exact redirect match all pass`.

#### 10.4 — OpenAPI export, SDK generation hooks, Docker packaging

**What**: Ship the spec and deployable artifacts.

**Design**:
- CI step exports `openapi/openapi.json` (OpenAPI 3.1) from FastAPI; a generation script invokes openapi-generator for TypeScript/Python/Go client stubs (README lists React/Node/Python/Go/TS SDK targets).
- `Dockerfile` (multi-stage, uv) + `docker-compose.yml` (app, postgres, redis, celery-worker, celery-beat); `alembic upgrade head` entrypoint hook; `.env.example` documenting all `HAP_*` vars.

**Testing**:
- `Integration: exported openapi.json validates against OpenAPI 3.1 schema; every router path present`.
- `Integration: docker-compose up → /readyz 200 after migrations`.
- `E2E (compose): full register → passkey enrol → authorize → token → userinfo against the running container`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (config, schema, crypto, context)   ─── required by everything
    │
Phase 2: Identities & Password Credentials              ─── requires 1
    │
Phase 3: OAuth2/OIDC Authorization Server (core value)  ─── requires 1, 2
    │
Phase 4: Sessions, Step-Up & AAL                        ─── requires 3
    ├── Phase 5: MFA (TOTP, Email/SMS OTP)               ─── requires 4 ─┐ can parallel
    ├── Phase 6: WebAuthn / Passkeys                     ─── requires 4 ─┤ with each
    └── Phase 7: Federation (Social/OIDC/SAML)           ─── requires 4 ─┘ other
         │
Phase 8: AI-Native Risk Layer                           ─── requires 2,4 (uses login_attempts); richer with 5–7
    │
Phase 9: Operations (Webhooks, SCIM, Audit, Email/SMS)  ─── requires 1,2; SCIM richer with 10.1; can parallel with 8
    │
Phase 10: RBAC, Admin APIs, Hardening, Packaging        ─── requires all; 10.3/10.4 are final hardening/release
```

**Parallelism opportunities:**
- Phases **5, 6, 7** can be developed concurrently once Phase 4 lands — they are independent authentication-method modules sharing only the session/AAL machinery.
- Phase **9** (operations) can proceed in parallel with Phase **8** (risk layer); both depend only on Phases 1–2 plus the events emitted by 3–7.
- Within Phase 10, **10.1 (RBAC)** can start as early as Phase 4; **10.3/10.4** are release-gating and run last.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pytest`), including the named edge cases.
3. Integration tests run against **real** Postgres and Redis via testcontainers (not just mocks).
4. `ruff check` and `ruff format --check` pass; `mypy --strict` passes on changed modules.
5. Alembic migration(s) created, `upgrade head` and `downgrade` both succeed cleanly.
6. The feature works end-to-end through the HTTP API (demonstrated by at least one integration or e2e test).
7. New endpoints appear in the auto-generated OpenAPI 3.1 spec with correct request/response schemas and security requirements.
8. New configuration options are documented in `.env.example` and `README.md`.
9. Every mutating operation writes an `audit_log` row.
10. Relevant standards are honoured and asserted in tests where applicable (PKCE/S256, exact redirect match, JWT alg allow-list, WebAuthn sign-count, SAML signature/replay, SCIM schema).
11. `docker-compose up` still yields a healthy `/readyz` after the phase's migrations (from Phase 1 onward).
```
