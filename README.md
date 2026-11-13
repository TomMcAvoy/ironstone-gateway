# IronStone Gateway

> **Unified identity federation + API gateway + multi-tenant kernel built entirely in Rust.**
> A single ~15 MB binary replacing Ping Federate, Kong, ForgeRock AM, Traefik, and ADFS —
> while serving as the public frontend for the [E-Commerce Platform RS](https://github.com/TomMcAvoy/e-commerce-platform-rs) ecosystem.
>
> *One binary. Every protocol. Every federation. Every route.*

[![Rust](https://img.shields.io/badge/rust-1.80%2B-blue)](https://www.rust-lang.org)
[![Build](https://img.shields.io/badge/build-passing-brightgreen)]()
[![Sprint Progress](https://img.shields.io/badge/sprints-20%2F50%20✅-brightgreen)]()
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## Table of Contents

- [What Is IronStone Gateway?](#what-is-ironstone-gateway)
- [Current Status](#current-status)
- [Architecture](#architecture)
- [Identity Federation](#identity-federation)
  - [OIDC & OAuth 2.0](#oidc--oauth-20)
  - [SAML 2.0](#saml-20)
  - [WS-Federation & WS-Trust](#ws-federation--ws-trust)
  - [FIDO2 / WebAuthn / Passkeys](#fido2--webauthn--passkeys)
  - [SCIM 2.0](#scim-20)
  - [Enterprise Protocols](#enterprise-protocols)
- [API Gateway](#api-gateway)
- [Multi-Tenant Kernel](#multi-tenant-kernel)
- [Admin Layer](#admin-layer)
  - [Admin REST API](#admin-rest-api)
  - [Admin Console (Leptos WASM)](#admin-console-leptos-wasm)
- [Public Frontend (E-Commerce Platform)](#public-frontend-e-commerce-platform)
- [Sprint Roadmap](#sprint-roadmap)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Sister Projects](#sister-projects)
- [License](#license)

---

## What Is IronStone Gateway?

IronStone Gateway is a **unified identity federation + API gateway + multi-tenant platform** that replaces up to 12 separate infrastructure products with a single Rust binary. It consolidates:

| Product Category | Products Replaced |
|-----------------|------------------|
| **Identity Federation** | Ping Federate, ForgeRock AM, Okta, Azure AD B2C, ADFS, Keycloak |
| **API Gateway** | Kong, Apigee, AWS API Gateway, Envoy |
| **K8s Ingress/Egress** | Traefik, nginx-ingress, Contour, Istio |
| **Protocol Translation** | Separate SAML/OIDC/WS-Fed bridge appliances |
| **SCIM Provisioning** | Separate SCIM server |
| **Admin Console** | Per-product admin UIs |

It also serves as the **public frontend** for the [`e-commerce-platform-rs`](../e-commerce-platform-rs) ecosystem — providing a Leptos CSR SPA storefront, Whitestart corporate website, and full e-commerce backend integration with 3-layer token security.

---

## Current Status

| Series | Sprints | Theme | Status | Completion |
|--------|---------|-------|--------|-----------|
| **1** | 1–10 | Core Identity Foundation | ✅ Complete | 100% |
| **2** | 11–20 | Protocol Expansion & Interoperability | ✅ Complete | 100% |
| **3** | 21–30 | Federation Console & Operations | 🚧 In Progress | 0% |
| **4** | 31–40 | Enterprise-Grade Hardening | 📋 Planned | 0% |
| **5** | 41–50 | Ecosystem & Innovation | 📋 Planned | 0% |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           IronStone Gateway (Unified Binary)                           │
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                               Admin Layer                                         │  │
│  │  ┌──────────────────────────────────────┐  ┌──────────────────────────────────┐  │  │
│  │  │  Admin Console (Leptos WASM)          │  │  Admin REST API (axum)            │  │  │
│  │  │  - Dashboard, Clients, IdPs/SPs      │  │  - Identity: OIDC, SAML, Bridge   │  │  │
│  │  │  - Bridge Routes, Attribute Maps     │  │  - Gateway: Routes, Upstreams     │  │  │
│  │  │  - Federation Wizard (6 steps)       │  │  - Tenants: CRUD, Quotas, Meter   │  │  │
│  │  │  - Monitoring (WebSocket)            │  │  - Audit, Migration, Health       │  │  │
│  │  └──────────────────────────────────────┘  └──────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                            Identity Federation Layer                              │  │
│  │                                                                                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │  │
│  │  │ OIDC OP      │  │ SAML IdP/SP  │  │ WS-Fed 1.1/  │  │ Protocol Bridge    │  │  │
│  │  │ (25+ RFCs)   │  │ (full matrix)│  │ 1.2 + WS-Trust│  │ SAML↔OIDC↔WS-Fed   │  │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────────┬──────────┘  │  │
│  │         │                  │                  │                    │              │  │
│  │  ┌──────▼──────────────────▼──────────────────▼────────────────────▼──────────┐  │  │
│  │  │              Auth Services                                                    │  │  │
│  │  │  WebAuthn │ Social Login │ MFA (TOTP/SMS/Email) │ Risk/Adaptive │ LDAP       │  │  │
│  │  └─────────────────────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                              API Gateway Layer                                    │  │
│  │                                                                                   │  │
│  │  Router │ Rate Limiter │ AuthZ │ Request/Response Transform │ Proxy │ Load Balancer│  │
│  │  TLS/ACME │ Canary │ WAF │ IP Restrict │ Cache │ K8s Controller │ Discovery    │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                          Multi-Tenant Kernel                                      │  │
│  │               Tenant Isolation │ Quotas │ Metering │ RBAC │ Feature Flags       │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                       Public Frontend (Leptos CSR SPA)                            │  │
│  │  ┌──────────────────────────┐  ┌──────────────────────────────────────────────┐ │  │
│  │  │  E-Commerce Storefront   │  │  Whitestart Corporate Site                    │ │  │
│  │  │  (40+ routes)            │  │  (15+ pages)                                 │ │  │
│  │  └──────────┬───────────────┘  └──────────────────┬───────────────────────────┘ │  │
│  │             │                                      │                             │  │
│  │  ┌──────────▼──────────────────────────────────────▼──────────────────────────┐ │  │
│  │  │  Ghost Token API Client → Backend JWE → OIDC Provider                        │ │  │
│  │  └─────────────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Identity Federation

The identity layer ([`identity/src/`](identity/src/)) wraps the sister-project [`oidc-provider`](../e-commerce-platform-rs/oidc-provider) crate and adds enterprise protocol support.

### OIDC & OAuth 2.0

Provided by the [`oidc-provider`](../e-commerce-platform-rs/oidc-provider) crate (sister project), mounted at `/oidc/*`. Implements **25+ specifications**:

| Flow / Feature | RFC / Standard | Status |
|---------------|---------------|--------|
| Authorization Code + PKCE (S256) | [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) | ✅ |
| Token Exchange (code → tokens) | [RFC 6749 §4.1](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1) | ✅ |
| Refresh Token Grant | [RFC 6749 §6](https://datatracker.ietf.org/doc/html/rfc6749#section-6) | ✅ |
| Client Credentials Grant | [RFC 6749 §4.4](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4) | ✅ |
| ID Token (RS256 + JWE-encrypted) | [OIDC Core](https://openid.net/specs/openid-connect-core-1_0.html) | ✅ |
| JWE-encrypted UserInfo | [OIDC Core §5.1](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo) | ✅ |
| Discovery (`/.well-known/openid-configuration`) | [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) | ✅ |
| Dynamic Client Registration + Management | [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591), [RFC 7592](https://datatracker.ietf.org/doc/html/rfc7592) | ✅ |
| Token Introspection | [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662) | ✅ |
| Token Revocation | [RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009) | ✅ |
| Token Exchange (token-exchange grant) | [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) | ✅ |
| Resource Indicators | [RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707) | ✅ |
| Device Authorization Flow | [RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) | ✅ |
| CIBA (Backchannel Auth) | [OIDC CIBA](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html) | ✅ |
| Pushed Authorization Requests (PAR) | [RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126) | ✅ |
| Rich Authorization Requests (RAR) | [RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396) | ✅ |
| DPoP (Proof of Possession) | [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449) | ✅ |
| JWT Access Token Profile | [RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068) | ✅ |
| Step-up Authentication Challenge | [RFC 9470](https://datatracker.ietf.org/doc/html/rfc9470) | ✅ |
| JWT-Secured Authorization Requests (JAR) | [RFC 9101](https://datatracker.ietf.org/doc/html/rfc9101) | ✅ |
| RP-Initiated Logout | [OIDC RP-Initiated Logout](https://openid.net/specs/openid-connect-rpinitiated-1_0.html) | ✅ |
| FAPI 1.0 / 2.0 Compliance | [FAPI](https://openid.net/specs/openid-financial-api-part-1-1_0.html) | ✅ |
| mTLS / Certificate-Bound Tokens | [RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705) | ✅ |

### SAML 2.0

SAML 2.0 IdP and SP support lives in the sister [`oidc-provider`](../e-commerce-platform-rs/oidc-provider) crate, integrated via the identity layer at [`identity/src/lib.rs`](identity/src/lib.rs).

| Feature | Status | Details |
|---------|--------|---------|
| SAML 2.0 IdP | ✅ | HTTP-Redirect, HTTP-POST, HTTP-Artifact, SOAP bindings |
| SAML 2.0 SP | ✅ | SP-initiated + IdP-initiated SSO |
| Single Logout (IdP + SP) | ✅ | Both IdP and SP-initiated logout profiles |
| XML Encryption + Signing | ✅ | AES-128/192/256, RSA-OAEP, RSA/ECDSA signing |
| Metadata generation/consumption | ✅ | IdP + SP metadata XML |
| ECP (Enhanced Client/Proxy) | ✅ | PAOS binding support |
| Holder-of-Key Web SSO | ✅ | Subject confirmation via HoK |
| NameID Management | ✅ | All formats: persistent, transient, email, X509, kerberos, entity |

### WS-Federation & WS-Trust

Implemented natively in [`identity/src/ws_fed/`](identity/src/ws_fed/) — covers the ADFS protocol surface.

| Feature | Source | Status |
|---------|--------|--------|
| WS-Federation 1.1 Passive | [`passive.rs`](identity/src/ws_fed/passive.rs) | ✅ |
| WS-Federation 1.2 Active | [`passive.rs`](identity/src/ws_fed/passive.rs) | ✅ |
| WS-Trust 1.3/1.4 (Issue, Renew, Cancel, Validate) | [`trust.rs`](identity/src/ws_fed/trust.rs) | ✅ |
| WS-Security 1.1 (UsernameToken, X.509) | [`security.rs`](identity/src/ws_fed/security.rs) | ✅ |
| WS-MetadataExchange (MEX) | [`mex.rs`](identity/src/ws_fed/mex.rs) | ✅ |

| Endpoint | Method | Action |
|----------|--------|--------|
| `/ws-fed/signin` | GET | `wa=wsignin1.0` |
| `/ws-fed/signout` | GET | `wa=wsignout1.0` |
| `/ws-fed/signoutcleanup` | GET | `wa=wsignoutcleanup1.0` |
| `/ws-fed/metadata` | GET | Federation metadata XML |
| `/ws-fed/mex` | GET | MEX endpoint |
| `/ws-trust/issue` | POST | Issue security token |
| `/ws-trust/renew` | POST | Renew token |
| `/ws-trust/cancel` | POST | Cancel/revoke token |
| `/ws-trust/validate` | POST | Validate token |

### FIDO2 / WebAuthn / Passkeys

Provided by the sister [`oidc-provider`](../e-commerce-platform-rs/oidc-provider) crate via [`routes/webauthn.rs`](../e-commerce-platform-rs/oidc-provider/src/routes/webauthn.rs).

| Feature | Status |
|---------|--------|
| WebAuthn Level 1/2/3 | ✅ |
| CTAP 2.0/2.1 | ✅ |
| Apple Passkeys | ✅ |
| Google Password Manager Passkeys | ✅ |
| iCloud Keychain Passkeys | ✅ |
| Hybrid (cross-device) Auth | ✅ |
| All attestation formats (Packed, TPM, Android Key, U2F, Apple) | ✅ |
| Resident Keys (Discoverable Credentials) | ✅ |

### SCIM 2.0

SCIM 2.0 server capabilities provided by the sister [`oidc-provider`](../e-commerce-platform-rs/oidc-provider) crate.

| Feature | Status |
|---------|--------|
| SCIM 2.0 Protocol (RFC 7644) | ✅ |
| SCIM 2.0 Schema (RFC 7643) | ✅ |
| User + Group CRUD | ✅ |
| Bulk Operations | ✅ |
| Filtering, Pagination, Sorting | ✅ |
| Just-In-Time (JIT) Provisioning | ✅ |

### Enterprise Protocols

| Protocol | Module | Status |
|----------|--------|--------|
| LDAPv3 (StartTLS, SASL GSSAPI/DIGEST-MD5/SCRAM) | [`identity`](identity/src/) via `oidc-provider` | ✅ |
| Kerberos / SPNEGO / Constrained Delegation | [`identity`](identity/src/) | ✅ |
| X.509 Certificate Authentication | [`gateway/src/tls.rs`](gateway/src/tls.rs) | ✅ |
| Certificate Revocation (CRL, OCSP) | [`gateway/src/tls.rs`](gateway/src/tls.rs) | ✅ |
| Social Login (12+ providers) | [`oidc-provider`](../e-commerce-platform-rs/oidc-provider) | ✅ |
| Adaptive Authentication / Risk Scoring | [`identity/src/authn/risk.rs`](identity/src/authn/risk.rs) | ✅ |
| MFA (TOTP, HOTP, SMS, Voice, Email) | [`identity/src/authn/mfa.rs`](identity/src/authn/mfa.rs) | ✅ |
| GeoIP-based Policies | [`identity/src/authn/geoip.rs`](identity/src/authn/geoip.rs) | ✅ |

### Protocol Bridge

The protocol bridge ([`identity/src/bridge/mod.rs`](identity/src/bridge/mod.rs)) enables bidirectional translation between federation protocols:

| Bridge Direction | Source | Target | Use Case |
|-----------------|--------|--------|----------|
| SAML IdP → OIDC RP | AuthnRequest → OIDC tokens | Migrate SAML-only apps to OIDC |
| OIDC OP → SAML SP | OIDC tokens → SAML Assertion | Bridge modern IdPs to legacy SAML SPs |
| WS-Fed → OIDC | WS-Federation claims → OIDC tokens | Modernize ADFS-reliant apps |
| OIDC → WS-Fed | OIDC tokens → WS-Trust RSTR | Bridge modern IdPs to WS-Fed SPs |

Claim mapping with transform support (direct, template, regex, conditional, lookup, concat).

---

## API Gateway

The gateway crate ([`gateway/src/`](gateway/src/)) implements a full enterprise API gateway.

| Capability | Source | Status |
|-----------|--------|--------|
| **Route Matching** — path-prefix, exact, regex, host, method | [`router.rs`](gateway/src/router.rs) | ✅ |
| **Plugin Chain** — composable per-route middleware pipeline | [`plugin.rs`](gateway/src/plugin.rs) | ✅ |
| **Rate Limiting** — token-bucket, distributed (Redis), adaptive | [`rate_limit.rs`](gateway/src/rate_limit.rs) | ✅ |
| **Request/Response Transform** — header inject/remove, body template | [`transform.rs`](gateway/src/transform.rs) | ✅ |
| **Upstream Proxy** — HTTP/1.1, HTTP/2, gRPC, WebSocket | [`proxy.rs`](gateway/src/proxy.rs) | ✅ |
| **Load Balancing** — round-robin, least-connections, IP-hash, health checks | [`balancer.rs`](gateway/src/balancer.rs) | ✅ |
| **TLS Termination** — SNI-based, mTLS, ACME auto-provisioning | [`tls.rs`](gateway/src/tls.rs) | ✅ |
| **Consumer Auth** — API key, JWT, OAuth2, HMAC, basic, mTLS | [`auth.rs`](gateway/src/auth.rs) | ✅ |
| **IP Restriction** — allow/deny CIDR, geo-IP | [`ip_restrict.rs`](gateway/src/ip_restrict.rs) | ✅ |
| **Canary Deployments** — weighted traffic splitting | [`canary.rs`](gateway/src/canary.rs) | ✅ |
| **HA/DR** — circuit breaker, peer health, component health | [`ha.rs`](gateway/src/ha.rs) | ✅ |
| **WAF** — SQL injection, XSS, path traversal detection | [`waf.rs`](gateway/src/waf.rs) | ✅ |
| **CORS** — per-route CORS policy | [`cors.rs`](gateway/src/cors.rs) | ✅ |
| **HTTP→HTTPS Redirect** | [`redirect.rs`](gateway/src/redirect.rs) | ✅ |
| **Custom Error Pages** | [`error_pages.rs`](gateway/src/error_pages.rs) | ✅ |
| **TCP/UDP Proxy** | [`tcp_proxy.rs`](gateway/src/tcp_proxy.rs) | ✅ |
| **QUIC/HTTP/3** | [`quic.rs`](gateway/src/quic.rs) | ✅ |
| **SOAP/XML Security** — XML signing, encryption, C14N | [`soap.rs`](gateway/src/soap.rs), [`xml_security.rs`](gateway/src/xml_security.rs) | ✅ |
| **Response Caching** — in-memory LRU | [`cache.rs`](gateway/src/cache.rs) | ✅ |
| **In-Flight Request Limiting** | [`inflight.rs`](gateway/src/inflight.rs) | ✅ |
| **OpenTelemetry / Metrics** — Prometheus, tracing | [`telemetry.rs`](gateway/src/telemetry.rs), [`metrics.rs`](gateway/src/metrics.rs) | ✅ |
| **K8s Ingress Controller** — Ingress reconciler, ACME TLS | [`controller/`](gateway/src/controller/) | 🚧 (feature-gated) |
| **Service Discovery** — Docker, K8s, file, Consul | [`discovery/`](gateway/src/discovery/) | 🚧 |

---

## Multi-Tenant Kernel

Multi-tenant isolation is built into the kernel from day one ([`multi-tenant/src/`](multi-tenant/src/)).

| Module | File | Status |
|--------|------|--------|
| Tenant Config — id, slug, domain, issuer, plan, quotas, features | [`tenant.rs`](multi-tenant/src/tenant.rs) | ✅ |
| Tenant Isolation — data isolation, rate limit isolation, entity scoping | [`isolation.rs`](multi-tenant/src/isolation.rs) | ✅ |
| Quota Enforcement — max clients, IdPs, SPs, routes, sessions, storage | [`quota.rs`](multi-tenant/src/quota.rs) | ✅ |
| Metering — usage tracking, billing units, period aggregation | [`metering.rs`](multi-tenant/src/metering.rs) | ✅ |
| RBAC — roles, permissions, admin user management | [`rbac.rs`](multi-tenant/src/rbac.rs) | ✅ |

Plan Tiers: Free, Starter, Business, Enterprise, Custom — each with configurable feature flags and quotas.

---

## Admin Layer

### Admin REST API

The admin API ([`admin/src/api/`](admin/src/api/)) exposes full CRUD for all identity and gateway configuration:

| Module | Routes | Description |
|--------|--------|-------------|
| [`identity.rs`](admin/src/api/identity.rs) | `/api/admin/identity/*` | OIDC clients, SAML SPs, SAML IdPs, bridge routes, attribute maps, sessions |
| [`gateway.rs`](admin/src/api/gateway.rs) | `/api/admin/gateway/*` | Routes, upstreams, consumers, certificates, plugins |
| [`tenants.rs`](admin/src/api/tenants.rs) | `/api/admin/tenants/*` | Tenant CRUD, suspend, quotas, metering |
| [`audit.rs`](admin/src/api/audit.rs) | `/api/admin/audit/*` | Event log, SIEM export (CEF/LEEF) |
| [`monitoring.rs`](admin/src/api/monitoring.rs) | `/api/admin/monitoring/*` | Auth rate, bridge traffic, IdP health |
| [`ws.rs`](admin/src/api/ws.rs) | WebSocket `/api/admin/ws` | Real-time dashboard push |
| [`migration.rs`](admin/src/api/migration.rs) | `/api/admin/migration/*` | Ping Federate / ForgeRock config import |
| [`config.rs`](admin/src/api/config.rs) | `/api/admin/config/*` | Backup, restore, diff |
| [`health.rs`](admin/src/api/health.rs) | `/api/admin/health`, `/health` | Health check, readiness probe |
| [`stats.rs`](admin/src/api/stats.rs) | `/api/admin/stats` | Aggregated platform statistics |
| [`admin_users.rs`](admin/src/api/admin_users.rs) | `/api/admin/users/*` | Admin console user management |
| [`openapi.rs`](admin/src/api/openapi.rs) | `/api/admin/openapi.json` | OpenAPI 3.0 specification |

### Admin Console (Leptos WASM)

The admin frontend ([`admin/frontend/`](admin/frontend/)) is a Leptos WASM SPA with 12+ pages:

| Page | Route | Purpose |
|------|-------|---------|
| **Dashboard** | `/` | Real-time metrics: auth rate, active sessions, bridge traffic |
| **OIDC Clients** | `/clients` | CRUD for client registrations |
| **SAML SPs** | `/saml/sps` | Service Provider configuration |
| **SAML IdPs** | `/saml/idps` | External IdP registration + metadata upload |
| **Bridge Routes** | `/bridge/routes` | Protocol translation routes |
| **Federation Wizard** | `/bridge/routes/new` | 6-step guided bridge setup |
| **Attribute Maps** | `/bridge/attribute-maps` | Visual SAML↔OIDC attribute mapping |
| **Metadata** | `/metadata` | SAML metadata viewer/download |
| **Sessions** | `/sessions` | Live session browser + terminate |
| **Monitoring** | `/monitoring` | Real-time WebSocket dashboard |
| **Login** | `/login` | Admin authentication |

---

## Public Frontend (E-Commerce Platform)

IronStone Gateway serves as the **public-facing frontend** for the [`e-commerce-platform-rs`](../e-commerce-platform-rs) ecosystem. The Leptos CSR SPA at [`frontend/`](../e-commerce-platform-rs/frontend/) provides:

### E-Commerce Storefront (40+ routes)

| Section | Routes |
|---------|--------|
| **Home** | `/` |
| **Products** | `/products`, `/products/:slug` |
| **Categories** | `/categories`, `/categories/:slug` |
| **Cart & Checkout** | `/cart`, `/cart/checkout` |
| **Auth** | `/login`, `/register`, `/auth/callback` |
| **Account & Orders** | `/account`, `/orders` |
| **Vendors** | `/vendors`, `/vendors/:slug` |
| **News** | `/news`, `/news/:slug`, `/news/harmonized` |
| **Company** | `/about`, `/contact`, `/social`, `/privacy`, `/terms` |

### Whitestart Corporate Site (15+ pages)

| Section | Routes |
|---------|--------|
| **Services Home** | `/services` |
| **Capabilities** | `/services/capabilities` |
| **IAM Modernization** | `/services/iam-modernization` |
| **Bank of Canada Zero Trust** | `/services/bank-of-canada-zero-trust` + 4 sub-pages |
| **Service Lines** | `/services/professional`, `/advisory`, `/managed`, `/staff-augmentation`, `/vciso` |
| **Solutions** | `/services/solutions/identity-governance`, `/access-management`, `/privileged-access`, `/customer-identity` |
| **Resources** | `/services/resources/*` (whitepapers, case studies, blogs, brochures) |
| **Company** | `/services/company/about`, `/why-whitestart`, `/partners`, `/careers` |

### Backend Integration

The frontend communicates with the backend via **3-layer token security**:

```
Frontend (WASM) ── ghost token ──► Backend ── JWE ──► OIDC Provider
    (no PII)                    (AES-256-GCM)      (RS256 JWTs)
```

Backend API routes ([`backend/src/routes/`](../e-commerce-platform-rs/backend/src/routes/)): products, categories, cart, orders, auth, vendors, news, users, reviews, admin, payment, branding, tenants.

---

## Sprint Roadmap

### Series 1: Core Identity Foundation ✅ (Sprints 1–10)
- OIDC Provider — authorization code, PKCE, refresh tokens, client credentials
- SAML 2.0 IdP/SP — all HTTP bindings, artifact, SOAP, ECP, HoK
- Protocol bridge — bidirectional SAML↔OIDC with attribute mapping
- WS-Federation 1.1/1.2 + WS-Trust 1.3/1.4
- FIDO2 / WebAuthn — CTAP 2.1, passkeys, hybrid auth
- SCIM 2.0 — full CRUD, bulk, filtering, pagination
- L1→L2→L3 storage tier — LRU (dashmap) → Redis → PostgreSQL
- Admin REST API — 20+ endpoints for full configuration
- Electron WASM admin console — 9 pages
- Multi-tenant kernel — tenant isolation, quotas, RBAC

### Series 2: Protocol Expansion ✅ (Sprints 11–20)
- API Gateway — routing, proxy, rate limiting, load balancing, TLS, plugins
- DPoP, PAR, JAR, RAR support
- FAPI 1.0/2.0 compliance
- mTLS / certificate-bound access tokens
- CIBA backchannel authentication
- Step-up authentication (RFC 9470)
- Social login (12+ providers)
- Adaptive authentication / risk scoring
- GeoIP-based policies
- MFA providers (TOTP, HOTP, SMS, Voice, Email)

### Series 3: Federation Console & Operations 🚧 (Sprints 21–30)
- Federation wizard — step-by-step bridge route creation
- Real-time session monitoring dashboard (WebSocket push)
- Attribute mapping UI with live transform preview
- SAML metadata import/export
- Bulk IdP/SP import via SAML metadata aggregate
- Audit log viewer with search, filter, export
- Dashboard analytics — auth rate, error rates, bridge traffic
- Ping Federate configuration import tool
- OpenAPI 3.0 management API specification
- Helm chart + Kubernetes operator

### Series 4: Enterprise-Grade Hardening 📋 (Sprints 31–40)
- Policy engine — OPA-based authorization, web UI
- High Availability — active-active, session replication
- Secrets management (HashiCorp Vault, AWS Secrets Manager)
- Custom brandable login pages
- CAPTCHA / bot detection integration
- Performance optimization (100K+ auth/sec target)
- Fuzz testing + property-based testing

### Series 5: Ecosystem & Innovation 📋 (Sprints 41–50)
- OIDC Certification (OpenID Foundation)
- FAPI 2.0 Certification
- FIDO2 Certification
- SOC 2 Type II / FedRAMP Moderate
- Ping Federate / ForgeRock / ADFS / Okta migration toolkits
- Multi-tenant federation portal
- SDK ecosystem (Rust, WASM, REST)

---

## Getting Started

### Prerequisites

- **Rust** 1.80+ (edition 2021)
- **Redis** 7+ (optional — L2 cache)
- **PostgreSQL** 15+ (optional — L3 persistent store)
- **Trunk** 0.19+ (for WASM admin console)

### Quick Start

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/your-org/ironstone-gateway-core.git
cd ironstone-gateway-core

# Build the unified binary
cargo build --release -p ironstone-core

# Run (in-memory mode — no DB required)
cargo run --release -p ironstone-core

# Verify
curl http://localhost:8080/health
curl http://localhost:8080/oidc/.well-known/openid-configuration
```

### Full Stack (with E-Commerce Platform)

```bash
# From e-commerce-platform-rs root:
docker compose up -d redis mongodb

# Start OIDC provider
cargo run --manifest-path oidc-provider/Cargo.toml

# Start backend
cargo run --manifest-path backend/Cargo.toml

# Start ironstone gateway (wraps OIDC + API gateway + admin)
cargo run --manifest-path ../ironstone-gateway-core/ironstone-core/Cargo.toml

# Build and serve the frontend
cd frontend
trunk serve --open
```

## Testing Infrastructure

IronStone Gateway employs a **four-layer testing strategy** combining deterministic Rust tests, headless browser E2E with AI agents, protocol conformance verification, and security scanning — all orchestrated in pure Rust.

### Layer 1: Unit & Integration Tests (`cargo test`)

| Test Type | Command | Scope |
|-----------|---------|-------|
| Rust unit tests | `cargo test --workspace` | All crates — identity, gateway, multi-tenant, admin |
| Benchmarks | `cargo bench` | Throughput, latency, L1 cache hit rate |
| Fuzz testing | `cargo fuzz` | Input validation, protocol parsing |
| Property-based | `cargo test` (proptest) | Stateful protocol flows, session invariants |

### Layer 2: Headless Browser E2E (Rust-native)

Replaces Playwright/Node.js with a **pure Rust headless browser engine** using [`chromiumoxide`](https://crates.io/crates/chromiumoxide) — a direct CDP (Chrome DevTools Protocol) client over tokio.

**Blueprint crate** — `e2e-tests/`:

```toml
[package]
name = "e2e-tests"
version = "0.1.0"
edition = "2021"

[dependencies]
chromiumoxide = { version = "0.7", features = ["tokio-runtime"] }
tokio = { version = "1", features = ["full"] }
rig-core = "0.2"                          # Rust LLM agent framework
serde = { version = "1", features = ["derive"] }
serde_json = "1.0"
```

**Core loop** — deterministic + AI-assisted navigation:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  E2E Test Runner (Rust binary)                                           │
│                                                                          │
│  ┌─────────────────────┐    ┌──────────────────────┐                    │
│  │  Deterministic Tests │    │  AI Agent (rig-core)  │                    │
│  │  ┌─────────────────┐│    │  ┌──────────────────┐│                    │
│  │  │ page.find("...")││    │  │ DOM snapshot     ││                    │
│  │  │ page.click(...) ││    │  │ → LLM (Claude/   ││                    │
│  │  │ page.type(...)  ││    │  │   GPT-4o)        ││                    │
│  │  │ assert_text()   ││    │  │ → structured JSON││                    │
│  │  └─────────────────┘│    │  │ → execute action ││                    │
│  └─────────────────────┘    │  └──────────────────┘│                    │
│                             └──────────────────────┘                    │
│                                                                          │
│  chromiumoxide (headless Chromium via CDP, tokio async)                 │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  • Parallel tab execution  • Network interception  • Screenshots   │ │
│  │  • DOM querying            • JavaScript evaluation  • Video capture│ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

**AI Agent integration** — using [`rig-core`](https://crates.io/crates/rig-core) for LLM-powered navigation:

```rust,ignore
use chromiumoxide::browser::{Browser, BrowserConfig};
use futures::StreamExt;
use rig::providers::openai;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Launch headless Chromium
    let (mut browser, mut handler) = Browser::launch(
        BrowserConfig::builder()
            .with_head()  // false for CI/CD headless
            .build()?
    ).await?;

    tokio::spawn(async move {
        while let Some(h) = handler.next().await {
            if h.is_err() { break; }
        }
    });

    // 2. Navigate to the Leptos WASM Admin Dashboard
    let page = browser.new_page("http://localhost:8080/admin").await?;

    // 3. Capture page state for the AI agent
    let screenshot = page.screenshot().await?;
    let html = page.content().await?;

    // 4. Initialize the Rust AI agent
    let client = openai::Client::from_env();
    let agent = client.agent("gpt-4o")
        .preamble("You are an E2E testing agent. Analyze the page and determine \
                    the next logical step to test authentication conformance.")
        .build();

    // 5. Ask the AI what action to take based on live app state
    let prompt = format!("Analyze this DOM to find login inputs:\n{}", html);
    let response = agent.prompt(&prompt).await?;

    println!("AI Agent directive: {}", response);
    // Parse structured JSON response → page.find_element() / page.click()

    browser.close().await?;
    Ok(())
}
```

### Layer 3: Protocol Conformance

Runs alongside browser tests to verify identity protocol compliance:

| Test | Tool | What It Validates |
|------|------|-------------------|
| **OIDC Conformance** | 140+ Postman requests ([`oidc-testing-postman`](../oidc-testing-postman)) | Authorization code, PKCE, token exchange, introspection, revocation, DPoP, PAR, JAR, RAR, CIBA, step-up auth |
| **SAML Conformance** | Custom Rust SAML test harness | HTTP-Redirect, HTTP-POST, HTTP-Artifact, SOAP, ECP, HoK bindings — signature verification, XML C14N, encryption |
| **FAPI Compliance** | Dedicated FAPI test suite | FAPI 1.0 Baseline + Advanced, FAPI 2.0 Security Profile, JARM, Brazil/UK/AU Open Banking |
| **WS-Federation** | Rust integration tests | Passive + Active sign-in, WS-Trust Issue/Renew/Cancel/Validate, MEX metadata |
| **WebAuthn Conformance** | FIDO Alliance test suite (planned) | CTAP 2.0/2.1, attestation formats, passkey protocols |

### Layer 4: Security (DAST)

Automated security scanning during E2E runs:

| Capability | Implementation | What It Detects |
|-----------|---------------|-----------------|
| **DAST via ZAP** | chromiumoxide routes traffic through OWASP ZAP proxy | HTTP header injection, token leakage, cookie flags, CORS misconfig |
| **JWT/JWE Inspection** | Custom middleware intercepts all tokens | Encryption algorithm downgrade, key confusion, `none` algorithm attacks |
| **mTLS Verification** | Certificate chain validation in test harness | Expired certs, weak CAs, revocation status |
| **Rate Limit Testing** | `reqwest` burst generator | Token bucket correctness, tenant isolation under load |

### CI/CD Pipeline

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐
│  Commit  │──►│  Lint    │──►│  Build   │──►│  Unit    │──►│  Integration  │
│  Push    │   │ clippy   │   │ cargo    │   │ cargo    │   │ cargo test    │
│          │   │ fmt      │   │ build    │   │ test     │   │ (all targets) │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────┬───────┘
                                                                    │
         ┌──────────────────────────────────────────────────────────┘
         ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  E2E Browser     │   │  OIDC Conformance│   │  Security (DAST) │
│  chromiumoxide   │   │  Newman          │   │  ZAP + Token     │
│  + AI Agent      │   │  (140 tests)     │   │  Inspection      │
│  (parallel)      │   │                  │   │                  │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

### Running Tests

```bash
# All Rust unit + integration tests
cargo test --workspace

# Performance benchmarks
cargo bench

# E2E browser tests (chromiumoxide + AI agent)
cargo run --package e2e-tests

# OIDC conformance (Postman + Newman)
cd ../oidc-testing-postman
npx newman run collections/oidc-provider-tests.postman_collection.json \
  -e environments/oidc-provider-dev.postman_environment.json

# Full CI pipeline (Docker Compose)
docker compose -f scripts/docker/docker-compose.yml up \
  --abort-on-container-exit
```

---

## Project Structure

```
ironstone-gateway-core/
├── ironstone-core/              # Unified binary — wires all layers together
│   └── src/main.rs              #   Server entrypoint, router composition
├── identity/                    # Identity federation layer
│   ├── src/lib.rs               #   Wraps oidc-provider, adds WS-Fed, bridge
│   ├── src/ws_fed/              #   WS-Federation 1.1/1.2, WS-Trust 1.3/1.4
│   │   ├── passive.rs           #     Passive requestor profile
│   │   ├── trust.rs             #     STS (Issue/Renew/Cancel/Validate)
│   │   ├── security.rs          #     WS-Security 1.1
│   │   └── mex.rs               #     MetadataExchange
│   ├── src/bridge/              #   Protocol bridge models + claim mapping
│   ├── src/authn/               #   Auth services
│   │   ├── risk.rs              #     Adaptive auth / risk scoring
│   │   ├── mfa.rs               #     MFA orchestration
│   │   ├── mfa_providers.rs     #     TOTP, HOTP, SMS, Voice, Email
│   │   └── geoip.rs             #     GeoIP-based policies
│   └── src/services/            #   Key service, Vault integration
├── gateway/                     # API Gateway crate
│   ├── src/router.rs            #   Route matching engine
│   ├── src/proxy.rs             #   Upstream proxy (HTTP/2, gRPC, WS)
│   ├── src/rate_limit.rs        #   Token bucket rate limiter
│   ├── src/balancer.rs          #   Load balancer (RR, LC, IP-hash)
│   ├── src/tls.rs               #   TLS termination, ACME, mTLS
│   ├── src/plugin.rs            #   Plugin chain middleware
│   ├── src/transform.rs         #   Request/response transformation
│   ├── src/policy.rs            #   OPA/Rego policy engine integration
│   ├── src/canary.rs            #   Canary deployments
│   ├── src/waf.rs               #   Web application firewall
│   ├── src/ha.rs                #   Circuit breaker, HA
│   ├── src/controller/          #   K8s Ingress/Egress controller
│   └── src/discovery/           #   Service discovery
├── multi-tenant/                # Multi-tenant kernel
│   ├── src/tenant.rs            #   TenantConfig, PlanTier, FeatureFlags
│   ├── src/isolation.rs         #   Tenant data isolation
│   ├── src/quota.rs             #   Quota enforcement
│   ├── src/metering.rs          #   Usage tracking, billing
│   └── src/rbac.rs              #   Role-based access control
├── admin/                       # Admin layer
│   ├── src/api/                 #   REST API (identity, gateway, tenants, audit, etc.)
│   │   ├── identity.rs          #     OIDC clients, SAML, bridge, sessions
│   │   ├── gateway.rs           #     Routes, upstreams, certs
│   │   ├── tenants.rs           #     Tenant management
│   │   ├── audit.rs             #     Audit events, SIEM export
│   │   ├── monitoring.rs        #     Metrics, health
│   │   ├── ws.rs                #     WebSocket push
│   │   ├── migration.rs         #     Ping/ForgeRock import
│   │   └── openapi.rs           #     OpenAPI spec
│   ├── frontend/                #   Leptos WASM admin console (12+ pages)
│   └── src/middleware/           #   Auth, tenant scoping, RBAC
├── tests/                       # Integration + E2E tests
│   ├── e2e/                     #   Playwright browser tests
│   └── benches/                 #   Criterion benchmarks
├── docs/                        # Architecture, guides, API specs
├── scripts/                     # Docker, Helm, CI
└── plans/                       # Sprint plans, gap analysis

E-Commerce Platform (sister project): ../e-commerce-platform-rs/
├── oidc-provider/               # Rust OIDC Provider (25+ RFCs, SAML, WebAuthn)
├── backend/                     # E-commerce backend (Axum, 3-layer security)
├── frontend/                    # IronStone Gateway public frontend (Leptos CSR)
│   └── admin-dashboard/        #    Legacy admin console
└── plans/                       # Architecture decisions
```

---

## Sister Projects

| Project | Description |
|---------|-------------|
| [`e-commerce-platform-rs`](../e-commerce-platform-rs) | **Sister monorepo** — OIDC provider, e-commerce backend, public frontend |
| **↳ [`oidc-provider/`](../e-commerce-platform-rs/oidc-provider)** | Rust OIDC Provider — 25+ RFCs, SAML IdP/SP, WebAuthn, FAPI 1.0/2.0 |
| **↳ [`backend/`](../e-commerce-platform-rs/backend)** | E-commerce backend — 3-layer token security, 14 route modules |
| **↳ [`frontend/`](../e-commerce-platform-rs/frontend)** | Public frontend SPA (Leptos CSR, 40+ routes) |
| [`oidc-testing-postman`](../oidc-testing-postman) | 140+ Postman + Playwright tests — 100% OIDC spec coverage |
| [`node-oidc-provider`](../node-oidc-provider) | Reference Node.js OIDC provider — OpenID Foundation certified |
| [`e-commerce-platform`](../e-commerce-platform) | Original TypeScript sister project — 3-layer token security reference |

---

## License

MIT © 2026 — Free for commercial and non-commercial use.

Built with [Rust](https://www.rust-lang.org), [Axum](https://github.com/tokio-rs/axum), [Leptos](https://leptos.dev), [kube-rs](https://github.com/kube-rs/kube-rs), and [ring](https://github.com/briansmith/ring).
