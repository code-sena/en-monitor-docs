# Design Decisions — Security Service

> Technical decisions specific to `en-monitor-security-service`. Platform decisions are the ADRs
> in `05-architecture/decisions/records/` (sessions and tokens: `ADR-010`, which supersedes `ADR-005`). Source: `docs/Architecture.md`
> and `README.md` of the repository. Dates are those of the commits that introduced each decision.

> **Update (28-Oct-2026):** these decisions describe the extraction from the modular monolith as it
> was done on 24-Oct-2026. The monolith's other modules have since become services of their own
> (`05-home-service` … `08-recommendation-service`) and `energy-monitor-backend` was retired, so every
> mention of "the backend" below means what the backend was at that time; its proxy packages no
> longer exist.

---

## [DEC-001] Moved out of the monolith unchanged in structure

**Date:** 2026-10-20
**Status:** Active

**Decision:** the `security` module of `energy-monitor-backend` was moved out with its package
(`com.energymonitor.security`), use cases, tests and changesets. The backend keeps a thin proxy
package that validates tokens with the public key and asks this service the rest.

**Consequences:**
- ✅ The other backend modules kept using the same `security::api`
- ⚠️ Permissions and profiles now cross the network (cached by the backend: 30 s / 60 s)

---

## [DEC-002] No roles in the access token

**Status:** Active

**Decision:** the access token carries `sub` and `iss`, no roles; permissions are read from the
database on every request.

**Consequences:**
- ✅ A granted or revoked role, or a blocked account, takes effect on the next request
- ⚠️ A database read per protected admin request

---

## [DEC-003] Tokens in HttpOnly cookies for the web app

**Date:** 2026-10-24
**Status:** Active

**Decision:** with `X-Session-Transport: cookie`, sign-in and refresh set `em_access` and
`em_refresh` as HttpOnly, `SameSite=Strict` cookies and return the tokens as `null` in the body;
the mobile app keeps receiving them in the body.

**Consequences:**
- ✅ No script on the page can read the session, so an XSS cannot steal it
- ⚠️ The gateway must turn the cookie into a bearer token (`AccessTokenCookieFilter`), and only
  with `X-Requested-With: XMLHttpRequest`, as CSRF protection

---

## [DEC-004] Six-digit recovery code with a pepper and a rate limit

**Status:** Active

**Decision:** recovery codes are stored as `HMAC-SHA256(code, pepper)` (pepper in configuration,
not in the table), redemption is limited to ten attempts per caller address per 15 minutes, and
issuing a code consumes the previous ones. BCrypt was rejected because a salted hash cannot be
looked up by the presented code.

**Consequences:**
- ✅ A leaked database dump yields no usable code
- ⚠️ The attempt counter is in memory: per instance and reset on restart
- ⚠️ Changing `SECURITY_RESET_PEPPER` invalidates every code already issued

---

## [DEC-005] Account deletion asks the backend first

**Status:** Active

**Decision:** before deleting, the service calls `GET /internal/accounts/{id}/deletion-check` on
the backend (1 s to connect, 2 s to read). 409 is returned to the user as is (for example, the
only owner of a home with other members); no answer refuses the deletion with 503. After the
deletion, `account.deleted` tells the backend to clean up.

---

## [DEC-006] Trust `X-Forwarded-For` only from loopback and the Docker network

**Status:** Active

**Decision:** `server.forward-headers-strategy: native` with
`server.tomcat.remoteip.internal-proxies` limited to loopback and `172.16.0.0/12`.

**Consequences:**
- ✅ Behind the gateway, the audit trail and the recovery limit see the real caller
- ✅ A machine of the LAN cannot pass as `127.0.0.1` to reach `/internal/**`
