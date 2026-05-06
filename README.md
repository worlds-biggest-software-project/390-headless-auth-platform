# Headless Auth Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source identity engine that delivers SSO, MFA, social login, and session management as APIs — leaving the UI entirely to you.

Headless Auth Platform is an authentication-as-a-service backend with no opinionated frontend. It is for product teams that want to build fully custom login experiences while delegating the security-critical logic — token issuance, session management, MFA orchestration, social provider federation, and SSO — to a managed or self-hosted identity engine.

---

## Why Headless Auth Platform?

- **Rolling your own auth is dangerous.** Custom auth systems routinely ship with credential stuffing, session fixation, token leakage, and MFA bypass vulnerabilities that are well-understood but easy to get wrong.
- **Incumbents force a UI/cost trade-off.** Auth0 and Clerk provide strong DX but lock customers into proprietary SDKs and per-MAU pricing that scales painfully for enterprise deployments.
- **Open-source options are fragmented.** Ory is genuinely headless but ships no UI; Keycloak depends on Freemarker themes; ZITADEL moved its self-hosted licence from Apache 2.0 to AGPL 3.0 in 2025.
- **Passkeys are now table-stakes.** Major consumer platforms completed FIDO2/WebAuthn rollouts in 2025–2026, and most existing open-source auth servers still treat passkeys as a bolt-on.
- **AI-native risk signals are missing.** Adaptive MFA, anomaly detection, and credential-compromise checks are concentrated in proprietary platforms; an open alternative does not yet exist.

---

## Key Features

### Protocols and Federation

- OpenID Connect (OIDC) and OAuth 2.0 with full protocol compliance
- SAML 2.0 for enterprise SSO and federation with customer IdPs
- Social login across Google, GitHub, Apple, Microsoft, and other major providers
- LDAP / Active Directory federation for legacy environments
- SCIM for user provisioning and deprovisioning

### Authentication Methods

- WebAuthn / FIDO2 passkey support as a first-class authentication method
- TOTP, email OTP, and SMS OTP for multi-factor authentication
- Password authentication with optional passwordless-only mode
- Email verification and password reset flows
- Step-up authentication for sensitive operations

### Sessions and Tokens

- JWT-based session tokens with verifiable claims
- Server-side session management with remote revocation
- Refresh token rotation and configurable session lifetimes
- Custom claims in ID tokens and SAML assertions

### Multi-Tenancy and Authorisation

- Multi-tenant model with per-tenant branding and security policies
- Role-based access control (RBAC) for authorisation
- Per-tenant SAML / OIDC IdP configuration with domain verification and JIT provisioning
- User management API (CRUD, claims retrieval)

### Operations and Compliance

- Audit logging for SOC 2, ISO 27001, HIPAA, and PCI-DSS baselines
- Real-time audit log streaming to SIEM platforms
- Credential migration tooling, including bcrypt hash verification and on-demand re-hashing
- Webhooks for user lifecycle events

---

## AI-Native Advantage

The research identifies adaptive MFA, anomaly detection, and credential-compromise checks as features concentrated in proprietary platforms. Headless Auth Platform targets this gap with risk-based authentication driven by contextual signals (device, location, login velocity), session anomaly detection that triggers step-up authentication on unusual characteristics, and real-time cross-referencing of login attempts against known breached password lists. Phishing-resistant recovery flows can detect and block account-recovery requests that resemble attacker patterns without requiring developer instrumentation.

---

## Tech Stack & Deployment

The platform is API-first: every capability is exposed as a typed REST or gRPC endpoint so that applications can render fully custom frontends. SDKs target React, Node.js, Python, Go, and TypeScript. Both managed (cloud) and self-hosted deployment modes are supported, mirroring the operating model of ZITADEL and Ory. The platform aligns with open standards — OIDC, OAuth 2.0, SAML 2.0, WebAuthn / FIDO2, and SCIM — none of which are encumbered by active patents.

---

## Market Context

The customer identity and access management (CIAM) and developer-focused auth market spans commercial SaaS (Auth0, Clerk, Descope) and open-source platforms (ZITADEL, Ory Kratos + Hydra, Keycloak, Hanko, Supabase Auth). Pricing on commercial platforms scales with monthly active users (MAU), and enterprise deployments can be expensive; open-source alternatives offer portability but require more engineering investment. Primary buyers are application engineering teams, security engineering teams, and platform teams at companies that need enterprise SSO without vendor lock-in.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
