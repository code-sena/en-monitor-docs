# ADR-012 — Internal REST API Between Services

- **ID:** ADR-012
- **Date:** 2026-10-20
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra

---

## Context

Once a context runs as its own service, some questions need an immediate answer from another
context and cannot wait for an event: before deleting an account the security service must know
whether the person is the only owner of a home with other members; before listing a home's
devices or its consumption, the device and measurement services must know whether the caller
belongs to that home; the backend's alerts need the consumption levels and the hourly energy.
Events (`ADR-008-rabbitmq-transactional-outbox.md`) announce what happened; they do not answer
questions.

These calls are not part of the public API and must never be reachable from the internet.

**Known constraints:**
- All services run in the same Docker network, behind `en-monitor-api-gateway` and the web
  container's nginx
- The callers are services, not users: there is no user token to forward in every case
- No budget or time for a service mesh or a certificate authority for mutual TLS

---

## Decision

**We decided:** synchronous calls between services go to a separate path prefix, `/internal/**`,
protected by network and by a shared secret.

- Each service gives `/internal/**` its own security filter chain: the caller must come from an
  allowed network (loopback and the Docker range `172.16.0.0/12`) **and** send
  `X-Internal-Token` equal to `INTERNAL_API_TOKEN`, compared in constant time. With no token
  configured every internal call is refused.
- A user's access token never opens an internal path, and the gateway and nginx never forward
  `/internal/**`.
- Callers use short timeouts, cache what is safe to cache (device and level lookups, permissions,
  profiles) and answer 503 when the owner does not respond and nothing was cached. Membership and
  home access are never cached.
- The current internal endpoints are listed in `07-api/README.md` and in each service folder of
  `09-microservices/services/`.

**Justification:** a separate prefix with its own chain cannot be confused with the public API,
and requiring both the network and the token means neither a container on the network nor a leaked
token alone is enough. It reuses Spring Security, which every service already has, and costs one
shared secret in the environment of each service.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **`/internal/**` with allowed networks and a shared token (chosen)** | Simple; testable per service; never exposed by the gateway | One secret shared by all services; rotating it means updating every service | — (chosen) |
| Mutual TLS between services | Strong service identity | A certificate authority, certificates and rotation for every service | Too much operational work for the team and deadline |
| Service tokens issued by the security service (client credentials) | Per-service identity | A new grant type and token handling in every caller | More moving parts than the threat requires |
| Read the other context's tables directly | No HTTP hop | Breaks table ownership (`ADR-003`) and couples schemas | Against ownership |
| Events only | No synchronous coupling | Cannot answer a question at request time | Some decisions need an answer before replying to the user |

---

## Consequences

**Positive:**
- The public API surface does not grow with internal needs
- Each internal endpoint is covered by tests for the token, the network and forwarded addresses

**Negative / Trade-offs:**
- Synchronous coupling: a service that does not answer turns some requests into 503
- `INTERNAL_API_TOKEN` must be identical in every service

**Impact on the system:**
- Affected services: all of them
- Documents updated: `07-api/README.md`, `05-architecture/overview.md`, `09-microservices/`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| The shared token leaks | Low | High | It never leaves the server environment; the network check still applies; rotate it in every `.env` |
| A proxy forwards `/internal/**` by mistake | Low | High | The gateway refuses the prefix and nginx only forwards `/api/`; services trust `X-Forwarded-For` only from the Docker network |
| An owner service is down | Medium | Medium | Short timeouts, caches with stale fallback where safe, 503 instead of a guess |

---

## References

- Events between services → `ADR-008-rabbitmq-transactional-outbox.md`
- Broker authentication endpoints, same network rule → `ADR-011-mqtt-authentication-by-device-service.md`
- Code: `InternalApiConfiguration` / `InternalCallerCheck` of each service; `docs/Architecture.md`
  of `en-monitor-security-service` ("Internal API")
