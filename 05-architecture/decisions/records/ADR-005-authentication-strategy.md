# ADR-005 — Authentication Strategy

- **ID:** ADR-005
- **Date:** 2026-09-10
- **Status:** Accepted
- **Authors:** Miguel Angel Garcia Artunduaga and Juan Esteban Cortes Parra

---

## Context

`01-context/overview.md` already commits to Spring Security + OAuth2 as the authentication
library choice, and `00-governance/security-policy.md` already specifies concrete JWT values
(signing algorithm, token lifetimes, refresh token handling) — but neither of these is written
down as a formal architectural decision with alternatives considered, and no ADR currently
covers this topic under any number. This ADR formalizes what is already decided in those two
documents, so the reasoning behind it is recorded once instead of assumed wherever it's used.

**Known constraints:**

- `security-service` is the identity authority for the whole system (`02-domain/domain-map.md`);
  every other service must be able to verify a caller's identity without calling
  `security-service` synchronously on every single request, or RNF1's (see
  `04-requirements/non-functional.md`) response-time budget gets consumed by an extra network
  hop on every protected endpoint
- Two client types (web, Android) must authenticate the same way against the same backend
- `00-governance/security-policy.md` already fixes several concrete values that this ADR must be
  consistent with, not re-litigate

---

## Decision

**We decided:** stateless JWT access tokens (RS256, 1-hour expiration) plus stateful, rotating
refresh tokens, exactly as already specified in `00-governance/security-policy.md`:

- Access tokens are signed with RS256; `security-service` holds the private signing key, and
  publishes the public key so every other service can verify a token's signature locally,
  without calling `security-service` per request.
- Access tokens expire after 1 hour and carry `sub` (user ID), `iat`, `exp`, and `jti` — never a
  password or full PII.
- Refresh tokens are stored server-side, hashed (never in plain text), expire after 7 days,
  rotate on every use (one refresh token = one use), and are invalidated on logout, on password
  change, and entirely (all active tokens for that user) if a revoked refresh token is ever
  reused — a sign of possible theft.

**Justification:** Local signature verification (RS256 + a published public key) means every
service can authenticate a request without a network call to `security-service`, which is what
makes RNF1's response-time target achievable across all six services, not just the ones that
happen to be fast. Short-lived access tokens limit how long a stolen token is useful; rotating,
revocable refresh tokens give the system a way to cut off a session immediately when it needs to
(logout, password change, suspected compromise) despite access tokens themselves being
stateless.

---

## Evaluated alternatives

| Alternative                                                            | Pros                                                                                                             | Cons                                                                                                                                                                                                                         | Reason for discarding                                                                                                                                                 |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RS256 JWT access token + stateful, rotating refresh token — chosen** | Local, no-network-call verification for access tokens (supports RNF1); refresh tokens are individually revocable | Requires distributing/rotating a public key across five other services when the key ever rotates                                                                                                                             | — (chosen; already the standing decision in `security-policy.md`)                                                                                                     |
| HS256 (symmetric secret shared by all services)                        | Simpler key management — one secret, no public/private key pair                                                  | Every service that can verify a token can also forge one, since verification and signing use the same secret; a leak in any one of six services compromises the whole system                                                 | Discarded — `security-policy.md` allows HS256 only as a fallback with a 256+ bit secret, and RS256's asymmetric split is the safer default for a multi-service system |
| Fully stateless refresh tokens (long-lived JWT, no DB lookup)          | No database dependency for refresh; simplest to implement                                                        | Cannot be revoked before expiration — a stolen refresh token remains valid for its full lifetime even after logout or a password change                                                                                      | Discarded — directly conflicts with `security-policy.md`'s explicit "invalidated on logout and on password change" requirement                                        |
| Server-side sessions instead of JWT (cookie + session store)           | Instantly revocable, simple mental model                                                                         | Requires a shared session store reachable by all six services on every request, reintroducing the network-call-per-request cost this decision exists to avoid; awkward fit for a native Android client compared to a browser | Discarded — works against RNF1 and doesn't fit the mobile client as naturally as a bearer token                                                                       |

---

## Consequences

**Positive:**

- Every service can verify a caller's identity independently, supporting RNF1 without adding a
  synchronous call to `security-service` on every protected endpoint
- A compromised or leaked access token is only useful for up to 1 hour
- Logout, password change, and detected token theft can each immediately invalidate a user's
  active sessions, despite access tokens being stateless

**Negative / Trade-offs:**

- Rotating the RS256 signing key requires distributing the new public key to five other services
  before the old key is retired — an operational step that has to be planned, not assumed away
- Refresh token rotation requires a database lookup on every refresh — an accepted cost, since
  refreshing happens far less often than authenticated requests do

**Impact on the system:**

- Affected services: `security-service` (issues and verifies), all five other services (verify
  only, using the published public key)
- Documents that must be updated: `09-microservices/services/02-auth-service/decisions.md` and
  `README.md` currently describe a **different, generic, unfilled example** (PostgreSQL + Redis,
  an unrelated "appointment" permission example, placeholder `[fecha]` dates) that predates this
  project's actual stack decision (MySQL, per `01-context/overview.md`) — those two files need
  their own correction pass to match this ADR and the rest of the project, not just a citation
  update

---

## Risks

| Risk                                                                                            | Probability                                                | Impact                                               | Mitigation                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The RS256 private key leaks from `security-service`                                             | Low, if handled per `cross-cutting.md`/secrets conventions | High — would allow forging valid tokens for any user | Private key supplied only via environment variable/secret store, never logged or committed; rotate immediately if suspected compromised                           |
| A revoked refresh token's detection logic has a bug, letting a stolen token keep working        | Low                                                        | High                                                 | Cover this path explicitly in `security-service`'s test suite; this is exactly the kind of rule that should have a dedicated test, not just be implied by the ADR |
| Five services fail to pick up a rotated public key in time, causing valid tokens to be rejected | Medium during a key rotation event                         | Medium (availability impact, not a security one)     | JWKS-style publication of the current key(s), allowing overlap between the old and new key during rotation                                                        |

---

## References

- `00-governance/security-policy.md` — the concrete values this ADR formalizes (already decided,
  not proposed here for the first time)
- `01-context/overview.md` — the Spring Security + OAuth2 technology choice this ADR is
  consistent with
- `02-domain/domain-map.md` — `Access Token`, `Refresh Token`, `Password Reset Token` in the
  Security bounded context's ubiquitous language
- Related to: `ADR-003-shared-database.md` (refresh tokens are persisted in the shared database,
  in `security-service`'s own tables)
- Note: `09-microservices/services/01-api-gateway/README.md` cites its own "ADR-003 —
  Authentication strategy at the gateway vs. in each service", a different, narrower question
  (where authorization checks are performed) than this ADR (how tokens are issued and verified).
  If that gateway-placement question still needs its own decision, it should get a number that
  doesn't collide with the shared-database ADR-003 — see the note accompanying this batch.
