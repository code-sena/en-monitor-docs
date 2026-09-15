# Cross-Cutting Concerns

> This document covers what applies to every microservice, not to one bounded context. Where a
> decision already exists elsewhere in this repository, it is referenced, not repeated. Where it
> does not exist yet, that is stated plainly rather than invented here.

---

## Standard logging

**Not yet decided as a project-wide standard** — no logging library, format, or aggregation tool
is fixed in any document reviewed. Given the stack is Java 21 + Spring Boot across all six
services (`01-context/overview.md`), the natural default is SLF4J + Logback with structured
(JSON) output, but this has not been confirmed as a decision.

What **is** already established, and should anchor whatever logging standard gets chosen:

- `AuditLog` (Security) already captures **security-relevant** actions — who did what, when, from
  where (`01-context/scope.md`, feature #4). This is a business record with its own retention and
  query needs, not a substitute for application logs, and should not be conflated with them.
- Every domain event envelope already carries `metadata.correlationId` and `metadata.causationId`
  (`02-domain/domain-events.md`) — any logging standard adopted should log these same IDs on every
  line related to processing that event, so a log line can be tied back to the distributed trace
  it belongs to (see "Distributed tracing" below).

**Open question:** log format (plain text vs. structured JSON), minimum log level per environment,
and whether logs are aggregated anywhere (a local file, a self-hosted stack like Grafana Loki, or
nothing beyond `docker logs`) — relevant given `01-context/scope.md`'s constraint that no budget
is allocated for paid hosting or third-party services, which rules out most managed logging SaaS.

---

## Distributed tracing

**Partially established.** There is no tracing tool (OpenTelemetry, Zipkin, Jaeger) adopted yet,
but the correlation mechanism a tracing tool would rely on already exists in the domain event
contract:

- `metadata.correlationId` — ties every event in one business transaction together across
  services, even when they communicate asynchronously over Kafka.
- `metadata.causationId` — records which event or command caused this one, letting a chain of
  causally-related events be reconstructed (`02-domain/domain-events.md`, "Standard fields for
  all events").

What is missing is the **synchronous** half: none of the REST calls documented in
`05-architecture/overview.md` (Security's access-token claims, the `homeId`/`deviceId`
cross-context lookups) are documented as propagating a correlation ID today. Until that is
decided, a request that fans out over REST and then triggers a Kafka event cannot be traced
end-to-end — only the Kafka portion can.

**Open question:** whether to adopt a tracing standard (OpenTelemetry is the least vendor-specific
option) that propagates the same `correlationId` already used in domain events across REST calls
too, so one ID covers a request from the moment it enters the system to every event it triggers.

---

## Centralized configuration

**Not yet decided.** No configuration server (Spring Cloud Config, Consul, etc.) or convention for
per-service configuration is documented anywhere in this repository.

Two relevant constraints from `01-context/scope.md` narrow the realistic options:

- No budget for paid hosting or cloud infrastructure — rules out managed configuration services.
- No Kubernetes in the stack (`05-architecture/deployment.md`) — rules out ConfigMaps/Secrets as
  the mechanism, since there is no cluster to hold them.

Realistic options for this project's scale (six services, two developers, Docker only) are
per-service environment variables passed through Docker Compose, or a lightweight self-hosted
config server — neither has been chosen.

**Open question:** where does each service's configuration live (per-service `application.yml` +
environment variables, vs. a shared config service), and how are secrets (database credentials,
the `PasswordHasher` algorithm's parameters, device `api_key` validation secrets) kept out of any
repository.

---

## Feature flags

**Not yet decided.** No feature-flag mechanism (a library, a database-backed toggle table, or a
config-based approach) appears in any document reviewed, and no current feature in
`01-context/scope.md` is described as needing to be toggled independently of a deployment.

**Open question:** whether this project needs feature flags at all, given its scale and two-person
team, or whether a deploy-to-release-it approach is sufficient for the MVP scope. If a concrete
need shows up (for example, rolling out push notifications, `01-context/scope.md` feature #15,
gradually), it should be decided then rather than built speculatively now.

---

## Error handling

**Not yet decided as a documented, project-wide convention**, though the architecture already
implies where it must live: per Hexagonal Architecture (`01-context/glossary.md`), the domain
layer does not know about HTTP, so a domain or use-case exception cannot itself carry an HTTP
status code — that mapping has to happen in the inbound (web) adapter, service by service, but
the mapping table itself (which domain exception → which HTTP status → which error shape in the
response body) is not written down anywhere yet.

What is already established for one specific error path:

- `LoginErrorLog` records failed-login error types (`INVALID_PASSWORD`, `USER_NOT_FOUND`,
  `ACCOUNT_BLOCKED`, `ACCOUNT_INACTIVE`) as first-class, queryable data, not just a log line
  (`06-data/models.md`, Security). This is closer to a domain error model than to generic
  exception handling, and is a useful reference for the shape other services' error handling
  could follow: specific, named error types rather than a single generic "bad request."

**Open question:** a shared error response shape across all six services (for example, a
`ProblemDetails`/RFC 7807-style body), and whether unhandled exceptions get a single global
handler pattern per service or are left to Spring Boot's defaults.

---

## Retry policies

**Established for async consumers, not yet decided for synchronous calls.**

`02-domain/domain-events.md` already fixes the async retry policy for every Kafka consumer in
this project:

| Setting            | Value                                 |
| ------------------ | ------------------------------------- |
| Retries before DLQ | 3–5                                   |
| Backoff            | Exponential (1s → 2s → 4s → 8s)       |
| DLQ retention      | 7 days                                |
| Alert              | When the DLQ has more than 0 messages |

This applies uniformly to `alert-service` and `recommendation-service` consuming
`MeasurementRecorded`, and to `alert-service` consuming `DeviceConnectivityLost` — see
`05-architecture/pattern-guide.md` ("Dead Letter Queue") for when this pattern applies and why.

**Not covered by the above, and not yet decided:** retry behavior for the **synchronous** REST
calls in `05-architecture/overview.md` — for example, what `alert-service` does if its REST call
to `home-service` to resolve a `homeId` times out or returns a 5xx. Options range from no retry
(fail the request immediately) to a bounded retry with backoff, possibly with a circuit breaker
if a downstream service is degraded. None of these has been chosen.

---

## Regulatory compliance — Habeas Data (Law 1581 of 2012)

`04-requirements/non-functional.md` (ERNF3.1) commits this project to complying with Colombia's
Law 1581 of 2012 (Habeas Data). That law sets eight principles for handling personal data
(Article 4); the table below checks this project's actual data model
(`06-data/models.md`) and mechanisms against each one — not a generic compliance checklist, this
project's own gaps.

Personal data in scope: `Person` (name, last_name, cellphone, address, profile_image), `User`
(email), and the incidental personal data every table's audit trail carries (`ip_address`,
`user_agent` in `user_session`; `ip_address` in `login_error_log` and `audit_log`).

| Principle (Art. 4)               | What it requires                                                                                       | Status in this project                                                                                                                                                                                                                                                                                                                           |
| -------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Legalidad                        | Treatment must follow the law and its implementing rules                                               | No project-specific gap beyond the others below — this principle is satisfied by satisfying the rest                                                                                                                                                                                                                                             |
| Finalidad                        | Each use of personal data must serve a legitimate purpose, stated to the data subject                  | **Not documented.** No privacy notice or stated-purpose text is referenced anywhere in this repository — `HU1`'s "accept the terms and license" scenario (`04-requirements/user-stories.md`) implies one exists as a product artifact, but it is not linked from here                                                                            |
| Libertad                         | Treatment requires prior, express, informed consent — data cannot be collected or disclosed without it | **Gap.** `HU1`'s acceptance criteria require accepting "terms, conditions, and license permissions" at registration, but `06-data/models.md`'s `user` table has **no field recording that consent was given** — no `terms_accepted_at`, no `consent_version`, nothing. The product requires consent; the schema does not preserve evidence of it |
| Veracidad o calidad              | Data must be truthful, complete, accurate, and kept up to date                                         | Supported structurally: `UpdateUserProfileUseCase` exists (`05-architecture/hexagonal-architecture.md`) so a data subject can correct their own data, satisfying this principle's practical requirement                                                                                                                                          |
| Transparencia                    | The data subject can obtain, at any time, information about what data concerning them exists           | **Not implemented.** No use case or endpoint exposes "what data do you hold about me" as a single, dedicated capability — a user can see individual screens of their own data, but there is no consolidated access mechanism                                                                                                                     |
| Acceso y circulación restringida | Treatment is limited to those authorized by the data subject or by law                                 | Backed by the RBAC mechanism already documented in `05-architecture/pattern-guide.md` and the home-scoped access model (`UserHome`) — access is already structurally restricted, not open by default                                                                                                                                             |
| Seguridad                        | Technical, human, and administrative measures to prevent alteration, loss, unauthorized access         | **Same gaps already tracked in `05-architecture/security-threat-model.md`** (T3/T4: no TLS decision; I4: no documented home-isolation enforcement) — this principle does not need a new gap, it needs those existing ones resolved                                                                                                               |
| Confidencialidad                 | Everyone who handles the data must keep it reserved, even after their involvement ends                 | **Not documented.** No confidentiality/NDA expectation for team members handling production data is written down anywhere — relevant given this is a two-person team (`01-context/scope.md`) with direct database access, not an abstract policy question                                                                                        |

**What this adds to the open questions below:** a consent-evidence field on `user` (Libertad), a
consolidated "my data" capability (Transparencia), and a documented data-handling confidentiality
expectation for the team (Confidencialidad) are three concrete, new gaps — not restatements of
the TLS/encryption gaps already tracked elsewhere.

---

## Open questions — summary

| Concern                   | Status                                                                                                                                        |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Standard logging          | Not decided (library, format, aggregation)                                                                                                    |
| Distributed tracing       | Correlation IDs exist for Kafka events; not propagated over REST; no tracing tool chosen                                                      |
| Centralized configuration | Not decided (where config and secrets live)                                                                                                   |
| Feature flags             | Not decided whether even needed at this scale                                                                                                 |
| Error handling            | No shared response shape or exception-mapping convention documented                                                                           |
| Retry policies            | Fixed for Kafka consumers; not decided for synchronous REST calls                                                                             |
| Habeas Data (Law 1581)    | Partially met structurally (RBAC, correction); consent evidence, "my data" access, and team confidentiality expectations are not yet in place |

None of these block the data model or domain work already done in `02-domain/` and `06-data/`,
but each one will need an owner and a decision before `09-microservices/` implementation details
can be written consistently across all six services.

---

## Correlations

- Domain event envelope (correlationId/causationId, retry/DLQ settings referenced above) →
  `02-domain/domain-events.md`
- Idempotent Consumer and Dead Letter Queue patterns → `05-architecture/pattern-guide.md`
- Communication patterns these concerns apply to → `05-architecture/overview.md`
- Deployment constraints (no paid hosting, no Kubernetes) that narrow the realistic options above
  → `05-architecture/deployment.md`, `01-context/scope.md`
- Security-relevant audit trail (distinct from application logging) → `06-data/models.md` (`audit_log`, `login_error_log`)
- The requirement this section's new compliance table traces back to → `04-requirements/non-functional.md` (ERNF3.1)
- Personal data fields referenced above → `06-data/models.md` (`person`, `user`, `user_session`)
- Security gaps this Habeas Data table reuses rather than duplicates → `05-architecture/security-threat-model.md` (T3, T4, I4)
