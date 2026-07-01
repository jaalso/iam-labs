## 🔐 [IAM Labs](https://github.com/jaalso/iam-labs) <sup><sub>← click to open</sub></sup>
> Identity & Access Management · OIDC · JWT · Keycloak · OAuth 2.0

| # | Lab | Tools | Status |
|---|---|---|---|
| 01 | Keycloak - OIDC Authorization Code Flow | Keycloak · Docker · curl · JWT | ✅ Complete |

# 01 🔐 Keycloak Identity Broker — OIDC Authentication Lab

> **Category:** Identity & Access Management (IAM) · Security Engineering
> **Environment:** Kali Linux · Docker · Keycloak 26.6.3
> **Focus:** OpenID Connect (OIDC) Authorization Code flow · JWT analysis · identity brokering concepts

---

## Overview

This lab documents the hands-on deployment of **Keycloak**, an open-source Identity and Access Management (IAM) server, and a complete walk-through of the **OIDC Authorization Code flow** performed manually — from authentication request through token exchange to JWT decoding.

The goal was to understand, from the inside, how a modern identity provider (IdP) authenticates users and issues signed tokens that applications trust — the same category of system used in enterprise **WAF/WAM and Web Access Management** architectures.

---

## Objectives

- Deploy an identity provider (Keycloak) in a containerised lab environment
- Configure a realm, a user, and a confidential OIDC client
- Manually execute the full OIDC Authorization Code flow
- Perform the code-for-token exchange as an application back-end would
- Decode and analyse the resulting JWT (identity claims + RBAC roles)
- Understand the trust model: readable but non-forgeable signed tokens

---

## Lab Environment

| Component | Detail |
|-----------|--------|
| Host | Kali Linux (VirtualBox) |
| Runtime | Docker |
| IdP | Keycloak 26.6.3 (`quay.io/keycloak/keycloak`) |
| Tools | `curl`, `base64`, `python3`, Firefox |

---

## Key Concepts

| Term | Meaning |
|------|---------|
| **IdP** | Identity Provider — verifies users and issues identity assertions |
| **Realm** | Isolated tenant in Keycloak (own users, clients, signing keys) |
| **Client** | An application that delegates authentication to the IdP |
| **OIDC** | OpenID Connect — modern authentication protocol built on OAuth 2.0 (JSON-based) |
| **Authorization Code flow** | Most secure OIDC flow; browser receives a single-use code, back-end exchanges it for tokens |
| **JWT** | JSON Web Token — signed, self-contained token carrying identity + role claims |
| **RBAC** | Role-Based Access Control — authorization via roles carried in the token |

---

## Methodology

### 1. Deploy Keycloak

Keycloak was launched in development mode via Docker, exposing port 8080:

```bash
docker run -d --name keycloak \
  -p 8080:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:26.6.3 \
  start-dev
```

### 2. Configure realm, user, and client

Via the admin console (`http://localhost:8080/admin`):

- Created realm `lab-realm` (isolated tenant)
- Created user `jaime.test` with a permanent password
- Created a **confidential** OIDC client `lab-client` (client authentication enabled → issues a client secret)
- Registered a valid redirect URI — an allow-list control that prevents tokens being sent to unauthorised callbacks

> **Security note:** an over-broad redirect URI (e.g. a wildcard) is a real-world IAM misconfiguration — the identity-layer equivalent of an over-broad firewall rule. Redirect URIs should be scoped as narrowly as the use case allows.

### 3. Initiate the Authorization Code flow

The authorization endpoint was called in the browser to begin the flow:

```
http://localhost:8080/realms/lab-realm/protocol/openid-connect/auth
  ?client_id=lab-client
  &response_type=code
  &scope=openid
  &redirect_uri=http://localhost:8080/lab-callback
  &state=xyz123
```

After authenticating as `jaime.test`, the browser was redirected to the callback carrying a **single-use authorization code** (visible in the URL). The code is short-lived (~60s) and single-use — a stolen code is of little value.

### 4. Exchange the code for tokens

Acting as the application's back-end, the code was exchanged at the **token endpoint**, together with the client secret:

```bash
curl -s -X POST \
  http://localhost:8080/realms/lab-realm/protocol/openid-connect/token \
  -d "grant_type=authorization_code" \
  -d "client_id=lab-client" \
  -d "client_secret=$SECRET" \
  -d "code=$CODE" \
  -d "redirect_uri=http://localhost:8080/lab-callback" | python3 -m json.tool
```

The response returned three tokens:

- **access_token** — grants access to protected resources (`expires_in: 300` → 5 min)
- **id_token** — proves user identity (the OIDC-specific token)
- **refresh_token** — renews the access token without re-authentication

### 5. Decode and analyse the JWT

The access token's payload (middle segment) was decoded to inspect its claims:

```bash
echo "$ACCESS_TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

**Observed claims:**

| Claim | Value | Meaning |
|-------|-------|---------|
| `sub` | `c4ea4dac-…` | Permanent unique user ID |
| `preferred_username` | `jaime.test` | Identity |
| `iss` | `…/realms/lab-realm` | Issuer — proves which realm minted the token |
| `exp` / `iat` | timestamps | Expiry / issued-at |
| `realm_access.roles` | `[…]` | **RBAC roles** used for authorization |

---

## Key Findings

1. **Tokens are readable but not forgeable.** The JWT payload decodes with plain `base64` — no key required — so anyone can *read* it. However, the signature (third segment) is generated with the IdP's private key, so it cannot be *forged* without that key. This is the core trust model of token-based identity.

2. **The authorization code is single-use and short-lived.** During the lab the code expired twice before exchange — demonstrating firsthand that short-lived, single-use codes are a deliberate security control, not an inconvenience.

3. **The token carries authorization, not just identity.** The `realm_access.roles` claim shows RBAC data travelling inside the token — the application reads these roles to make access decisions.

4. **The client secret protects the exchange.** A confidential client must present its secret at the token endpoint. An attacker intercepting only the code cannot redeem it without the secret, which lives solely on the application back-end (never in the browser).

---

## Relevance to Enterprise IAM

This lab operates the **internal OIDC authentication path** used in enterprise WAF/WAM and Web Access Management architectures, where a central identity service (an SLS / identity broker) authenticates users and issues OIDC tokens to applications. The same identity provider can act as a **broker** — accepting SAML federation from external partners and translating it into OIDC — so that applications only ever implement a single protocol.

Understanding this flow from the inside — token issuance, validation, and the signing/trust model — directly supports IAM engineering, Web Access Management, and secure application-integration work.

---

## Tools & Technologies

`Keycloak` · `Docker` · `OIDC` · `OAuth 2.0` · `JWT` · `curl` · `base64` · `Kali Linux`

---

## Lessons Learned

This lab was built as preparation for a technical interview involving a WAF/WAM and
identity brokering architecture. Two gaps surfaced during the interview panel that
the lab directly addresses in retrospect:

**1. What sits next to the WAF in production?**
The WAF operates at Layer 7 (application inspection). In front of it sits a
**hardware network firewall at Layers 3/4** — packet filtering, port blocking, IP
allow/deny. Defense in depth: the L3/4 firewall is the outer perimeter, the WAF is
the inner application-layer control. The lab's containerised setup collapsed these
into one host, which obscured this architectural separation.

**2. The missing step in the Authorization Code flow**
Between the application receiving the authorization code (step 6) and the token
exchange at the back-end (step 7), there is an intermediate step: **the browser
delivers the code to the application back-end** via the redirect URI callback.
The browser never touches the token endpoint — only the back-end does, using the
client secret. Walking the flow manually in this lab (and hitting the 60-second
code expiry twice) makes this step concrete rather than theoretical.

---

## ⚖️ Legal & Ethical Notice

All activities in this lab were performed exclusively in an isolated personal lab environment, using self-created test accounts and clients. No external or unauthorised systems were accessed. All work complies with Swiss law and ethical standards.
