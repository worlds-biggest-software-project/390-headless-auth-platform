# Standards & API Reference

> Project: Headless Auth Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### OASIS Standards

**Security Assertion Markup Language (SAML) 2.0**
- URL: https://www.oasis-open.org/standard/saml/
- Technical Overview: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
- Ratified as an OASIS Standard in March 2005. Defines the XML-based format for exchanging authentication and authorisation assertions between identity providers (IdP) and service providers (SP). Mandatory for enterprise SSO integration; the Web Browser SSO Profile is the most widely deployed variant. SAML 2.0 metadata (for endpoint and key exchange) and bindings (HTTP-POST, HTTP-Redirect) are required companion specifications for full interoperability.

### W3C & IETF Standards

**W3C Web Authentication (WebAuthn) Level 2**
- URL: https://www.w3.org/TR/webauthn-2/
- Specifications reference: https://passkeys.dev/docs/reference/specs/
- W3C Recommendation defining the browser API and ceremony for public-key-based authenticator registration and assertion. WebAuthn Level 2 (April 2021) adds resident credentials (passkeys), user verification enforcement, and enhanced attestation options. Required for passkey and FIDO2 hardware security key support. Paired with the FIDO Alliance's Client to Authenticator Protocol (CTAP2) to form the complete FIDO2 stack.

**FIDO Alliance — FIDO2 / CTAP2**
- URL: https://fidoalliance.org/specifications/
- The FIDO2 specification suite combines W3C WebAuthn with the FIDO Client to Authenticator Protocol (CTAP). CTAP2 handles communication between the browser or platform and external authenticators over USB, NFC, and Bluetooth. Any headless auth platform offering passkey or hardware key support must implement the full FIDO2 stack (WebAuthn + CTAP2) and may seek FIDO Alliance certification.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The foundational authorisation framework. Defines roles (resource owner, client, authorisation server, resource server), grant types (authorisation code, client credentials, device), token formats, and the authorisation code flow. All modern auth platforms build on this foundation; subsequent RFCs extend and harden it.

**RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage**
- URL: https://www.rfc-editor.org/rfc/rfc6750
- Defines how bearer tokens are transmitted in HTTP requests (Authorization header, form body, query parameter). The Authorization header method is required; query parameter usage is deprecated by RFC 9700.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Specifications index: https://openid.net/developers/specs/
- An identity layer built on OAuth 2.0 that adds the ID token (a signed JWT containing user claims), the UserInfo endpoint, and standardised claim names. Any platform acting as an identity provider must implement OIDC Core to interoperate with third-party relying parties. OpenID Foundation certification is a credible conformance signal.

**OpenID Connect Discovery 1.0**
- URL: https://openid.net/specs/openid-connect-discovery-1_0.html
- Defines the `/.well-known/openid-configuration` well-known metadata endpoint through which clients auto-discover authorisation, token, userinfo, and JWKS endpoint URLs, supported grant types, and signing algorithms. Essential for zero-configuration client integration.

**RFC 8414 — OAuth 2.0 Authorization Server Metadata**
- URL: https://www.rfc-editor.org/rfc/rfc8414.html
- Extends OIDC Discovery to non-OIDC OAuth 2.0 servers via `/.well-known/oauth-authorization-server`. Allows clients to discover server capabilities (supported response types, PKCE methods, token endpoint auth methods) without manual configuration. Referenced in RFC 9700 as a requirement.

**RFC 7636 — Proof Key for Code Exchange (PKCE)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Defines the `code_challenge` / `code_verifier` mechanism that prevents authorisation code interception attacks. Originally designed for native apps, RFC 9700 mandates PKCE for all clients using the authorisation code flow. Servers must reject authorisation code requests without a code challenge in compliant deployments.

**RFC 9700 — OAuth 2.0 Security Best Current Practice**
- URL: https://www.rfc-editor.org/rfc/rfc9700.html
- Consolidates security requirements for OAuth 2.0 clients and servers, mandating PKCE universally, requiring exact-match redirect URI comparison, deprecating implicit and ROPC grants, and restricting bearer token usage in query strings. The definitive security reference for any new OAuth 2.0 / OIDC implementation.

**OAuth 2.1 (IETF Draft)**
- URL: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- Overview: https://oauth.net/2.1/
- A consolidation draft incorporating the security improvements from RFC 9700 and related RFCs into a single updated OAuth core document. Removes the implicit grant, resource owner password credentials grant, and bearer token query-parameter usage. Authorisation servers targeting long-term compliance should track this draft.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Defines the compact, URL-safe format for representing claims as a signed or encrypted JSON object. JWTs are used for ID tokens (OIDC), access tokens, and session tokens across the industry. Auth platforms must implement JWT creation, signing (JWS), optional encryption (JWE), and signature verification.

**RFC 7517 — JSON Web Key (JWK) and RFC 7518 — JSON Web Algorithms (JWA)**
- JWK URL: https://datatracker.ietf.org/doc/html/rfc7517
- JWA URL: https://datatracker.ietf.org/doc/html/rfc7518
- JWK defines the JSON representation of cryptographic keys; JWA enumerates the signing and encryption algorithms (RS256, ES256, HS256, etc.). The JWKS endpoint (`/jwks.json`) uses JWK format to publish public keys for token signature verification. Together with RFC 7515 (JWS) and RFC 7516 (JWE), these form the JOSE (JavaScript Object Signing and Encryption) suite.

**RFC 7662 — OAuth 2.0 Token Introspection**
- URL: https://datatracker.ietf.org/doc/html/rfc7662
- Defines the `/introspect` endpoint that resource servers query to determine if an opaque or structured token is active and retrieve its metadata (scope, expiry, subject). Required when resource servers cannot or should not validate JWTs locally — common in microservice architectures and gateway-based deployments.

**RFC 7009 — OAuth 2.0 Token Revocation**
- URL: https://datatracker.ietf.org/doc/html/rfc7009
- Defines the `/revoke` endpoint for clients to invalidate access tokens and refresh tokens. Mandatory for logout flows that must propagate session termination to the authorisation server and prevent refresh token reuse after logout.

**RFC 7643 & RFC 7644 — SCIM 2.0 (System for Cross-domain Identity Management)**
- Core Schema: https://www.rfc-editor.org/rfc/rfc7643.html
- Protocol: https://datatracker.ietf.org/doc/html/rfc7644
- Reference: https://scim.cloud/
- Defines an HTTP-based protocol for automating user provisioning and deprovisioning between identity providers and applications. RFC 7643 specifies the core User and Group schema; RFC 7644 defines the REST protocol (GET, POST, PUT, PATCH, DELETE). Required for enterprise integrations with identity management systems such as Okta, Azure AD, and OneLogin that provision users into downstream SaaS.

**RFC 8693 — OAuth 2.0 Token Exchange**
- URL: https://datatracker.ietf.org/doc/html/rfc8693
- Defines a grant type for exchanging one token for another (e.g., impersonation, delegation, act-as). Relevant for multi-service architectures where service-to-service calls must propagate user context or for implementing step-up authentication flows.

**RFC 8628 — OAuth 2.0 Device Authorization Grant**
- URL: https://datatracker.ietf.org/doc/html/rfc8628
- Defines the device code flow for devices without a browser (smart TVs, CLI tools, IoT). Increasingly relevant as headless auth platforms serve non-browser clients. The flow polls the token endpoint after user approval on a secondary device.

**RFC 7522 — SAML 2.0 Profile for OAuth 2.0**
- URL: https://datatracker.ietf.org/doc/html/rfc7522
- Defines how SAML 2.0 assertions can be used as OAuth 2.0 grant types. Enables interoperability between SAML-based enterprise IdPs and OAuth 2.0 resource servers without requiring full OIDC adoption by the IdP.

### Data Model & API Specifications

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.1.2.html
- The industry standard for documenting REST APIs using JSON or YAML. OpenAPI 3.1 achieves full JSON Schema 2020-12 compatibility. Auth platform REST APIs should be documented with OpenAPI 3.1 to enable automatic SDK generation, interactive documentation, and conformance testing. ZITADEL, Ory, and Hanko all publish OpenAPI specifications for their APIs.

**Protocol Buffers / gRPC**
- URL: https://protobuf.dev/ / https://grpc.io/
- Binary wire protocol and IDL used by ZITADEL and emerging auth platforms for typed, high-performance APIs. gRPC-gateway can generate REST + OpenAPI from proto definitions, enabling dual REST/gRPC surface. Relevant for platforms targeting cloud-native and microservice deployments.

### Security & Authentication Standards

**NIST SP 800-63-4 — Digital Identity Guidelines (July 2025)**
- URL: https://pages.nist.gov/800-63-4/
- Authentication volume: https://csrc.nist.gov/pubs/sp/800/63/b/4/final
- The definitive US federal framework for digital identity, defining Identity Assurance Level (IAL), Authentication Assurance Level (AAL), and Federation Assurance Level (FAL). Revision 4 (finalised July 2025) integrates guidance on syncable authenticators (passkeys), risk-based approaches, and privacy-enhancing technologies. AAL2 mandates MFA; AAL3 requires phishing-resistant hardware authenticators. Auth platforms targeting US federal or regulated sectors must align with this framework.

**OWASP Authentication Cheat Sheet**
- URL: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Community-maintained best practice guide covering password storage (bcrypt, Argon2), session management, MFA bypass mitigations, account lockout, and secure recovery flows. A practical complement to NIST SP 800-63 for implementation teams.

**OWASP Top 10 — A07:2021 Identification and Authentication Failures**
- URL: https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/
- Enumerates the most critical authentication vulnerabilities: credential stuffing, weak password policies, session fixation, and insecure "remember me" mechanisms. Auth platform design should address all enumerated failure modes.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://gdpr.eu/
- Article 32 requires appropriate technical measures to ensure confidentiality, integrity, and availability of personal data processing, including MFA and encryption. Auth platforms processing EU user data must implement data minimisation, user consent management, right-to-erasure capabilities, and breach notification mechanisms. GDPR compliance in 2026 increasingly focuses on AI processing, consent manipulation prevention, and proactive privacy engineering.

---

## Similar Products — Developer Documentation & APIs

### Auth0 (Okta)

- **Description:** Market-leading commercial CIAM platform with broad protocol support, 7,000+ integrations, adaptive MFA, and an extensive SDK ecosystem.
- **API Documentation:** https://auth0.com/docs/api
- **SDKs/Libraries:** https://auth0.com/docs/libraries — SDKs for JavaScript/Node.js, .NET, Go, Python, Java, Swift (iOS), Android, Ruby, and PHP
- **Developer Guide:** https://developer.auth0.com/resources
- **Standards:** REST/JSON, OIDC, SAML 2.0, OAuth 2.0, SCIM, PKCE, JWT (RS256/ES256)
- **Authentication:** OAuth 2.0 Bearer Token (Management API key), PKCE for SPA/mobile, mTLS for M2M

### Clerk

- **Description:** Developer-first CIAM platform with pre-built React/Next.js components, hybrid JWT+cookie sessions, and seamless edge function compatibility.
- **API Documentation:** https://clerk.com/docs/reference/backend-api
- **SDKs/Libraries:** https://clerk.com/docs/reference/overview — JavaScript, TypeScript, React, Next.js, Remix, Node.js, Python, Go, Java, Ruby, PHP, C#
- **Developer Guide:** https://clerk.com/docs
- **Standards:** REST/JSON, OIDC, OAuth 2.0, JWT, PKCE, WebAuthn/Passkeys
- **Authentication:** Session token (short-lived JWT), API key (backend management)

### Descope

- **Description:** No-code CIAM platform featuring a drag-and-drop flow builder for authentication journeys, adaptive MFA, and self-service enterprise SSO configuration.
- **API Documentation:** https://docs.descope.com/api
- **SDKs/Libraries:** https://docs.descope.com/backend-sdk — Node.js, Python, Go, Java, Ruby, PHP, .NET; Client SDKs: React, Vue, Angular, Next.js, Vanilla JS
- **Developer Guide:** https://docs.descope.com/
- **Standards:** REST/JSON, OIDC, SAML 2.0, OAuth 2.0, JWT, WebAuthn/Passkeys
- **Authentication:** API key (project ID + management key), session JWT for end users

### ZITADEL

- **Description:** API-first, open-source identity platform (AGPL 3.0 self-hosted / SaaS) with event-sourced audit trail, gRPC + REST APIs, and strong multi-tenancy.
- **API Documentation:** https://zitadel.com/docs/apis/introduction
- **SDKs/Libraries:** https://zitadel.com/docs/sdk-examples/introduction — Go, Node.js, Python, .NET, Angular, React; community SDKs for Rust, Dart, Java
- **Developer Guide:** https://zitadel.com/docs
- **Standards:** gRPC + REST (OpenAPI), OIDC, SAML 2.0, OAuth 2.0, SCIM, JWT (RS256/ES256), WebAuthn
- **Authentication:** OAuth 2.0 Bearer Token (service account JWT), PKCE for interactive clients, mTLS for M2M

### Ory (Kratos + Hydra)

- **Description:** Open-source headless identity microservices (Apache 2.0). Kratos handles identity flows (registration, login, MFA); Hydra is an OpenID-certified OAuth 2.0 and OIDC provider.
- **API Documentation:** https://www.ory.com/docs/hydra/reference/api (Hydra); https://www.ory.com/docs/kratos/reference/api (Kratos)
- **SDKs/Libraries:** https://github.com/ory/sdk — auto-generated from OpenAPI specs; Go, TypeScript, Python, PHP, Rust, Java, .NET
- **Developer Guide:** https://www.ory.com/docs
- **Standards:** REST/JSON (OpenAPI 3.x), OIDC (certified), OAuth 2.0, JWT, WebAuthn/Passkeys, PKCE
- **Authentication:** OAuth 2.0 client credentials (Hydra), session cookie (Kratos flows), API key (Ory Network management)

### Keycloak

- **Description:** Widely-deployed open-source (Apache 2.0) identity provider supporting SAML, OIDC, LDAP/AD federation, and realm-based multi-tenancy; backed by Red Hat.
- **API Documentation:** https://www.keycloak.org/docs-api/latest/rest-api/index.html
- **SDKs/Libraries:** https://www.keycloak.org/documentation — official Java Admin Client; community SDKs for Node.js, Python, Go, .NET, PHP
- **Developer Guide:** https://www.keycloak.org/docs/latest/server_development/index.html
- **Standards:** REST/JSON, OIDC, SAML 2.0, OAuth 2.0, SCIM (via plugin), JWT, WebAuthn, LDAP v3
- **Authentication:** Bearer token (obtained via client credentials or user login), realm admin credentials for management API

### Hanko

- **Description:** Open-source (MIT), passkey-first auth platform with a privacy-by-design philosophy, server-side sessions, SAML SSO, and a dedicated Passkey API for embedding into existing systems.
- **API Documentation:** https://docs.hanko.io/api-reference/introduction
- **SDKs/Libraries:** https://teamhanko.github.io/hanko/jsdoc/hanko-frontend-sdk/ — Frontend SDK (TypeScript/JavaScript); REST API is framework-agnostic
- **Developer Guide:** https://docs.hanko.io/
- **Standards:** REST/JSON (OpenAPI), WebAuthn/FIDO2 (certified), SAML 2.0, OAuth 2.0 (social), JWT, TOTP
- **Authentication:** Session cookie (server-side sessions), JWT for API-based integrations

### WorkOS

- **Description:** Commercial platform targeting B2B SaaS teams, providing enterprise SSO (SAML/OIDC), Directory Sync (SCIM), and Admin Portal with minimal integration effort.
- **API Documentation:** https://workos.com/docs/reference
- **SDKs/Libraries:** https://workos.com/docs — Node.js, Python, Ruby, Go, .NET, PHP, Java; open-source SDK repositories on GitHub
- **Developer Guide:** https://workos.com/docs/sso
- **Standards:** REST/JSON, SAML 2.0, OIDC, OAuth 2.0, SCIM 2.0, JWT
- **Authentication:** API key (secret key passed in Authorization header), PKCE for AuthKit interactive flows

### Supabase Auth (GoTrue)

- **Description:** Open-source (MIT) JWT-based auth service tightly integrated with PostgreSQL, providing social login, email/SMS OTP, SAML SSO, and row-level security (RLS) bindings.
- **API Documentation:** https://supabase.com/docs/reference/self-hosting-auth/introduction
- **SDKs/Libraries:** https://supabase.com/docs/reference/javascript/auth-api — JavaScript/TypeScript (primary); community Go, Python, C#, Dart, Swift SDKs
- **Developer Guide:** https://supabase.com/docs/guides/auth
- **Standards:** REST/JSON, OIDC, OAuth 2.0, SAML 2.0, JWT (RS256, verified via Postgres JWK), OTP
- **Authentication:** JWT (short-lived access token + refresh token), service role key for backend management

---

## Notes

**Emerging standards to monitor:**

- **OAuth 2.1 finalisation**: The draft consolidating RFC 9700 and related security improvements is in active development (draft-ietf-oauth-v2-1-15 as of early 2026). Implementations should track this for alignment as it approaches RFC status.
- **W3C WebAuthn Level 3**: Work is ongoing on the next WebAuthn revision, which includes additional attestation formats and enterprise attestation improvements. Follow https://www.w3.org/TR/webauthn-3/ for updates.
- **OpenID Connect for Identity Assurance (OIDC4IDA)**: A specification for conveying verified identity claims (e.g., KYC-verified attributes) within OIDC tokens. Relevant for regulated use cases (financial services, healthcare). See https://openid.net/specs/openid-connect-4-identity-assurance-1_0.html.
- **Decentralised Identity (DID / Verifiable Credentials)**: W3C Decentralised Identifiers (DID) 1.0 and Verifiable Credentials Data Model 2.0 are published Recommendations. These remain niche but are gaining traction in regulated and privacy-sensitive sectors. Platforms may consider DID resolution as an optional federation mechanism.
- **MCP Server Authentication**: The Model Context Protocol (used by AI agents) increasingly requires OAuth 2.0 integration for tool authorisation. Auth platforms may need to expose OIDC-compatible token issuance endpoints that MCP-based agents can use for delegated access. See https://modelcontextprotocol.io/.
