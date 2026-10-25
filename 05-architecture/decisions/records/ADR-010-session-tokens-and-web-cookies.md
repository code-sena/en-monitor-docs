# ADR-010 — Session Tokens and the Web Session in HttpOnly Cookies

- **ID:** ADR-010
- **Date:** 2026-10-24
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra

---

## Context

`ADR-005-authentication-strategy.md` decided stateless RS256 access tokens of **1 hour**, a public
key **published** by the security service, and rotating refresh tokens. Three things changed
while the system was built and moved to `en-monitor-security-service`:

- The access token was implemented with a **15-minute** lifetime, the value
  `00-governance/security-policy.md` already stated, and without roles: permissions are read on
  every request so a revoked role takes effect at once.
- The public key is handed to the gateway and the services as a **file** (`public.pem`); no
  service exposes a JWKS endpoint.
- The web app kept both tokens where page scripts could read them. Any XSS in the page would have
  been enough to steal a session, a risk recorded in the project's quality documents.

These are changes to the decision itself, not to its implementation, so they are recorded here
and `ADR-005` is superseded.

**Known constraints:**
- The mobile app (Expo / React Native) has no browser cookie jar shared with the API and needs the
  tokens in the response body
- Every request goes through `en-monitor-api-gateway` (`ADR-004-api-gateway.md`), and each service
  validates a bearer token
- Delivery on 28-Oct-2026; the change must not alter the public API paths

---

## Decision

**We decided:** short-lived access tokens, rotating refresh tokens, a public key distributed as a
file, and, for the web app only, the session carried in HttpOnly cookies.

- **Access token:** RS256, **15 minutes**, signed only by `en-monitor-security-service` (the only
  holder of the private key). It carries `sub` and `iss` and no roles; permissions are resolved
  from the database on each request.
- **Refresh token:** opaque, stored as a hash, valid **7 days**, rotated on every use, with reuse
  detection by token family; logout closes only a session of the caller.
- **Public key:** distributed as `public.pem` to `en-monitor-api-gateway` and every service. The
  token is validated twice: by the gateway and again by the service that serves the route.
- **Web session:** when the request sends `X-Session-Transport: cookie`, sign-in, Google sign-in
  and refresh answer the tokens as cookies instead of in the body:
  - `em_access` (`Path=/api`) and `em_refresh` (`Path=/api/v1/auth`), both `HttpOnly`,
    `SameSite=Strict` and `Secure` in production;
  - the gateway turns `em_access` into `Authorization: Bearer …` only when the request also
    carries `X-Requested-With: XMLHttpRequest`, a header a forged cross-site form or image cannot
    send.
- **Mobile session:** unchanged, the tokens travel in the response body.

**Justification:** the 15-minute token limits how long a stolen access token is useful, and
reading permissions per request makes role changes immediate without short-circuiting the token
lifetime. Distributing the key as a file needs no extra endpoint or key-fetching logic for one
signer and a handful of services. Moving the web tokens into HttpOnly cookies removes them from
the reach of page scripts, so an XSS can no longer exfiltrate a session; `SameSite=Strict` plus
the required header cover the CSRF exposure that ambient cookies would otherwise add.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **15-minute token, file-distributed key, HttpOnly cookies for the web (chosen)** | Short exposure window; tokens out of reach of scripts; services keep validating a bearer token | Two transports to maintain (cookie and body); the gateway has to translate the cookie | — (chosen) |
| Keep `ADR-005` as written (1-hour token, tokens readable by the web page) | No change | A stolen token is valid for an hour; any XSS steals the session | The XSS exposure is the risk this decision removes |
| Server-side sessions (session id cookie, no JWT) | Immediate revocation | Every service would have to ask the security service per request; stateful | Breaks the local validation `ADR-005` chose for response time |
| JWKS endpoint for the public key | Standard key rotation | One more public endpoint and a fetch/caching path in every service | Not needed for one signer and file-based deployment |

---

## Consequences

**Positive:**
- An XSS in the web app cannot read the session tokens
- Role and status changes apply on the next request
- Services and the gateway validate tokens without calling the security service

**Negative / Trade-offs:**
- The security service supports two token transports, chosen by a header
- The web app must send `X-Requested-With: XMLHttpRequest` and `withCredentials` on every call
- Rotating the signing key means replacing `public.pem` everywhere and restarting the services

**Impact on the system:**
- Affected services: `en-monitor-security-service`, `en-monitor-api-gateway`, the web app
- Documents updated: `00-governance/security-policy.md`, `05-architecture/security-threat-model.md`,
  `09-microservices/services/02-security-service/`, `09-microservices/services/01-api-gateway/`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A cookie is sent over plain HTTP in production | Low | High | `Secure` is on unless explicitly disabled; it is disabled only in the local stack and development mode |
| The signing key leaks | Low | High | Only the security service mounts `private.pem`; the others receive the public key only |
| A cross-site request rides the cookies | Low | Medium | `SameSite=Strict` and the required `X-Requested-With` header |

---

## References

- Supersedes → `ADR-005-authentication-strategy.md`
- Gateway that validates and translates the cookie → `ADR-004-api-gateway.md`
- Security policy → `00-governance/security-policy.md`
- Code: README ("Sessions: tokens in the body or in HttpOnly cookies") and `docs/Architecture.md` of
  `en-monitor-security-service`; README of `en-monitor-api-gateway` (`AccessTokenCookieFilter`)
