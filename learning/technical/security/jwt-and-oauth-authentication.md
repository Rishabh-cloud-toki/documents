# JWT, OAuth 2.0 & OpenID Connect — Authentication and Authorization

Architect-level reference on how modern applications authenticate users and
authorize API calls: the OAuth 2.0 / OpenID Connect (OIDC) protocols, the JWT
token format, how to validate tokens correctly, how to choose a grant type,
where to store tokens in a browser, how to propagate identity across
microservices, and how to handle refresh, revocation and logout. It ends with a
worked **Angular + Okta + Spring Boot** implementation, threats and
mitigations, anti-patterns and an architect checklist.

Related: security-architecture items in
[../architecture/architecture-and-design-patterns-checklist.md](../architecture/architecture-and-design-patterns-checklist.md)
and Spring-specific material in
[../backend-and-messaging/spring-boot-and-microservices-qa.md](../backend-and-messaging/spring-boot-and-microservices-qa.md).

## Contents

- [Core idea: separate who you are from what you may do](#core-idea-separate-who-you-are-from-what-you-may-do)
- [Vocabulary: the four OAuth roles](#vocabulary-the-four-oauth-roles)
- [OAuth 2.0 vs OpenID Connect vs SAML](#oauth-20-vs-openid-connect-vs-saml)
- [The three tokens: access, ID and refresh](#the-three-tokens-access-id-and-refresh)
- [Opaque (reference) tokens vs JWT (self-contained) tokens](#opaque-reference-tokens-vs-jwt-self-contained-tokens)
- [JWT anatomy](#jwt-anatomy)
- [Signing algorithms and key management](#signing-algorithms-and-key-management)
- [JWT validation checklist](#jwt-validation-checklist)
- [Choosing an OAuth grant type](#choosing-an-oauth-grant-type)
- [Authorization Code + PKCE in detail](#authorization-code--pkce-in-detail)
- [Token storage in the browser and the BFF pattern](#token-storage-in-the-browser-and-the-bff-pattern)
- [Refresh tokens: lifetimes, rotation and reuse detection](#refresh-tokens-lifetimes-rotation-and-reuse-detection)
- [Scopes, roles, claims and where authorization lives](#scopes-roles-claims-and-where-authorization-lives)
- [Spring Boot as a resource server](#spring-boot-as-a-resource-server)
- [Identity propagation across microservices](#identity-propagation-across-microservices)
- [Revocation and logout](#revocation-and-logout)
- [Sender-constrained tokens: DPoP and mTLS](#sender-constrained-tokens-dpop-and-mtls)
- [Worked example: Angular + Okta (OIDC)](#worked-example-angular--okta-oidc)
- [Threats and mitigations](#threats-and-mitigations)
- [When not to use JWTs](#when-not-to-use-jwts)
- [Operational concerns: performance, observability, multi-tenancy](#operational-concerns-performance-observability-multi-tenancy)
- [Anti-patterns](#anti-patterns)
- [Architect checklist](#architect-checklist)
- [Interview Q&A](#interview-qa)
- [References](#references)

---

## Core idea: separate who you are from what you may do

- **Authentication (AuthN)** answers *"who is this?"* — proving identity
  (password, passkey, MFA, federation with a corporate IdP).
- **Authorization (AuthZ)** answers *"what may they do?"* — deciding whether an
  identified caller may perform an action on a resource.
- Modern architectures **centralize authentication** in an **Identity Provider /
  Authorization Server** (Okta, Auth0, Microsoft Entra ID, Keycloak, AWS
  Cognito, Ping) and **decentralize authorization enforcement** into each API,
  which trusts **tokens** issued by that server.
- Benefits of this split:
  - Applications never see passwords → smaller blast radius, one place for MFA,
    password policy, account lockout and audit.
  - Single sign-on (SSO) across many apps.
  - APIs can verify a caller **without calling back** to the IdP on every
    request (when tokens are self-contained JWTs).
- The trade-off: you now depend on **correct token handling** in every client
  and every service. Most real-world breaches in this area come from
  misconfiguration (skipped validation, tokens in logs, wrong grant type), not
  from flaws in the protocols themselves.

---

## Vocabulary: the four OAuth roles

| Role | What it is | Example in this note |
|---|---|---|
| **Resource Owner** | The entity that owns the data — usually the end user | The person logging in |
| **Client** | The application requesting access *on behalf of* the resource owner (or itself) | Angular SPA, mobile app, a batch job |
| **Authorization Server (AS)** | Authenticates the resource owner, obtains consent, issues tokens | Okta (`https://<okta-domain>/oauth2/default`) |
| **Resource Server (RS)** | The API that hosts protected resources and accepts tokens | Spring Boot microservices |

Other terms you will meet:

- **Confidential client** — can keep a secret (server-side web app, backend
  service). Authenticates to the AS with a `client_secret`, `private_key_jwt`
  or mTLS.
- **Public client** — cannot keep a secret (SPA, mobile/desktop app). Any
  "secret" shipped to the browser or app binary is public. Must use **PKCE**.
- **Scope** — a string that names a *permission the client is asking for*
  (`orders:read`, `openid`, `offline_access`).
- **Claim** — a name/value statement inside a token (`sub`, `email`,
  `groups`).
- **Issuer (`iss`)** — the unique identifier (URL) of the AS that minted the
  token.
- **Audience (`aud`)** — who the token is *meant for*. An API must reject
  tokens whose audience is not itself.
- **Discovery document** — `<issuer>/.well-known/openid-configuration`, a JSON
  document listing the AS's endpoints (`authorization_endpoint`,
  `token_endpoint`, `jwks_uri`, …). Libraries use it so you configure only the
  issuer URL.

---

## OAuth 2.0 vs OpenID Connect vs SAML

| | **OAuth 2.0** | **OpenID Connect (OIDC)** | **SAML 2.0** |
|---|---|---|---|
| Purpose | **Delegated authorization** — let a client call an API on a user's behalf | **Authentication** layer on top of OAuth 2.0 — tells the client *who the user is* | Authentication / SSO federation (mainly enterprise web) |
| Main artifact | Access token (format unspecified) | **ID token** (always a JWT) + access token | XML assertion |
| Transport | JSON over HTTPS, redirects | Same as OAuth | XML via browser POST / redirect |
| Good for | APIs, mobile, SPAs, service-to-service | Login for any modern app | Legacy enterprise SSO, B2B federation |
| Key extras | Scopes, grants | `openid` scope, `nonce`, `/userinfo`, discovery, standard claims, logout specs | Signed/encrypted XML, metadata exchange |

Key points:

- **OAuth 2.0 alone is not an authentication protocol.** An access token says
  "the bearer may call this API", not "this user just logged in". Building
  login purely on access tokens ("pseudo-authentication") led to many
  vulnerabilities — OIDC exists to fix that.
- Requesting the **`openid`** scope turns an OAuth flow into an OIDC flow and
  returns an **ID token**.
- **SAML** is still common for enterprise SSO. A typical modern pattern is: the
  corporate IdP speaks SAML to Okta/Entra, and Okta/Entra speaks OIDC/OAuth to
  your applications. Your apps never deal with SAML directly.
- **OAuth 2.1** (IETF draft) consolidates best practice: PKCE required for all
  authorization-code clients, Implicit and Password grants removed, exact
  redirect-URI matching, refresh tokens for public clients must be
  sender-constrained or rotated. Design to OAuth 2.1 even if your vendor
  still says "2.0".

---

## The three tokens: access, ID and refresh

| | **Access token** | **ID token** | **Refresh token** |
|---|---|---|---|
| Defined by | OAuth 2.0 | OIDC | OAuth 2.0 |
| Audience | The **API** (resource server) | The **client** (`aud` = `client_id`) | The **AS** only |
| Purpose | Authorize API calls | Tell the client who logged in, how and when | Get new access tokens without re-login |
| Format | JWT or opaque | Always a JWT | Usually opaque |
| Sent to | APIs, in `Authorization: Bearer …` | **Nobody** — consumed by the client | Only the AS `/token` endpoint |
| Typical lifetime | 5–60 min | Minutes (used once at login) | Hours to days (with rotation) |

Rules that are often broken:

- **Never send the ID token to an API as a bearer credential.** Its audience is
  the client, it carries no scopes, and APIs that accept it can be fooled by an
  ID token issued to *another* app.
- **The client should treat the access token as opaque.** Even if it happens
  to be a JWT, only the API is the intended reader. The format can change
  without notice.
- **The refresh token is the most valuable token** (long-lived, mints new
  access tokens). Protect it most carefully: rotation, sender-constraining,
  server-side storage where possible.

---

## Opaque (reference) tokens vs JWT (self-contained) tokens

**Opaque token**: a random string (`2YotnFZFEjr1zCsicMWpAA`). The API must ask
the AS what it means via **token introspection** (RFC 7662):

```http
POST /oauth2/v1/introspect
Authorization: Basic <api-client-credentials>
Content-Type: application/x-www-form-urlencoded

token=2YotnFZFEjr1zCsicMWpAA&token_type_hint=access_token
```

```json
{ "active": true, "sub": "00u1abc", "scope": "orders:read", "exp": 1767225600, "client_id": "0oa9xyz" }
```

**JWT token**: the claims are inside the token, signed by the AS. The API checks
the signature locally with the AS's public key — no network call.

| Concern | Opaque + introspection | Self-contained JWT |
|---|---|---|
| Validation cost | Network call per request (cacheable) | Local CPU (signature check) |
| AS availability | AS outage = API outage (unless cached) | APIs keep working while the AS is down until tokens expire |
| Revocation | **Immediate** — AS returns `active:false` | Hard — token valid until `exp` unless you add a denylist |
| Privacy | Claims stay server-side | Claims readable by anyone holding the token (Base64, not encrypted) |
| Size | Small | 1–2 KB+, grows with claims |
| Best for | Public-facing tokens, high-sensitivity, need for instant revocation | Internal service-to-service, high throughput, offline validation |

**Phantom token / split token pattern** — get both benefits: issue **opaque**
tokens to external clients (nothing leaks if intercepted), and have the **API
gateway** introspect once and swap it for a **JWT** that flows to internal
services. See [Identity propagation](#identity-propagation-across-microservices).

---

## JWT anatomy

A JWT (RFC 7519) is three Base64URL-encoded parts joined by dots:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1rZzEifQ . eyJpc3MiOiJodHRwczovL...  . SflKxwRJSMeKKF2QT4fwpMeJf36P...
            HEADER                                PAYLOAD                       SIGNATURE
```

**Header** — how the token is protected:

```json
{
  "alg": "RS256",          // signing algorithm
  "kid": "Mkg1",           // key id: which key in the JWKS signed this token
  "typ": "at+jwt"          // RFC 9068 type for access tokens (prevents token-type confusion)
}
```

**Payload** — the claims. A realistic decoded Okta access token:

```json
{
  "ver": 1,
  "jti": "AT.f3k2Jx9...",                 // unique token id (for denylists, replay detection)
  "iss": "https://acme.okta.com/oauth2/default",
  "aud": "api://orders",                  // which API this token is for
  "iat": 1767222000,                      // issued at   (seconds since epoch)
  "nbf": 1767222000,                      // not before
  "exp": 1767225600,                      // expires at  (1 hour later)
  "cid": "0oa9xyzSpaClient",              // client that requested it
  "uid": "00u1abc",
  "sub": "rishabh@acme.com",              // the subject (user, or client for M2M)
  "scp": ["openid", "orders:read"],       // granted scopes
  "groups": ["Order-Admins", "Everyone"]  // custom claim configured on the AS
}
```

| Registered claim | Meaning | Validate? |
|---|---|---|
| `iss` | Issuer | **Yes** — exact match to the trusted issuer |
| `sub` | Subject — stable unique id of the principal | Use as the user key (not `email`, which can change) |
| `aud` | Audience | **Yes** — must contain *this* API |
| `exp` | Expiry | **Yes** (with small clock-skew allowance) |
| `nbf` | Not before | Yes |
| `iat` | Issued at | Optional: reject absurdly old tokens |
| `jti` | Token id | Optional: replay detection / revocation lists |

**Signature** — `RS256` means: `RSASSA-PKCS1-v1_5( SHA-256( base64url(header) + "." + base64url(payload) ), privateKey )`.
Anyone with the public key can verify it; only the AS can create it.

Critical facts:

- **Base64URL is encoding, not encryption.** Anyone can paste a JWT into
  jwt.io and read it. **Never put secrets or sensitive PII** (national IDs,
  medical data) in a signed JWT.
- A signed JWT is technically a **JWS** (RFC 7515). If confidentiality is
  required, use a **JWE** (RFC 7516, encrypted — five parts) or keep the token
  opaque. Nested JWT (sign then encrypt) gives both integrity and
  confidentiality.
- JWTs are **immutable**. To change a claim (new role) you need a new token, so
  permission changes take effect only after the current token expires.

---

## Signing algorithms and key management

| Algorithm | Type | Key | Use it when | Notes |
|---|---|---|---|---|
| **HS256** | Symmetric HMAC | One shared secret | Single service signs *and* verifies its own tokens | Every verifier can also **mint** tokens. Avoid in microservices. |
| **RS256** | Asymmetric RSA | Private key signs, public key verifies | Default for most IdPs; broad library support | Keys ≥ 2048 bits; larger signatures |
| **PS256** | Asymmetric RSA-PSS | Same as RS256 | Required by some high-security profiles (FAPI) | Probabilistic padding, more modern than PKCS#1 v1.5 |
| **ES256** | Asymmetric ECDSA P-256 | EC key pair | Smaller tokens, faster signing | Needs good randomness when signing |
| **EdDSA** (Ed25519) | Asymmetric | EC key pair | Modern, fast, deterministic | Check library and IdP support |
| **none** | No signature | — | **Never** | Must be rejected by every verifier |

**Why asymmetric for distributed systems:** with RS256/ES256 only the AS holds
the private key. Fifty microservices can hold the public key and verify tokens,
but none of them can forge one. With HS256, a compromise of any one service
lets an attacker mint admin tokens for all of them.

### JWKS and key rotation

The AS publishes its **public** keys as a **JSON Web Key Set** (RFC 7517) at the
`jwks_uri` from the discovery document:

```json
{
  "keys": [
    { "kty": "RSA", "kid": "Mkg1", "use": "sig", "alg": "RS256", "n": "0vx7agoebG...", "e": "AQAB" },
    { "kty": "RSA", "kid": "Pz7q", "use": "sig", "alg": "RS256", "n": "xjlCRBqkOq...", "e": "AQAB" }
  ]
}
```

Rotation without downtime:

1. AS **publishes the new key** in the JWKS alongside the old one (pre-publish).
2. Verifiers refresh their cached JWKS (they already re-fetch when they meet an
   unknown `kid`).
3. AS **starts signing** with the new key.
4. After the maximum token lifetime has passed, AS **removes the old key**.

Architect guidance: cache the JWKS (minutes to hours), refetch on unknown
`kid` with a rate limit (so attackers can't force fetch storms with random
`kid`s), and plan for **emergency rotation** after a key compromise.

---

## JWT validation checklist

Every resource server must perform **all** of these, in this order. Use a
well-maintained library (Spring Security / Nimbus, `jose`, `PyJWT`,
`Microsoft.IdentityModel`) — do not hand-roll.

1. **Parse** the token; reject anything not well-formed (3 parts for JWS).
2. **Pin the algorithm.** Accept only the algorithms you expect (e.g. `RS256`).
   Never let the token's `alg` header choose. Reject `none`.
3. **Select the key** by `kid` from the trusted issuer's JWKS — never from a URL
   in the token (`jku`/`x5u` headers) unless explicitly allow-listed.
4. **Verify the signature.**
5. **Check `iss`** — exact string match against the configured issuer.
6. **Check `aud`** — must contain this API's identifier. *(Most frequently
   forgotten check — without it, a token minted for any other app in the same
   tenant is accepted.)*
7. **Check time claims** — `exp` in the future, `nbf` in the past, allowing
   ~30–60 s clock skew.
8. **Check token type** — e.g. `typ: at+jwt` so an ID token can't be replayed
   as an access token.
9. **Check scopes / roles** required by the specific endpoint (authorization,
   not just authentication).
10. **Optional:** `jti` against a revocation denylist, `cnf` binding for
    DPoP/mTLS tokens, `azp`/`cid` allow-list of permitted clients.

Classic attacks this prevents:

- **`alg: none`** — attacker removes the signature and sets `alg` to `none`.
- **Algorithm confusion (RS256 → HS256)** — attacker changes `alg` to `HS256`
  and signs with the server's *public* key as the HMAC secret. A naive library
  that uses "the key" for whatever `alg` says will accept it.
- **Cross-audience replay** — a token issued for the low-risk "newsletter" API
  is used against the "payments" API in the same tenant.
- **`kid` injection** — `kid` used unsafely in a SQL query or file path to look
  up the key.

---

## Choosing an OAuth grant type

| Scenario | Grant | Notes |
|---|---|---|
| Web app with a backend (incl. BFF for SPA) | **Authorization Code** (+ PKCE) | Confidential client; tokens stay on the server |
| SPA without a backend | **Authorization Code + PKCE** | Public client; prefer a BFF if possible |
| Native / mobile app | **Authorization Code + PKCE** via system browser | Use claimed HTTPS links or custom scheme redirect; never an embedded WebView (RFC 8252) |
| Service-to-service, no user | **Client Credentials** | `sub` = the client; secret, `private_key_jwt` or mTLS |
| TV, CLI, IoT — no browser/keyboard | **Device Authorization** (RFC 8628) | User approves on a phone at a short URL with a code |
| Service calling a downstream API on a user's behalf | **Token Exchange** (RFC 8693) or On-Behalf-Of | Swap the incoming token for one scoped to the downstream API |
| Renew an access token | **Refresh Token** | With rotation for public clients |
| ~~Browser SPA (legacy)~~ | ~~Implicit~~ | **Deprecated** — tokens in URL fragment, no refresh, leaks via history/referrer |
| ~~Trusted first-party app~~ | ~~Resource Owner Password~~ | **Deprecated** — app handles passwords, breaks MFA/SSO/phishing protection |

Decision shortcut: **Is there a user?** No → Client Credentials. Yes → can the
device show a browser? No → Device Code. Yes → **Authorization Code + PKCE**,
with the code exchanged by a backend (BFF) whenever you can.

### Client Credentials example (service-to-service)

```http
POST /oauth2/default/v1/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=client_credentials&scope=inventory:reserve
```

```json
{ "token_type": "Bearer", "expires_in": 3600, "access_token": "eyJraWQiOi...", "scope": "inventory:reserve" }
```

Spring Boot client side (`spring-boot-starter-oauth2-client`) — tokens are
fetched, cached and renewed automatically:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          inventory:
            provider: okta
            client-id: ${INVENTORY_CLIENT_ID}
            client-secret: ${INVENTORY_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: inventory:reserve
        provider:
          okta:
            issuer-uri: https://acme.okta.com/oauth2/default
```

```java
@Bean
RestClient inventoryClient(RestClient.Builder builder, OAuth2AuthorizedClientManager manager) {
    var interceptor = new OAuth2ClientHttpRequestInterceptor(manager);
    interceptor.setClientRegistrationIdResolver(request -> "inventory");
    return builder.baseUrl("https://inventory.internal").requestInterceptor(interceptor).build();
}
```

*(`OAuth2ClientHttpRequestInterceptor` is Spring Security 6.4+; on older
versions use `ServletOAuth2AuthorizedClientExchangeFilterFunction` with
`WebClient`.)* Prefer **`private_key_jwt`** or **mTLS** client authentication
over shared secrets for production workloads: no shared secret to leak or
rotate.

---

## Authorization Code + PKCE in detail

**PKCE** (Proof Key for Code Exchange, RFC 7636, "pixy") stops an attacker who
intercepts the authorization `code` (malicious app on the same device,
leaked logs, referrer headers) from exchanging it for tokens.

How it works:

1. Client generates a random **`code_verifier`** (43–128 chars) and keeps it
   secret in memory.
2. Client computes **`code_challenge = BASE64URL(SHA-256(code_verifier))`** and
   sends only the challenge in the authorize request.
3. At the token endpoint, the client sends the original **verifier**. The AS
   hashes it and compares it with the stored challenge. An attacker who stole
   only the `code` doesn't have the verifier.

```ts
// What okta-auth-js / any OIDC library does for you under the hood
const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
const challenge = base64url(await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier)));
```

Sequence:

```
 Browser/SPA                         Authorization Server (Okta)                API (Spring Boot)
     |                                         |                                    |
 1.  | generate code_verifier, state, nonce    |                                    |
 2.  |-- GET /authorize?response_type=code     |                                    |
     |   &client_id=...&redirect_uri=...       |                                    |
     |   &scope=openid profile orders:read     |                                    |
     |   &state=xyz&nonce=abc                  |                                    |
     |   &code_challenge=E9Mel...&code_challenge_method=S256                        |
     |---------------------------------------->|                                    |
 3.  |              user authenticates (password + MFA), consents                   |
 4.  |<-- 302 redirect_uri?code=SplxlO&state=xyz                                    |
 5.  | verify state == xyz (CSRF protection)   |                                    |
 6.  |-- POST /token grant_type=authorization_code                                  |
     |   &code=SplxlO&redirect_uri=...&code_verifier=dBjft...                       |
     |---------------------------------------->|                                    |
 7.  |                                         | SHA256(verifier) == challenge?     |
 8.  |<-- { access_token, id_token, refresh_token, expires_in }                     |
 9.  | validate id_token (sig, iss, aud=client_id, exp, nonce == abc)               |
10.  |-- GET /orders  Authorization: Bearer <access_token> ------------------------>|
11.  |                                         |      validate JWT (see checklist)  |
12.  |<---------------------------------------------------------------- 200 [orders]|
```

What each protection parameter defends against:

| Parameter | Protects against | Checked by |
|---|---|---|
| `state` | CSRF on the redirect (attacker logs victim into attacker's account) | Client |
| `nonce` | ID-token replay / injection | Client (inside ID token) |
| `code_challenge` / `code_verifier` | Authorization-code interception and injection | AS |
| Exact `redirect_uri` match | Code/token theft via open redirect | AS |
| Short-lived, single-use `code` | Code replay | AS |

Hardening extensions worth knowing:

- **PAR — Pushed Authorization Requests (RFC 9126)**: the client POSTs the
  authorize parameters to the AS back-channel and gets a `request_uri`, so
  parameters can't be tampered with in the browser.
- **RAR — Rich Authorization Requests (RFC 9396)**: fine-grained
  `authorization_details` ("transfer 45 EUR to account X") instead of coarse
  scopes. Used in open banking.
- **FAPI 2.0** — a security profile (PAR + PKCE + sender-constrained tokens)
  for financial-grade APIs.

---

## Token storage in the browser and the BFF pattern

Where an SPA keeps tokens is one of the most important architectural
decisions. The main threat is **XSS**: any script running in your origin can
read anything JavaScript can read.

| Option | XSS can steal token? | CSRF risk | Survives reload | Verdict |
|---|---|---|---|---|
| `localStorage` | **Yes** | No | Yes | Convenient but riskiest. Default in many SDKs (incl. okta-auth-js). |
| `sessionStorage` | **Yes** | No | Per tab | Marginally better |
| In-memory (JS variable / closure) | Harder (still can *use* it while page is compromised) | No | No — needs silent renew via refresh token or iframe | Reasonable for pure SPAs |
| Web Worker holding tokens | Harder | No | No | Good hardening for pure SPAs |
| **`HttpOnly; Secure; SameSite` cookie + BFF** | **No** — JS can't read it | Mitigated by `SameSite` + CSRF token | Yes | **Recommended** for sensitive apps |

Note that even with perfect storage, XSS lets an attacker **make requests as
the user** while the page is open. Storage choices limit *exfiltration and
reuse later*. XSS prevention (CSP, output encoding, Angular's built-in
sanitization, dependency hygiene) is still mandatory.

### Backend-for-Frontend (BFF)

The IETF "OAuth 2.0 for Browser-Based Applications" guidance recommends a
**BFF** for anything handling sensitive data:

```
 Browser (Angular)            BFF (Spring Cloud Gateway / Node)          Okta            APIs
      |  session cookie only         | holds access + refresh tokens       |               |
      |----------------------------->|  (server-side session / Redis)      |               |
      |                              |-- Authorization Code + PKCE ------->|               |
      |                              |   (confidential client w/ secret)   |               |
      |  GET /api/orders (cookie)    |                                     |               |
      |----------------------------->|-- GET /orders  Bearer <AT> ------------------------>|
      |<-----------------------------|<----------------------------------------------------|
```

- The browser **never sees a token**. It holds only an `HttpOnly`, `Secure`,
  `SameSite=Strict/Lax` session cookie.
- The BFF is a **confidential client**: it can authenticate to the AS and
  refresh tokens safely.
- Spring example: Spring Cloud Gateway with `oauth2Login()` and the
  **`TokenRelay`** filter, which attaches the user's access token to proxied
  requests:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://orders-service
          predicates: [ Path=/api/orders/** ]
          filters: [ TokenRelay=, StripPrefix=1 ]
```

- Costs: you now run a stateful component (session store, sticky sessions or
  Redis), and must handle CSRF on state-changing requests.

---

## Refresh tokens: lifetimes, rotation and reuse detection

Short access tokens limit damage from theft. Refresh tokens provide a good UX
despite that.

**Typical lifetimes** (tune per risk):

| Token | Low-risk consumer app | Enterprise / financial |
|---|---|---|
| Access token | 30–60 min | 5–15 min |
| Refresh token idle timeout | 7–30 days | 30 min–8 h |
| Refresh token absolute lifetime | 90 days | 8–24 h (re-login daily) |

**Refresh token rotation:** every use returns a *new* refresh token and
invalidates the old one.

```
RT1 --use--> AT2 + RT2   (RT1 now invalid)
RT2 --use--> AT3 + RT3   (RT2 now invalid)
```

**Reuse detection:** if an already-used refresh token (RT1) is presented again,
either the attacker or the legitimate user has a stale copy. The AS cannot
tell which, so it **revokes the entire token family**. Everyone must log in
again and the theft is contained. Okta, Auth0 and Entra implement this. Allow a
short grace period for network retries.

Other guidance:

- Public clients (SPAs, mobile): refresh tokens **must** be rotated or
  sender-constrained (DPoP).
- Request refresh tokens only when needed (`offline_access` scope).
- Revoke refresh tokens on logout (RFC 7009 `/revoke`), password change and
  account disable.
- Mobile: store in Keychain (iOS) / Keystore-backed EncryptedSharedPreferences
  (Android), never in plain preferences.

---

## Scopes, roles, claims and where authorization lives

These are often confused:

| Concept | Answers | Granted to | Example |
|---|---|---|---|
| **Scope** | What the **client app** is allowed to do on the user's behalf | The client (with user consent) | `orders:read` |
| **Role / group** | What the **user** is allowed to do in the organization | The user | `Order-Admins` |
| **Claim** | A fact about the subject, used as input to decisions | — | `department=EU`, `tenant_id=42` |

Effective permission = **intersection**: the client must have the scope **and**
the user must have the role/attribute. An admin user using a read-only
reporting app should still only be able to read.

### Authorization layers

1. **Coarse-grained at the edge** — API gateway checks a valid token and a
   required scope per route (`/orders/** requires orders:read`).
2. **Endpoint-level in the service** — `@PreAuthorize("hasAuthority('SCOPE_orders:write')")`.
3. **Fine-grained / data-level in the domain** — "may *this* user see *this*
   order?" (ownership, tenant, region). **This cannot live in a token.** It
   needs domain data at request time.

Architect guidance:

- **Keep tokens small and stable.** Put identity and coarse roles in tokens,
  not hundreds of permissions. Large tokens hit HTTP header limits (often
  8 KB) and go stale as soon as permissions change.
- For complex rules, externalize to a **policy engine**: OPA / Rego, AWS Cedar
  / Verified Permissions, or Zanzibar-style relationship-based systems
  (OpenFGA, SpiceDB) for "user X is editor of document Y" models.
- Model choices: **RBAC** (roles, simple, can explode into many roles), **ABAC**
  (attributes and policies, flexible), **ReBAC** (relationships, for sharing/
  collaboration domains).
- Always **deny by default** and make authorization checks testable.

---

## Spring Boot as a resource server

Dependency: `spring-boot-starter-oauth2-resource-server`.

### Minimal configuration

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://acme.okta.com/oauth2/default   # discovery → jwks_uri, iss check
          audiences: api://orders                            # aud check (Boot 3.1+)
```

With only `issuer-uri`, Spring Boot fetches the discovery document at startup,
finds `jwks_uri`, caches the keys, and validates **signature, `iss`, `exp`/`nbf`**
(60 s default clock skew) on every request. **`aud` is not checked unless you
configure it.** Setting `jwk-set-uri` in addition avoids the call to the
discovery endpoint at startup, so the service doesn't fail to boot if the IdP
is briefly unreachable.

### Security filter chain

```java
@Configuration
@EnableMethodSecurity               // enables @PreAuthorize
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/orders/**").hasAuthority("SCOPE_orders:read")
                .requestMatchers(HttpMethod.POST, "/orders/**").hasAuthority("SCOPE_orders:write")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable());   // safe ONLY because auth is a bearer header, not a cookie
        return http.build();
    }
}
```

### Mapping roles from the token

By default Spring maps the `scope`/`scp` claim to authorities prefixed with
`SCOPE_`. Roles usually arrive in a custom `roles` or `groups` claim, which you
map with a `JwtAuthenticationConverter`:

```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter scopes = new JwtGrantedAuthoritiesConverter(); // scp -> SCOPE_xxx

    JwtGrantedAuthoritiesConverter roles = new JwtGrantedAuthoritiesConverter();
    roles.setAuthoritiesClaimName("groups");   // or "roles", depending on the IdP
    roles.setAuthorityPrefix("ROLE_");         // Order-Admins -> ROLE_Order-Admins

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> all = new ArrayList<>(scopes.convert(jwt));
        all.addAll(roles.convert(jwt));        // keep BOTH scopes and roles
        return all;
    });
    converter.setPrincipalClaimName("sub");    // stable id as the principal name
    return converter;
}
```

Why the `ROLE_` prefix matters: `hasRole('ADMIN')` checks for the authority
`ROLE_ADMIN`, whereas `hasAuthority('X')` matches the string exactly.

Okta note: groups are **not** in access tokens by default. Add a custom
`groups` claim on the authorization server (Security → API → Authorization
Server → Claims) with a group filter.

### Method-level and data-level checks

```java
@PreAuthorize("hasRole('Order-Admins') or hasAuthority('SCOPE_orders:admin')")
public void cancelAnyOrder(UUID orderId) { ... }

@PreAuthorize("hasAuthority('SCOPE_orders:read')")
@PostAuthorize("returnObject.customerId == authentication.token.claims['sub']")  // ownership check
public Order getOrder(UUID orderId) { ... }

// Access claims in a controller
@GetMapping("/me")
public Map<String, Object> me(@AuthenticationPrincipal Jwt jwt) {
    return Map.of("sub", jwt.getSubject(), "email", jwt.getClaimAsString("email"));
}
```

### Custom validators (e.g. allowed clients, token type)

```java
@Bean
JwtDecoder jwtDecoder(OAuth2ResourceServerProperties props) {
    String issuer = props.getJwt().getIssuerUri();
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuer);

    OAuth2TokenValidator<Jwt> audience = new JwtClaimValidator<List<String>>(
        JwtClaimNames.AUD, aud -> aud != null && aud.contains("api://orders"));
    OAuth2TokenValidator<Jwt> allowedClients = new JwtClaimValidator<String>(
        "cid", cid -> Set.of("0oa9xyzSpaClient", "0oaBatchJob").contains(cid));

    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer(issuer), audience, allowedClients));
    return decoder;
}
```

(Defining your own `JwtDecoder` bean replaces Boot's auto-configured one, so
include the audience check here instead of relying on the `audiences`
property.)

### Testing

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;

    @Test
    void readRequiresScope() throws Exception {
        mvc.perform(get("/orders/1").with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders:read"))))
           .andExpect(status().isOk());
        mvc.perform(get("/orders/1").with(jwt()))   // token without the scope
           .andExpect(status().isForbidden());
        mvc.perform(get("/orders/1"))                // no token
           .andExpect(status().isUnauthorized());
    }
}
```

`401 Unauthorized` = no/invalid token (authentication failed). `403
Forbidden` = valid token but insufficient permission. Spring returns a
`WWW-Authenticate: Bearer error="invalid_token"` header on 401s (RFC 6750).

---

## Identity propagation across microservices

When the Angular app calls `orders-service`, which calls `inventory-service`
and `payment-service`, how does identity flow? Options, from simplest to most
secure:

### 1. Token relay (pass-through)

Forward the user's access token unchanged to downstream services.

- Simple and preserves user context end-to-end.
- **Problems:** the token's `aud` must include every downstream API (an
  over-broad token). Any compromised service can replay it to any other
  service. Scopes can't be narrowed per hop.
- Acceptable inside a tightly trusted boundary for read-only call chains.

### 2. Edge validation + internal trust

The API gateway validates the external token. Internal services trust headers
(`X-User-Id`) set by the gateway.

- **Dangerous unless** the network guarantees only the gateway can reach
  services (zero trust says it can't). Any internal attacker can forge
  headers. If you do this, at least sign the internal assertion (see 4).

### 3. Token exchange (RFC 8693) / On-Behalf-Of

Each service exchanges the incoming token at the AS for a new token
**scoped and audienced for the specific downstream API**:

```http
POST /token
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<incoming user access token>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&audience=api://payments
&scope=payments:charge
```

The new token keeps the user as `sub` and records the calling service in an
`act` (actor) claim. This gives least privilege per hop and a clear audit
trail ("payments called by orders on behalf of user X"). Microsoft Entra's
"On-Behalf-Of" flow is the same idea. Cost: an extra AS round-trip (cache the
result per user+audience).

### 4. Phantom / internal token at the gateway

The gateway introspects the external opaque token, then mints a short-lived
**internal JWT** (signed by an internal key, internal audience) that carries
normalized identity to internal services. Externals never see internal
claims. Internals validate locally.

### 5. Workload identity for service-to-service (no user)

- **Client Credentials** per service: each service has its own identity and
  scopes.
- **mTLS with a service mesh** (Istio, Linkerd) and **SPIFFE/SPIRE** identities
  (`spiffe://acme/ns/prod/sa/orders`) authenticate the *calling workload*,
  combined with mesh authorization policies.
- Cloud-native: AWS IAM roles, GCP Workload Identity, Azure Managed Identity —
  avoid long-lived secrets entirely.

**Recommended combination:** mTLS / workload identity proves **which service**
is calling. A token (exchanged or relayed) proves **which user** it's acting
for. Each service authorizes on both. This prevents the **confused deputy**
problem: a service with broad privileges being tricked into using them on
behalf of a caller who shouldn't have them.

---

## Revocation and logout

The weak spot of self-contained JWTs: once issued, they're valid until `exp`.

| Technique | How it works | Trade-off |
|---|---|---|
| **Short access-token lifetime** | 5–15 min; revoke refresh tokens instead | Simplest; window of exposure = lifetime |
| **Refresh-token revocation** (RFC 7009) | `POST /revoke` on logout / compromise | Stops *new* access tokens; existing ones live until expiry |
| **Introspection** | APIs ask the AS on each (or cached) request | Real-time, but adds latency and AS dependency |
| **Denylist by `jti` / `sub`** | Revoked ids pushed to Redis / via events; checked after signature | Small, short-lived list (entries expire with tokens); extra lookup |
| **"Tokens issued before" timestamp** | Store `revoked_before` per user; reject tokens with older `iat` | Cheap way to "log out everywhere" |
| **Continuous Access Evaluation (CAE)** | AS pushes critical events (user disabled, IP change) to RPs (Shared Signals / CAEP) | Emerging standard; vendor support varies |

### Logout is several layers

1. **Local app session** — clear tokens / BFF session.
2. **IdP session (SSO cookie)** — otherwise the next "Login" silently signs the
   user back in. Use **OIDC RP-Initiated Logout**: redirect to
   `end_session_endpoint?id_token_hint=…&post_logout_redirect_uri=…`.
3. **Other apps in the SSO session** — **OIDC Back-Channel Logout**: the IdP
   POSTs a signed `logout_token` to each app's back-channel endpoint, so
   server-side sessions are killed. (Front-Channel Logout uses hidden iframes
   and is unreliable now that browsers block third-party cookies.)
4. **Refresh tokens** — revoke them.

```ts
// okta-auth-js: revokes tokens and ends the Okta session, then redirects
await oktaAuth.signOut({ postLogoutRedirectUri: window.location.origin + '/goodbye' });
```

---

## Sender-constrained tokens: DPoP and mTLS

A **bearer token** works for *whoever holds it* — like cash. Sender-constraining
binds the token to a key held by the legitimate client, so a stolen token is
useless.

**DPoP** (RFC 9449) — application-layer proof of possession, works in browsers:

1. Client generates a key pair (non-extractable via WebCrypto).
2. With every token request **and** API call it sends a `DPoP` header: a JWT
   signed by its private key containing the HTTP method, URL, timestamp and
   a hash of the access token.
3. The AS binds the token to the key thumbprint (`"cnf": {"jkt": "..."}`).
4. The API verifies that the DPoP proof is signed by the bound key.

```http
GET /orders HTTP/1.1
Authorization: DPoP eyJhbGciOiJSUzI1NiIs...      <- note: scheme is DPoP, not Bearer
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7...
```

**mTLS-bound tokens** (RFC 8705) — the token is bound to the client's TLS
certificate (`"cnf": {"x5t#S256": "..."}`). Natural for service-to-service and
FAPI. Hard in browsers.

Use sender-constraining for high-value APIs (payments, health, admin) and for
refresh tokens in public clients. Okta, Auth0, Keycloak and Spring Security
(6.5+) support DPoP.

---

## Worked example: Angular + Okta (OIDC)

The flow below uses **Authorization Code + PKCE** with tokens held in the SPA.
For high-sensitivity apps, run the same flow through a [BFF](#backend-for-frontend-bff).

### 0. Okta setup

- Create an app integration of type **OIDC → Single-Page Application**. This
  makes it a public client, enforces PKCE, and disables the Implicit grant.
- Grant types: **Authorization Code**, and **Refresh Token** with **rotation**
  enabled.
- Sign-in redirect URI: `https://your-app.com/login/callback` (exact match).
  Sign-out redirect URI: `https://your-app.com`.
- **Trusted Origins** → add the SPA origin for CORS.
- On the **Authorization Server** (`default` or a custom one): set audience
  (e.g. `api://orders`), define scopes (`orders:read`, `orders:write`), add a
  `groups` claim, and set access policies and token lifetimes.

### 1. Configure the SDK

```ts
// app.config.ts (standalone Angular, @okta/okta-angular + @okta/okta-auth-js)
import { OktaAuth } from '@okta/okta-auth-js';
import { OktaAuthModule } from '@okta/okta-angular';

const oktaAuth = new OktaAuth({
  issuer: 'https://acme.okta.com/oauth2/default',
  clientId: '0oa9xyzSpaClient',
  redirectUri: window.location.origin + '/login/callback',
  postLogoutRedirectUri: window.location.origin,
  scopes: ['openid', 'profile', 'email', 'offline_access', 'orders:read'],
  pkce: true,                                   // default for SPAs; shown for clarity
  tokenManager: {
    storage: 'sessionStorage',                  // default is localStorage; 'memory' is safest
    autoRenew: true,                            // renew shortly before expiry (default true)
    autoRemove: true,                           // remove expired tokens if renewal fails
  },
});

export const appConfig: ApplicationConfig = {
  providers: [
    importProvidersFrom(OktaAuthModule.forRoot({ oktaAuth })),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideRouter(routes),
  ],
};
```

### 2. Initial login trigger

```ts
await this.oktaAuth.signInWithRedirect({ originalUri: '/orders' });
```

The SDK generates `state`, `nonce` and the PKCE `code_verifier`/`code_challenge`,
stores them temporarily, and redirects the browser to:

```
https://acme.okta.com/oauth2/default/v1/authorize
  ?client_id=0oa9xyzSpaClient
  &response_type=code
  &scope=openid%20profile%20email%20offline_access%20orders%3Aread
  &redirect_uri=https%3A%2F%2Fyour-app.com%2Flogin%2Fcallback
  &state=Xk2...&nonce=Qa9...
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
```

### 3. User authenticates on the Okta-hosted page

- The user enters credentials and completes MFA. **Your app never sees the
  password.** Using the hosted page (rather than an embedded login form) keeps
  phishing resistance, MFA and passkeys working.
- If the user already has an Okta session (SSO), this step is skipped.
- Okta redirects back to:

```
https://your-app.com/login/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=Xk2...
```

### 4. Token exchange (handled by the SDK)

`OktaCallbackComponent` (or `oktaAuth.handleLoginRedirect()`) verifies `state`
and then calls the token endpoint:

```http
POST /oauth2/default/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=0oa9xyzSpaClient
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://your-app.com/login/callback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Note: there is no `client_secret`. PKCE replaces it for public clients.

Okta responds:

```json
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "eyJraWQiOi...",
  "id_token": "eyJraWQiOi...",
  "refresh_token": "8xLOxBtZp8...",          // only if offline_access was requested and allowed
  "scope": "openid profile email offline_access orders:read"
}
```

The SDK validates the ID token (signature, `iss`, `aud` = client id, `exp`,
`nonce`) and stores the tokens in the configured storage.

### 5. Routes, guards and redirect after login

```ts
export const routes: Routes = [
  { path: 'login/callback', component: OktaCallbackComponent },
  { path: 'orders', component: OrdersComponent, canActivate: [OktaAuthGuard] },
  { path: 'admin', component: AdminComponent, canActivate: [OktaAuthGuard],
    data: { okta: { acrValues: 'urn:okta:loa:2fa:any' } } },  // step-up MFA for admin area
];
```

- Default: after login, the user returns to the `originalUri` they tried to
  open (stored before redirect).
- To customize, write your own callback component:

```ts
@Component({ template: '<p>Signing you in…</p>' })
export class CustomLoginCallbackComponent implements OnInit {
  constructor(@Inject(OKTA_AUTH) private oktaAuth: OktaAuth, private router: Router) {}

  async ngOnInit() {
    try {
      await this.oktaAuth.handleLoginRedirect();      // exchanges code, stores tokens
      this.router.navigate(['/menu']);                // custom landing page
    } catch (e) {
      this.router.navigate(['/login-error'], { state: { error: String(e) } });
    }
  }
}
```

- UI guards are for **user experience only**. Real enforcement is on the API.
  Hiding an "Admin" button does not protect the admin endpoint.

### 6. Attaching the token to API calls

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const oktaAuth = inject(OKTA_AUTH);
  // Only send tokens to YOUR APIs - never to third-party URLs
  if (!req.url.startsWith('https://api.your-app.com')) return next(req);

  const token = oktaAuth.getAccessToken();
  return next(token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req);
};
```

The allow-list check matters: a naive interceptor leaks the user's token to
every analytics or CDN endpoint the app calls.

### 7. Access-token refresh

**Automatic.** With `autoRenew: true`, the token manager renews tokens shortly
before expiry. If a refresh token exists (`offline_access` scope and Refresh
Token grant enabled), it calls `/token` with `grant_type=refresh_token`.
Otherwise it falls back to a hidden-iframe silent auth (`prompt=none`), which
**breaks when browsers block third-party cookies**. That is why refresh tokens
with rotation are now the recommended approach for SPAs.

```http
POST /oauth2/default/v1/token
grant_type=refresh_token&client_id=0oa9xyzSpaClient&refresh_token=8xLOxBtZp8...&scope=openid orders:read
```

With rotation enabled, the response contains a **new** `refresh_token`, and the
old one is invalidated.

**Manual refresh** (e.g. after the user's roles change and you want new claims
immediately):

```ts
await this.oktaAuth.tokenManager.renew('accessToken');
// or renew everything:
const tokens = await this.oktaAuth.token.renewTokens();
this.oktaAuth.tokenManager.setTokens(tokens);
```

**Handle renewal failure and expiry:**

```ts
this.oktaAuth.tokenManager.on('expired', key => console.info(`${key} expired`));
this.oktaAuth.tokenManager.on('renewed', (key, newToken) => console.info(`${key} renewed`));
this.oktaAuth.tokenManager.on('error', async err => {
  console.error('Token renewal failed', err);       // e.g. refresh token revoked / session ended
  await this.oktaAuth.signInWithRedirect({ originalUri: this.router.url });  // or sign out
});
```

**401 handling in the interceptor.** If an API returns 401 (token revoked or
expired early), try one renewal and retry the request once. Never retry in a
loop:

```ts
return next(authReq).pipe(
  catchError(err => err.status === 401 && !req.headers.has('X-Retry')
    ? from(oktaAuth.tokenManager.renew('accessToken')).pipe(
        switchMap(t => next(req.clone({ setHeaders: { Authorization: `Bearer ${(t as AccessToken).accessToken}`, 'X-Retry': '1' } }))))
    : throwError(() => err)));
```

### 8. Summary diagram of the end-to-end flow

```
[User] -- clicks Login
   |
[Angular] -- generates state, nonce, PKCE verifier/challenge
   |        -- redirects to Okta /authorize (with code_challenge)
   v
[Okta hosted login] -- user authenticates (+MFA), or SSO session reused
   |
   v  302 to /login/callback?code=...&state=...
[Angular callback] -- verifies state
   |               -- POST /token (code + code_verifier)
   v
[Okta] -- returns access_token, id_token, refresh_token
   |
[Angular] -- validates id_token, stores tokens, navigates to originalUri (e.g. /menu)
   |
   |  every API call: Authorization: Bearer <access_token>   (interceptor, own APIs only)
   v
[Spring Boot API] -- validates signature (JWKS), iss, aud, exp, scopes/roles
   |              -- returns 200 / 401 (bad token) / 403 (not allowed)
   |
   |  before expiry: tokenManager autoRenew -> POST /token grant_type=refresh_token
   |  logout: signOut() -> revoke tokens + Okta end_session -> postLogoutRedirectUri
```

---

## Threats and mitigations

| Threat | Example | Mitigation |
|---|---|---|
| XSS token theft | Injected script reads `localStorage` | BFF + HttpOnly cookies, CSP, in-memory storage, Angular sanitization, short tokens, DPoP |
| CSRF | Cookie-authenticated POST from evil site | `SameSite` cookies, CSRF tokens (BFF); bearer-header APIs are not CSRF-prone |
| Authorization-code interception | Malicious app registers same custom URL scheme | PKCE, claimed HTTPS redirects |
| Login CSRF / session fixation | Attacker injects their own code | `state` parameter, PKCE |
| Open redirect | `redirect_uri=https://evil.com` | Exact redirect-URI registration and matching |
| Token leakage | Tokens in URLs, logs, error reports, `Referer` | Never use Implicit, never log tokens, scrub APM/log pipelines |
| Token replay from another API | Token for API A used on API B | Validate `aud`; per-API audiences; token exchange |
| Algorithm attacks | `alg:none`, RS→HS confusion | Pin algorithms, use vetted libraries |
| Signing key compromise | Private key leaked | Keys in HSM/KMS, rotation, emergency rotation runbook |
| Refresh-token theft | Stolen from device storage | Rotation + reuse detection, DPoP, short idle timeout |
| Privilege staleness | User removed from admin, token still says admin | Short access tokens, revoke on role change, fine-grained checks at runtime |
| Confused deputy | Service uses its own broad privileges for a caller | Token exchange, check end-user identity and calling service |
| Mix-up attack | Client with multiple IdPs sends code to the wrong one | `iss` in authorization response (RFC 9207), per-IdP redirect URIs |
| Phishing / credential stuffing | Fake login page | Hosted login, MFA, **passkeys / WebAuthn**, breached-password detection |

---

## When not to use JWTs

JWTs are the right tool for **stateless, distributed API authorization**. They
are often the wrong tool for:

- **Browser sessions of a single server-rendered web app.** A classic
  server-side session (opaque session id in an `HttpOnly` cookie, state in
  Redis) is simpler, revocable instantly, and smaller.
- **Anything needing instant revocation** (banking session kill-switch). Use
  opaque tokens + introspection, or very short JWTs plus a denylist.
- **Storing changing state** (cart, preferences). Tokens are immutable
  snapshots.
- **Long-lived API keys for third-party developers.** Use opaque API keys or
  client credentials with managed rotation.
- **Carrying large permission sets.** These hit header-size limits; use a
  policy service instead.

---

## Operational concerns: performance, observability, multi-tenancy

**Performance**

- RS256 verification costs on the order of tens of microseconds. It is rarely a
  bottleneck. JWKS fetching on cold start and on unknown `kid` *can* be: cache
  keys and pre-warm them.
- Token size × request rate = bandwidth. Avoid stuffing claims. Watch for
  `431 Request Header Fields Too Large` at proxies/load balancers.
- Introspection: cache results for a short TTL (≤ token lifetime, e.g.
  30–60 s) keyed by a token hash.
- The IdP is a **critical dependency**. Know its SLA and rate limits. Design so
  that existing sessions keep working during an IdP outage (JWT validation is
  local; refresh will fail, so access tokens shouldn't be *too* short).

**Observability & audit**

- **Never log raw tokens**, authorization codes or refresh tokens. Log `sub`,
  `cid`/`client_id`, `jti`, scopes and the decision. Add log-scrubbing rules
  for `Authorization` headers.
- Emit metrics: `401` vs `403` rates, token validation failures by reason
  (expired, bad signature, wrong audience), refresh failures, reuse-detection
  events. A spike in "bad signature" usually means a misconfiguration or a
  key rotation issue. A spike in reuse detection means token theft.
- Put the user id (`sub`) into the trace/log context for incident analysis
  (mind GDPR: pseudonymous ids, not emails).

**Multi-tenancy**

- One issuer per tenant (Okta org/AS per customer, Entra per tenant) →
  resolve the issuer dynamically. Spring Security provides
  `JwtIssuerAuthenticationManagerResolver` with an **allow-list** of trusted
  issuers. Never trust whatever `iss` the token claims.
- Or one issuer with a `tenant_id` claim → enforce tenant isolation in *every*
  data query, not just at the gateway.

**Environments & configuration**

- Separate authorization servers / client registrations per environment
  (dev/test/prod). A dev token must never be valid in prod. `iss` and `aud`
  checks enforce this.
- Manage client secrets and signing keys in a vault (HashiCorp Vault, AWS
  Secrets Manager / KMS, Azure Key Vault), never in Git.

---

## Anti-patterns

- Using the **ID token** to call APIs.
- Skipping **`aud` validation** ("we only check signature and expiry").
- **HS256 with a shared secret** across many services.
- **Implicit grant** or **Resource Owner Password** grant in new systems.
- Long-lived (days) **access tokens** "to avoid refresh complexity".
- Tokens in **URLs / query strings** (end up in logs, history, `Referer`).
- Putting **sensitive data** in JWT payloads assuming they're encrypted.
- **Decoding without verifying** (`jwt.decode()` instead of `verify()`) and
  trusting the claims.
- Rolling your own crypto, JWT parser or OAuth server.
- Trusting **`X-User-Id` headers** inside the network without authentication.
- Embedding the full **permission model** in tokens.
- Relying on **frontend route guards** for security.
- **Disabling CSRF** in a cookie-based (BFF) setup because "we use JWT".
- One **mega-audience** token accepted by every service.
- No plan for **key rotation**, **logout everywhere**, or **IdP outage**.

---

## Architect checklist

**Protocol & flow**
- [ ] OIDC for login, OAuth 2.0 for API access. Grant per client type chosen
      from the decision table (Auth Code + PKCE for all user-facing apps).
- [ ] Implicit and Password grants disabled at the IdP.
- [ ] Exact redirect URIs registered. `state`, `nonce`, PKCE (S256) used.
- [ ] Hosted login page with MFA. Passkeys/WebAuthn considered.

**Tokens**
- [ ] Asymmetric signing (RS256/ES256/PS256). Keys in KMS/HSM. Rotation via
      JWKS documented and tested.
- [ ] Access-token lifetime ≤ 15–60 min by risk. Refresh tokens rotated with
      reuse detection, idle and absolute timeouts defined.
- [ ] No sensitive data in JWT payloads. JWE or opaque tokens where needed.
- [ ] Token size budget agreed (claims reviewed).

**Resource servers**
- [ ] Every service validates signature, `alg`, `iss`, `aud`, `exp`/`nbf`,
      token type, scopes. Vetted library only.
- [ ] Deny-by-default authorization. Scope + role checks per endpoint;
      ownership/tenant checks in the domain layer.
- [ ] 401 vs 403 semantics correct. Auth tests in CI (no token, wrong scope,
      expired, wrong audience).

**Browser / mobile**
- [ ] Token storage decision recorded (BFF for sensitive apps). CSP enabled.
- [ ] HTTP interceptor sends tokens only to allow-listed API origins.
- [ ] Mobile uses system browser + secure OS key storage.

**Microservices**
- [ ] Identity propagation pattern chosen (token exchange / internal token /
      relay) and documented. No unauthenticated trust of internal headers.
- [ ] Workload identity (mTLS/SPIFFE or client credentials) for
      service-to-service calls.

**Lifecycle & operations**
- [ ] Logout covers app session, IdP session (RP-initiated), other apps
      (back-channel), refresh tokens.
- [ ] Revocation strategy for compromised accounts ("log out everywhere").
- [ ] Tokens scrubbed from logs. Auth metrics and alerts in place.
- [ ] IdP outage and emergency key-rotation runbooks exist.
- [ ] Separate IdP config per environment. Secrets in a vault.

---

## Interview Q&A

**Q: What is the difference between OAuth 2.0 and OIDC?**
OAuth 2.0 is a delegated *authorization* framework: it issues access tokens
that let a client call APIs. OIDC is an *authentication* layer on top: adding
the `openid` scope returns an ID token (a JWT) describing who the user is and
how they authenticated, plus standard claims, `/userinfo`, discovery and
logout specs.

**Q: Why is PKCE needed if the SPA can't keep a secret anyway?**
PKCE isn't a client secret. It's a per-request, one-time secret that proves
the party redeeming the authorization code is the same party that started the
flow. It defeats code interception and injection. OAuth 2.1 requires it for
confidential clients too.

**Q: How do you revoke a JWT?**
You can't "un-sign" it. Combine short access-token lifetimes with
refresh-token revocation. For immediate effect, add a `jti`/`sub` denylist or
"revoked-before" timestamp checked after signature validation, or switch to
opaque tokens with introspection for that API.

**Q: RS256 or HS256 for microservices?**
RS256 (or ES256). With HS256 every verifier holds the signing secret and can
mint tokens. With asymmetric keys only the AS can sign, services verify with
the public JWKS, and keys rotate via `kid` without redeploying.

**Q: Where should an SPA store tokens?**
Preferably nowhere: use a BFF that keeps tokens server-side and gives the
browser an HttpOnly, Secure, SameSite cookie. If the SPA must hold tokens,
keep them in memory (or a web worker) with refresh-token rotation, a strict
CSP and short lifetimes. Avoid `localStorage` for sensitive apps.

**Q: A user's role was removed but they can still access admin endpoints. Why, and how do you fix it?**
Their access token still carries the old `groups` claim until it expires.
Fixes: shorter access-token lifetime, revoke the user's refresh tokens on role
change, and check high-risk permissions at runtime against the source of truth
or a policy engine rather than only from the token.

**Q: How does a downstream service know which user the request is for?**
Either the user's token is relayed (simple, but over-broad), or the calling
service performs a token exchange (RFC 8693) to get a token for the
downstream audience with the user as `sub` and itself as `act`. That token is
combined with mTLS/workload identity so the downstream service knows both
the user and the calling service.

**Q: 401 vs 403?**
401: the request isn't authenticated (missing, malformed, expired or invalid
token). The client should (re)authenticate. 403: the caller is authenticated
but not allowed. Re-authenticating won't help.

**Q: Why not just validate tokens at the API gateway?**
The gateway is a good place for coarse checks, but zero trust means each
service validates its own audience and permissions. Otherwise any workload
that can reach a service bypasses all authorization. Data-level authorization
can only happen inside the service anyway.

---

## References

| Spec | Topic |
|---|---|
| RFC 6749 / RFC 6750 | OAuth 2.0 framework / Bearer token usage |
| OAuth 2.1 (IETF draft) | Consolidated modern OAuth |
| RFC 9700 | OAuth 2.0 Security Best Current Practice |
| OpenID Connect Core 1.0 | ID token, claims, flows |
| OIDC RP-Initiated / Back-Channel Logout 1.0 | Logout |
| RFC 7519 / 7515 / 7516 / 7517 / 7518 | JWT / JWS / JWE / JWK / JWA |
| RFC 8725 | JWT Best Current Practices |
| RFC 9068 | JWT profile for OAuth 2.0 access tokens (`at+jwt`) |
| RFC 7636 | PKCE |
| RFC 7662 / RFC 7009 | Token introspection / revocation |
| RFC 8693 | Token exchange |
| RFC 8628 | Device authorization grant |
| RFC 8252 | OAuth for native apps |
| RFC 9126 / RFC 9396 | PAR / RAR |
| RFC 9449 / RFC 8705 | DPoP / mTLS-bound tokens |
| RFC 9207 | Authorization server issuer identification (mix-up defence) |
| IETF draft: OAuth 2.0 for Browser-Based Apps | SPA guidance, BFF pattern |
| OpenID FAPI 2.0 | High-security API profile |
| Spring Security reference — OAuth2 Resource Server / Client | Implementation |
| Okta developer docs — Angular SPA, refresh token rotation | Vendor implementation |
