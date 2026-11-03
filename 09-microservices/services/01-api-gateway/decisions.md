# Design Decisions — API Gateway

> Technical decisions specific to the gateway. The platform decision to have a gateway, and to
> build it with Spring Cloud Gateway, is `ADR-004-api-gateway.md`. Source: `docs/Architecture.md`
> of `en-monitor-api-gateway`. Dates are those of the commits that introduced each decision.

---

## [DEC-001] Spring Cloud Gateway on the servlet stack (Server Web MVC)

**Date:** 2026-10-21
**Status:** Active

**Context:** Spring Cloud Gateway offers a reactive (WebFlux) and a servlet (Web MVC) server.

**Decision:** Server Web MVC, like the services behind it.

**Consequences:**
- ✅ Same Spring Security configuration, error shape and way of testing as the services
- ⚠️ Spring Boot 4.0.8 instead of 4.1.1: Spring Cloud 2025.1 is the train that supports it

---

## [DEC-002] The token is validated twice

**Date:** 2026-10-22
**Status:** Active

**Context:** the token could be checked only at the gateway (services trust forwarded headers)
or also in each service.

**Decision:** the gateway rejects a missing, forged or expired token before it costs a service
anything, and forwards the token unchanged; each service validates it again with the public key
and resolves the permissions live. The gateway decides no permission and adds no user headers.

**Consequences:**
- ✅ A service reached without the gateway is not open
- ⚠️ Two validations per request (local signature checks, no network call)
- ⚠️ A new public endpoint in the security service must also be added to
  `SecurityFilterChainConfiguration.PUBLIC_ENDPOINTS` here

---

## [DEC-003] CORS only at the gateway

**Date:** 2026-10-22
**Status:** Active

**Decision:** the gateway answers CORS and drops `Origin` before forwarding.

**Consequences:**
- ✅ The browser never sees the CORS headers twice
- ✅ In the Docker stack the web app calls `/api/v1/...` on its own origin, so no CORS is needed

---

## [DEC-004] `/internal/**` is refused, not routed

**Date:** 2026-10-22
**Status:** Active

**Decision:** internal calls go between the services over the Docker network with their own
token (`X-Internal-Token`); the gateway answers 401/403 for `/internal/**` and never forwards it.

---

## [DEC-005] Rate limit by client address and path, in memory

**Date:** 2026-10-22
**Status:** Active

**Decision:** a fixed-window limiter of the public account `POST`s (20 per client address and
path per minute). The address comes from `X-Forwarded-For` only when the connection is from
loopback or the Docker network, so a client cannot choose the address that is counted.

**Consequences:**
- ✅ Enough for one instance, no extra infrastructure
- ⚠️ Per instance and reset on restart; several instances would need a shared store (Redis)
- ⚠️ Everyone behind one NAT shares the same counter

---

## [DEC-006] Routes by path prefix, most specific first

**Date:** 2026-10-23
**Status:** Active

**Decision:** the device and measurement routes are matched before the general home route, so
moving a module out of the monolith only added a route here. The last extraction (28-Oct-2026)
removed the monolith's catch-all: an unknown `/api/v1/**` path now answers 404.

**Consequences:**
- ✅ The paths the clients call never changed during the three extractions
- ⚠️ `/api/v1/homes/{homeId}` and its members and thresholds go to the home service, while
  `/api/v1/homes/{homeId}/devices/**` and `/consumption/**` go to other services

---

## [DEC-007] Apache HttpClient 5 forwards the requests

**Date:** 2026-10-21
**Status:** Active

**Context:** the JDK HTTP client fails on a `DELETE` without a body.

**Decision:** Apache HttpClient 5.

---

## [DEC-008] The web app's token comes from an HttpOnly cookie

**Date:** 2026-10-24
**Status:** Active

**Context:** the web app receives its tokens in HttpOnly cookies set by the security service
(`em_access`, `em_refresh`), so page scripts cannot read them and cannot add the
`Authorization` header themselves.

**Decision:** `AccessTokenCookieFilter` validates `em_access` and forwards it as
`Authorization: Bearer <cookie>`, only when the request has no `Authorization` header and brings
`X-Requested-With: XMLHttpRequest` (which a forged cross-site form or image cannot send). The
public account endpoints are left alone.

**Consequences:**
- ✅ The services keep validating a bearer token; nothing changed behind the gateway
- ✅ The mobile app keeps sending the header itself
