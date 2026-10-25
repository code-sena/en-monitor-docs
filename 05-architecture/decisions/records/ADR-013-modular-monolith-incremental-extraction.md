# ADR-013 — Modular Monolith First, Incremental Extraction to Microservices

- **ID:** ADR-013
- **Date:** 2026-09-23
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra

---

## Context

The target design of this repository (`05-architecture/overview.md`, `ADR-003`, `ADR-004`)
described six independently deployed services from the start. With two developers, a fixed
delivery date and a domain still being discovered, building, deploying and debugging six
services before the first feature worked would have spent most of the schedule on distribution
instead of on the product.

The bounded contexts were already defined (`02-domain/domain-map.md`), so the question was how to
keep their boundaries real without paying for distribution on day one, and how to move to
services later without rewriting the clients.

**Known constraints:**
- Two-person team, delivery on 28-Oct-2026 (`01-context/scope.md`)
- One server for production (`05-architecture/deployment.md`)
- The public API paths used by the web and mobile apps must not change when a context moves

---

## Decision

**We decided:** build `energy-monitor-backend` as a **modular monolith** with Spring Modulith, one
module per bounded context, and extract the contexts to microservices **one at a time**.

- Each module exposes only its `api` package; `ModularityTests` fails the build when a module
  reaches into another one's internals.
- To extract a context: move its module, tests and Liquibase changelogs (same paths,
  `ADR-009`) to a new repository; replace the module in the backend with a **proxy package** that
  keeps the same `api` types, answered by an HTTP client of the new service's internal API
  (`ADR-012`) with a cache, and by consumers of its events (`ADR-008`); add the new service's
  routes to the gateway (`ADR-004`); remove the module from the backend.
- Order followed: Security (October 2026), Devices (October 2026), Monitoring and Measurements
  (24-Oct-2026). Home, Alerts, Recommendations and Notification remain in the backend; their
  migration is in progress, carried out by the teammate.

**Justification:** module boundaries checked at build time give most of the design benefit of
services, at the cost of one deployable. Because each context only depends on another one's
`api`, an extraction changes the implementation behind that `api`, not the modules that use it,
and the gateway keeps every public path unchanged.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Modular monolith, then extraction one context at a time (chosen)** | Fast start; boundaries enforced; each extraction is small and reversible | Two styles coexist during the migration; proxies to maintain | — (chosen) |
| Six microservices from the start | Independent deployment from day one | Distribution, data ownership and messaging before any feature works | Too costly for the team and schedule |
| Monolith without module boundaries | Simplest start | Contexts entangle; extraction becomes a rewrite | Would block the planned migration |
| Extract every context at once | One migration | High risk; long time with nothing deployable | Too risky before the deadline |

---

## Consequences

**Positive:**
- Each extraction reached production without changing the web or mobile apps
- The rest of the backend kept working unchanged during every extraction, behind the proxies

**Negative / Trade-offs:**
- Until the migration ends, the system mixes a modular monolith and microservices
- Proxy packages add caches and fallbacks the monolith did not need
- The shared database (`ADR-003`) remains until every context owns its own store

**Impact on the system:**
- Affected services: `energy-monitor-backend` and every extracted service
- Documents updated: `05-architecture/overview.md`, `02-domain/domain-map.md`,
  `09-microservices/service-catalog.md`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A module depends on another one's internals and blocks its extraction | Low | High | `ModularityTests` in the build |
| The remaining migration is not finished by the delivery date | Medium | Medium | Each extraction is independent; the backend keeps serving the contexts not yet moved |

---

## References

- Shared database → `ADR-003-shared-database.md`
- Gateway → `ADR-004-api-gateway.md`
- Events → `ADR-008-rabbitmq-transactional-outbox.md`
- Changelogs → `ADR-009-liquibase-changelogs-per-service.md`
- Internal API → `ADR-012-internal-service-api.md`
- Code: `docs/Architecture.md` of `energy-monitor-backend` ("Migration status")
