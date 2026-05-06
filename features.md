# Headless Auth Platform — Feature & Functionality Survey

> Candidate #390 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Auth0 (Okta) | Commercial SaaS | Proprietary | https://auth0.com/ |
| Clerk | Commercial SaaS | Proprietary | https://clerk.com/ |
| Descope | Commercial SaaS | Proprietary | https://www.descope.com/ |
| ZITADEL | Open Source / SaaS | AGPL 3.0 (self-hosted), Proprietary (SaaS) | https://zitadel.com/ |
| Ory (Kratos + Hydra) | Open Source | Apache 2.0 | https://github.com/ory/kratos |
| Keycloak | Open Source | Apache 2.0 | https://www.keycloak.org/ |
| Hanko | Open Source | MIT | https://www.hanko.io/ |
| Supabase Auth | Open Source / SaaS | MIT (PostgreSQL extension) | https://supabase.com/auth |

## Feature Analysis by Solution

### Auth0 (Okta)

**Core features**
- Universal Login experience with customizable branding and themes
- SAML 2.0, OpenID Connect (OIDC), and OAuth 2.0 support with full protocol compliance
- Social login via Google, Facebook, Apple, GitHub, LinkedIn, and 25+ providers
- Multi-factor authentication: TOTP, SMS, email, push notifications, and WebAuthn/passkeys
- Role-based access control (RBAC) and attribute-based access control (ABAC)
- User provisioning via SCIM
- Adaptive MFA with risk-based authentication (device posture, location, login velocity)
- Session management with remote revocation and refresh token rotation
- Comprehensive audit logging and compliance reporting

**Differentiating features**
- Adaptive MFA using contextual signals (device, location, geographic anomalies) reduces friction for legitimate users while blocking suspicious activity
- Broad integrations: 7,000+ third-party applications with pre-built connectors
- Enterprise-grade multi-tenancy with per-tenant IdP configuration
- Okta's extensive identity ecosystem (Workforce Identity Cloud, Customer Identity Cloud) provides unified management across workforce and customer identities

**UX patterns**
- Universal Login provides a white-labeled, OAuth-compliant login page requiring minimal frontend code
- Dashboard-centric configuration for enterprise operators
- Gradual policy enforcement: admins can monitor, log, and enforce MFA adoption without disruption
- SDK-driven integration via libraries for web, mobile, and backend

**Integration points**
- Auth0 Rules engine for custom logic execution during authentication
- SCIM for user provisioning and deprovisioning
- Webhooks for event-driven integrations
- Management API for programmatic tenant and policy management
- SDKs for React, Angular, Vue, Swift, Android, Node.js, and others
- Pre-built connectors to Slack, Salesforce, Jira, and 100+ SaaS applications

**Known gaps**
- No-code flow builders are unavailable; all customisation requires code
- Cost scales with MAU (monthly active users); enterprise deployments can be expensive
- Vendor lock-in is high; portable OIDC compliance exists but vendor-specific features are pervasive

**Licence / IP notes**
- Proprietary commercial service; SDKs are proprietary but some wrapper libraries are open source (Apache 2.0)

---

### Clerk

**Core features**
- Pre-built React and Next.js components for login, signup, user management, and MFA enrolment
- Session management using hybrid JWT + cookie approach (short-lived tokens with automatic background refresh)
- Social login (Google, GitHub, Apple, Discord, Twitter, LinkedIn, etc.)
- TOTP-based MFA and WebAuthn/passkey support (added 2026)
- Email and SMS verification
- Multi-tenant applications with per-user organisation and permission management
- Backend session tokens for API authentication
- User impersonation for development and support
- Developer-friendly CLI and dashboard

**Differentiating features**
- Pre-built components eliminate most frontend authentication UI work; developers rarely need to build login forms from scratch
- Hybrid JWT + cookie session approach combines the statelessness of JWTs with the security of server-set cookies, avoiding many session management pitfalls
- Seamless Next.js integration: middleware support, edge function compatibility, App Router native
- Session token automatic refresh in the background eliminates token expiration UX friction
- Free tier suitable for small projects; transparent per-user pricing scales smoothly

**UX patterns**
- Component-first: <SignIn />, <SignUp />, <UserProfile />, etc. with minimal props
- Middleware-based: session validation and user context propagate through Next.js middleware
- Progressive enhancement: start with pre-built components, then customize CSS, branding, and logic
- Backend session API: retrieve user info from backend with a single API call

**Integration points**
- Clerk CLI (clerk) for local development, environment variable management, and deployment
- GitHub Actions for CI/CD integration
- Next.js middleware and API routes
- SDKs for Remix, SvelteKit, Node.js, Python, and Ruby (growing ecosystem)
- Webhooks for sync to external systems (database, CRM, analytics)
- OIDC federation (apps can request claims from Clerk to bootstrap their own auth)

**Known gaps**
- Tightly coupled to frontend frameworks; backend-only applications require custom UI
- No self-hosted option; fully managed SaaS only
- Limited enterprise SSO; SAML/OIDC federation is available but less mature than Auth0
- Organisation management is different from traditional RBAC; users can belong to multiple organisations but role definitions are per-organisation, not global

**Licence / IP notes**
- Proprietary commercial service; SDKs are proprietary

---

### Descope

**Core features**
- No-code Flow Builder: drag-and-drop journey designer for login, registration, MFA, SSO, and step-up workflows
- SAML 2.0 and OIDC support for enterprise SSO
- Multi-factor authentication: TOTP, SMS, email, Magic Links, passkeys, and push notifications
- Step-up authentication for sensitive operations (e.g., account changes, payment confirmation)
- Risk-based adaptive authentication (flags risky logins, optionally triggers additional verification)
- User management dashboard with bulk operations
- Audit logging and compliance reporting
- Social login via Google, GitHub, Apple, LinkedIn, Discord, etc.

**Differentiating features**
- No-code Flow Builder is the unique standout: non-technical product teams and security engineers can modify authentication flows without developer involvement or code deploy
- Step-up authentication is first-class: easily insert additional verification at any point in a flow (e.g., before sensitive actions)
- Drag-and-drop SAML/OIDC IdP configuration for enterprises without needing IT or developers
- Risk-based authentication uses Descope's intelligence signals; administrators can configure response (block, challenge, or allow)

**UX patterns**
- Flow Builder is visual: drag screens, conditions, and actions; real-time preview of changes
- Admin-first: product teams and security teams can own authentication without developer context switching
- Progressive gates: MFA can be optional, enforced after a grace period, or required immediately
- Self-service SSO portal: enterprise customers configure their own SAML/OIDC integration without support tickets

**Integration points**
- REST API for custom frontends
- SDKs for React, Node.js, Python, and others
- Webhooks for user lifecycle events
- SAML and OIDC federation with customer IdPs
- Identity provider connectors (Okta, Azure AD, Google Workspace, etc.)

**Known gaps**
- More limited developer ecosystem than Auth0 or Clerk
- No password history or complexity policy configuration (customers request this for HIPAA)
- Limited on-premises deployment options; primarily SaaS

**Licence / IP notes**
- Proprietary commercial service

---

### ZITADEL

**Core features**
- API-first identity platform with complete REST and gRPC APIs for all operations
- OIDC and OAuth 2.0 with OpenID Foundation certification
- SAML 2.0 support for enterprise federation
- Multi-tenancy with Instance > Organisation > Project > Application hierarchy
- RBAC and fine-grained authorisation policies
- WebAuthn/passkey support (passwordless)
- TOTP, SMS, and email-based MFA
- Session management with configurable lifecycle
- Event-sourced audit trail for full compliance auditability
- Identity brokering (delegate to external IdPs like Okta, Azure AD)

**Differentiating features**
- API-first design: every capability is exposed as a typed API, enabling fine-grained automation and custom frontend development
- Event-sourced architecture: every identity state change is an immutable event, enabling complete audit trails and time-travel debugging
- Multi-tenancy built from the ground up: isolation, per-tenant branding, and per-tenant security policies without operational overhead
- Managed SaaS and self-hosted options; customers can choose control or convenience

**UX patterns**
- Console UI for configuration, but developers interact primarily via APIs
- API-driven flow: fetch authentication requirements from ZITADEL, render custom frontend, post credentials to ZITADEL for verification
- Event subscription: applications can subscribe to identity events (user created, session expired) for real-time sync
- Multi-tier tenancy: enterprises manage their own organisations and delegate to teams

**Integration points**
- gRPC and REST APIs (OpenAPI documented)
- OIDC for delegating authentication to other systems
- SAML for federation with enterprise IdPs
- SCIM for user provisioning
- Webhooks for event-driven integrations
- SDKs for Go, Node.js, Python, and others

**Known gaps**
- Licensing change (Apache 2.0 → AGPL 3.0 in 2025) requires disclosure of modifications; commercial license available
- Smaller ecosystem and community than Keycloak or Auth0
- More complex to deploy and operate; requires care in architecture and scaling decisions

**Licence / IP notes**
- Self-hosted: AGPL 3.0 (copyleft; modifications must be disclosed); commercial license available
- SaaS: Proprietary managed service

---

### Ory (Kratos + Hydra)

**Core features**
- **Ory Kratos**: Headless identity and credential management (login, registration, recovery, verification, profile management)
- **Ory Hydra**: OpenID Certified OpenID Connect and OAuth 2.0 provider (token issuance, claims generation)
- Together: full-featured headless identity platform
- Microservices architecture: Kratos and Hydra are independent, composable components
- WebAuthn/passkeys and passwordless magic links
- Social login (Google, GitHub, LinkedIn, Discord, etc.)
- Multi-MFA: TOTP, SMS, email
- Session and token management with configurable lifetimes
- Identity schemas for custom user attributes
- Audit logging and event subscription

**Differentiating features**
- Microservices-first: Kratos and Hydra can be deployed independently, scaled separately, and composed with other identity systems
- Headless by design: Kratos deliberately excludes UI templating, forcing explicit separation of identity logic from presentation
- Certified OIDC compliance: Hydra passes OpenID Foundation conformance tests (higher bar than self-claimed compatibility)
- Migration use case: Hydra can be placed in front of existing user databases, allowing gradual migration without forcing full adoption

**UX patterns**
- API-driven flows: frontend fetches form schema from Kratos, renders custom UI, posts credentials to Kratos for verification
- Event-driven: identity systems can subscribe to Kratos events (user registered, password changed) and react
- Composable: teams deploy Hydra for token issuance while keeping legacy user management in their system

**Integration points**
- RESTful APIs for Kratos (login, registration, session management)
- gRPC and REST for Hydra (token exchange, claims configuration)
- Webhooks for identity events
- SDKs for Go, Node.js, Python, TypeScript
- Identity providers (GitHub, Google, etc.) via OAuth federation
- Session hooks for custom logic during login/logout

**Known gaps**
- UI is customer's responsibility; no pre-built components or themes (unlike Clerk or Descope)
- Smaller documentation and community than Keycloak
- OIDC certification applies only to Hydra; Kratos is identity management (user federation must come from elsewhere)

**Licence / IP notes**
- Open source: Apache 2.0 (permissive; no copyleft requirements)
- Commercial managed service available via Ory Network

---

### Keycloak

**Core features**
- SAML 2.0, OpenID Connect (OIDC), and OAuth 2.0 support
- User federation: LDAP/Active Directory, custom user stores
- Social login via Google, Facebook, GitHub, LinkedIn, Twitch, etc.
- TOTP and WebAuthn-based MFA
- Realm-based multi-tenancy (each realm is an isolated identity domain)
- RBAC with fine-grained authorization policies
- User attribute mappers for custom claims and SAML attributes
- Account management console for users to self-manage profiles and security
- Session and offline token management
- Audit logging with event listeners
- Theme customisation (Freemarker templates)

**Differentiating features**
- LDAP/Active Directory federation: legacy environments can use Keycloak as an OIDC/SAML bridge without migrating user data
- Attribute mappers: flexible claim and assertion generation for SAML and OIDC (supports if-then logic, script execution)
- Realm-based isolation: each realm has independent configuration, policies, and user federation
- Strong open-source community: widely deployed in enterprises; strong commercial ecosystem (Red Hat, JBoss support)

**UX patterns**
- Admin console for realm and identity provider configuration
- Account console (User Account Management) for self-service user operations
- Theme customisation via Freemarker templates for branded login screens
- Policy-driven access control: permission evaluation via realms and client roles

**Integration points**
- OIDC and SAML for delegating authentication to Keycloak
- LDAP/AD for federation (read-only or sync-capable)
- User Federation SPI for custom user store plugins
- Event listeners for custom logic during login, user creation, etc.
- REST Management API for programmatic administration
- Realms for isolated multi-tenancy

**Known gaps**
- Custom UI requires Freemarker templates; no pre-built component library
- Clustering and scaling require careful operational planning (shared database, distributed caches)
- LDAP/AD federation is read-heavy; write-back (user provisioning to LDAP) is limited

**Licence / IP notes**
- Open source: Apache 2.0 (permissive)
- Commercial support via Red Hat, JBoss, and community partners

---

### Hanko

**Core features**
- Passkey-first authentication using WebAuthn/FIDO2
- TOTP-based MFA as a second factor
- Social login (Apple, Google, GitHub) via OAuth federation
- SAML 2.0 SSO for enterprise integration
- Server-side session management with session cookies
- Password option (can be disabled for passwordless-only mode)
- User account management UI (profile, security settings)
- REST API for custom frontends
- Email verification and recovery flows
- Framework-agnostic: works with any frontend or backend

**Differentiating features**
- Passkey-first philosophy: emphasises WebAuthn/FIDO2 as the primary authentication method, reducing password management burden
- Privacy-first design: minimal data collection (no analytics or tracking)
- Server-side sessions: avoids JWT token management complexity; sessions are server-stored and stateless to the client
- SAML + passkeys: enterprise-grade SSO with passkey support is rare; Hanko bridges the gap

**UX patterns**
- Passkey registration during signup or later in account settings
- Fallback to passwords (optional) for users without passkey devices
- Server-side sessions: clients receive session cookies; backend validates sessions server-side
- Multi-provider: users can link passkey, passwords, and social login simultaneously

**Integration points**
- REST API for custom frontends
- SDKs for React, Vue, and vanilla JavaScript
- SAML identity provider for federation with enterprise IdPs
- OAuth social logins (Apple, Google, GitHub, custom providers)
- Backend API for session validation and user management
- Webhooks for user lifecycle events

**Known gaps**
- Limited RBAC/ABAC capabilities; primarily focuses on authentication
- Smaller ecosystem and community compared to Keycloak or Ory
- Self-hosting can be operationally complex (database, caching, scaling)

**Licence / IP notes**
- Open source: MIT (permissive)
- Managed service (Hanko Cloud) available as commercial option

---

### Supabase Auth

**Core features**
- Social login (Google, Facebook, GitHub, Azure, GitLab, Twitter, Discord, etc.)
- SAML and Azure AD support for enterprise SSO
- Email and password authentication
- Email and SMS OTP (One-Time Password) for passwordless flows
- Row-Level Security (RLS) integration with PostgreSQL
- JWT-based session tokens
- User management API
- Phone authentication (Twilio integration)
- Custom SMTP configuration for branded emails
- OAuth provider capability (Supabase apps can act as OAuth2 servers)

**Differentiating features**
- Native PostgreSQL integration: RLS policies bind authentication to database-level authorisation; row visibility is enforced at the database layer, not application layer
- No external identity table: user data lives in Postgres; Auth seamlessly integrates with application data
- JWT tokens are verifiable against Postgres public keys; backend services can validate tokens without calling Auth service
- Email/SMS templates are customisable and can be self-hosted

**UX patterns**
- SDK-driven (JavaScript/TypeScript): `supabase.auth.signUp()`, `supabase.auth.signIn()` handle frontend authentication
- RLS policies: SQL-based row filtering enforces data access (e.g., `SELECT * FROM todos WHERE user_id = auth.uid()`)
- Real-time subscriptions: authenticated clients can subscribe to Postgres change events via WebSockets
- Postgres triggers can react to auth events (e.g., `AFTER INSERT ON auth.users` trigger for user onboarding)

**Integration points**
- JavaScript/TypeScript SDK for frontend and backend
- PostgreSQL RLS for per-row data access control
- Webhooks for user lifecycle events
- Custom SMTP for email templates
- Phone authentication via Twilio (or self-hosted SMS gateway)
- SAML and Azure AD for enterprise SSO
- Postgres functions can call Auth API for custom logic

**Known gaps**
- No self-hosted option for Auth (managed service only in Supabase Cloud)
- Limited RBAC/ABAC relative to Auth0 or Keycloak; relies on RLS for authorisation
- Social OAuth provider list is large but some niche providers may not be supported
- SAML configuration requires custom setup; no self-service admin interface

**Licence / IP notes**
- Proprietary managed service; underlying Postgres extension is not separately licensed
- GoTrue (Supabase Auth backend) is open source (MIT) but not commonly self-hosted

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

Every headless auth platform must support:

- **Social login**: Google, GitHub, and at least 5 other providers with minimal configuration
- **OIDC/OAuth2 compliance**: applications can receive access and ID tokens per OIDC spec; token formats are verifiable (JWTs)
- **Session management**: application can establish and revoke user sessions; tokens can be refreshed or rotated
- **Email verification**: confirm user email addresses to prevent spam account registration
- **MFA**: at minimum TOTP; SMS and email OTP are becoming expected
- **User management API**: create, read, update, delete users; fetch user metadata and claims
- **Audit logging**: record identity events (login, logout, MFA enrolment, password change) for compliance
- **Claim customisation**: applications can define custom claims in ID tokens and SAML assertions

### Differentiating Features

Platforms compete on:

- **WebAuthn/Passkey support**: increasingly table-stakes; platforms differ in maturity and ease of use
- **Adaptive/Risk-based MFA**: Auth0 and Descope offer contextual signals; most open-source platforms require custom logic
- **No-code flow builders**: Descope's drag-and-drop interface vs. code-driven configuration (Ory, Keycloak)
- **Pre-built UI components**: Clerk dominates with polished React components; others require custom frontend
- **Multi-tenancy model**: ZITADEL and Keycloak have strong multi-tenancy; others treat it as an add-on
- **LDAP/AD federation**: Keycloak is the gold standard; Ory and others can integrate but with more friction

### Underserved Areas / Opportunities

- **AI-powered risk scoring**: Behavioural anomaly detection during login (e.g., unusual time, new device, geographic impossibility) could be trained on platform aggregated signals without violating user privacy
- **Passwordless-by-default**: Most platforms offer passkeys as an option; a platform that defaults to passkeys and uses passwords as a fallback could lead adoption and reduce password attack surface
- **Compliance-first configuration**: Pre-built compliance templates (SOC 2, HIPAA, PCI-DSS) could auto-configure audit logging, MFA policies, and session lifetimes
- **Credential migration automation**: Most platforms require custom scripting for bcrypt hash migration; a platform with built-in hash verification and on-demand re-hashing could simplify legacy system replacement
- **AI-driven anomaly detection**: Detecting account takeover (ATO) via login patterns, device fingerprints, and velocity checks without requiring developer instrumentation
- **Zero-knowledge proofs for privacy-preserving verification**: Prove identity claims without leaking underlying attributes (e.g., prove age > 18 without revealing birthdate)

### AI-Augmentation Candidates

- **Dynamic MFA strength**: Machine learning models trained on organisational attack patterns (credential stuffing, ATO attempts) could dynamically adjust MFA requirements (strength, frequency) based on risk
- **Phishing-resistant recovery flows**: Detect and block account recovery requests that resemble attacker patterns (e.g., recovery from unusual location, without recent logins)
- **Credential compromise detection**: Cross-reference login attempts against known breached password lists in real-time; prompt users to change compromised passwords
- **Session anomaly detection**: Flag sessions with unusual characteristics (device, location, behaviour patterns) and trigger step-up authentication
- **Identity proofing**: Automated KYC (Know Your Customer) verification using document scanning and liveness detection for regulated use cases

## Legal & IP Summary

**Licensing diversity is high**: commercial platforms (Auth0, Clerk, Descope) are proprietary; open-source platforms span Apache 2.0 (Ory, Keycloak, Hanko), MIT (Hanko, Supabase), and AGPL 3.0 (ZITADEL self-hosted). **Vendor lock-in trade-off**: commercial platforms provide superior DX (pre-built components, flow builders) but create tight coupling; open-source platforms offer portability but require more engineering investment. **Compliance burden**: SOC 2, ISO 27001, HIPAA, and PCI-DSS all mandate audit logging, MFA, and encryption in transit/at-rest. Platforms differ in pre-built compliance templates and audit log retention. **No active patents** on core OIDC, SAML, or WebAuthn techniques were identified; standards are open and royalty-free. **Credential migration risk**: Platforms vary in built-in support for legacy bcrypt hash verification and on-demand rehashing. Custom migration logic is often required. **Recommendation**: Conduct vendor lock-in risk assessment and compliance requirement mapping before selecting a platform. OIDC/SAML compliance provides some portability, but vendor-specific extensions (Clerk components, Auth0 Rules, Descope Flows) may require refactoring during migration.

## Recommended Feature Scope

Based on the above, a competitive headless auth platform should prioritise:

**Must-have (MVP)**
- OIDC/OAuth2 and SAML 2.0 support with full protocol compliance
- Social login (Google, GitHub, Apple, Microsoft, at least 5 providers)
- Email verification and password reset flows
- Multi-factor authentication (TOTP, email OTP, SMS OTP)
- Session management with token refresh and revocation
- User management API (CRUD, claims retrieval)
- Audit logging for compliance (SOC 2, HIPAA baseline)
- Custom claims in ID tokens and SAML assertions
- WebAuthn/passkey support (passwordless authentication)

**Should-have (v1.1)**
- Adaptive/risk-based MFA (contextual signals: device, location, login velocity)
- LDAP/Active Directory federation for legacy environments
- SCIM for user provisioning and deprovisioning
- No-code flow builder (or visual policy configuration)
- Pre-built UI components (React, Vue, or framework-agnostic)
- Multi-tenancy with per-tenant branding and security policies
- Server-side session management (in addition to JWT)
- Credential migration tooling (bcrypt hash verification and re-hashing)
- Real-time audit log streaming to SIEM and compliance platforms

**Nice-to-have (backlog)**
- Step-up authentication (additional verification before sensitive actions)
- Device fingerprinting and anomaly detection for account takeover prevention
- Zero-knowledge proofs for privacy-preserving identity claims
- AI-powered behavioural analysis for risk scoring
- FIDO2 security key support (multi-device attestation)
- Biometric authentication (native app support)
- Decentralised identity (DID, verifiable credentials) support
- Push notification MFA
