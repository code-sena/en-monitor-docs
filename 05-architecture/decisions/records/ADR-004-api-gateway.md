# ADR-004 — API Gateway Selection

- **ID:** ADR-004
- **Date:** 2026-09-11
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra and Miguel Angel Garcia.

---

## Context

The system is built as six independently deployed services (`security-service`, `home-service`,
`device-service`, `measurement-service`, `alert-service`, `recommendation-service`, per
`02-domain/domain-map.md`), consumed by two clients — a web application and an Android
application (`01-context/overview.md`). Without a single entry point, each client would need to
know the address of every one of the six services individually, and every service would need to
verify a caller's identity on its own — duplicating the token-verification logic
`ADR-005-authentication-strategy.md` establishes, six times over, instead of once.

This ADR decides whether a gateway is used at all, and if so, which product — both questions are
resolved here, not assumed from elsewhere.

**Known constraints:**
- Two-person team, fixed delivery deadline of the last week of October 2026
  (`01-context/scope.md`)
- No budget for paid cloud infrastructure or third-party services (`01-context/scope.md`)
- Every one of the six backend services is Java 21 + Spring Boot (`01-context/overview.md`) —
  no other language or runtime is otherwise part of this stack
- The gateway needs to route to all six services and is the natural place to validate an access
  token once, per `ADR-005-authentication-strategy.md`'s RS256 signature verification, rather than
  duplicating that check ad hoc

---

## Decision

Use **Spring Cloud Gateway** as the single entry point for both clients.

**We decided:** the web application and the Android application call Spring Cloud Gateway, which
routes each request to the appropriate one of the six backend services.

**Justification:** Spring Cloud Gateway is Java and part of the Spring ecosystem the rest of the
stack already uses — it introduces no new language, runtime, or operational skill the team
doesn't already need for the six services themselves. It runs as an ordinary Spring Boot
application, deployable the same way as every other service, and can verify an access token's
RS256 signature at the gateway layer using the same public key mechanism `ADR-005` already
establishes, instead of every service needing to duplicate that logic independently.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|-------------|------|------|------------------------|
| **Spring Cloud Gateway (CHOSEN)** | Same stack as every backend service (Java/Spring) — no new language or runtime; deploys like any other Spring Boot service; integrates naturally with Spring Security for token verification | Fewer built-in enterprise features (a management UI, a plugin marketplace) than a dedicated gateway product like Kong | — (chosen) |
| Kong | Mature, widely used, large plugin ecosystem for rate limiting, auth, logging, etc. | Built on Lua/OpenResty — a language and runtime the team does not otherwise use anywhere in this stack; several deployment modes require their own configuration database | Introduces an entirely new technology stack for a two-person team already adopting Kafka (`ADR-002`) and Mosquitto (`ADR-007`) this same project cycle |
| Plain NGINX reverse proxy | Extremely lightweight, extremely well known, minimal resource footprint | Static, configuration-file-based routing only — no programmatic request handling, no natural place to plug in the token-verification logic `ADR-005` needs at this layer | Lacks the application-level capability (auth filters, request transformation) this project needs from its gateway, not just routing |
| No gateway — clients call the six services directly | Nothing extra to build or deploy | Each client would need to know all six service addresses individually; token verification (`ADR-005`) would need to be duplicated in every one of the six services instead of living in one place | The duplicated auth-verification cost alone, across six services, outweighs the cost of running one more Spring Boot application |

---

## Consequences

**Positive:**
- No new language, runtime, or operational skill enters the stack — the gateway is just another
  Spring Boot service to the team
- Token verification (`ADR-005`) can live once at the gateway instead of being duplicated six
  times
- Deploys and scales the same way the rest of the services already do (Docker, per
  `01-context/overview.md`)

**Negative / Trade-offs:**
- Spring Cloud Gateway has a narrower feature set than a dedicated gateway product — advanced
  capabilities (a management dashboard, a wide plugin marketplace) aren't available out of the
  box and would need to be built if ever needed
- The gateway becomes a single point of failure for every client request — if it is down, both
  clients lose access to all six services at once, not just one

**Impact on the system:**
- Affected: both clients (web, Android), all six backend services (routed to).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| The gateway becomes a single point of failure for the whole system | Medium, with a single instance | High — both clients lose access to everything at once | Acceptable at this project's current scale and team size; revisit (e.g. running more than one instance behind a load balancer) only if this becomes a real operational problem, not preemptively |
| Token verification logic at the gateway drifts out of sync with `ADR-005`'s actual signing key rotation process | Low | Medium | Gateway and `security-service` share the same published-public-key mechanism described in `ADR-005`, not a separately maintained copy of the verification logic |

---

## References

- Token verification this gateway performs → `ADR-005-authentication-strategy.md`
- Services this gateway routes to → `02-domain/domain-map.md`
- Related to: `ADR-002-domain-event-transport.md` and `ADR-007-mqtt-broker-selection.md` (other
  infrastructure decisions made in this same project phase, each for a different communication
  concern)