# Cross-Cutting Concerns

> This document covers what applies to every microservice, not to one bounded context. Where a
> decision already exists elsewhere in this repository, it is referenced, not repeated. Where it
> does not exist yet, that is stated plainly rather than invented here.

---

## Standard logging

**Not decided as a project-wide standard.** No service has its own Logback configuration or a
log-shipping setup: all eight Java components (gateway and the seven services)
log with the Spring Boot defaults, and logs are read with `docker compose logs <service>`
(`10-devops/local-setup.md`). There is no log aggregation tool.

What **is** already established, and should anchor whatever logging standard gets chosen:

- `AuditLog` (Security) captures **security-relevant** actions — who did what, when, from where
  (`01-context/scope.md`, feature #4). It is a business record with its own retention and query
  needs, not a substitute for application logs.
- Behind the gateway, every service records the caller's real address: `X-Forwarded-For` is
  trusted only when the connection comes from loopback or the Docker network (`172.16.0.0/12`),
  so the address in the audit trail and in the rate limits cannot be chosen by the client.
- Every RabbitMQ message carries its id (`id_message` of the outbox row), which is what a log
  line about consuming it should quote.

**Open question:** log format (plain text vs. structured JSON), minimum log level per environment,
and whether logs are aggregated anywhere beyond `docker logs` — relevant given
`01-context/scope.md`'s constraint that no budget is allocated for paid third-party services.

---

## Distributed tracing

**Not in place.** No tracing library (OpenTelemetry, Micrometer Tracing, Zipkin) is a dependency
of any service, and neither the internal REST calls (`/internal/**`) nor the RabbitMQ messages
propagate a correlation id today. The `metadata.correlationId` / `causationId` fields described in
`02-domain/domain-events.md` belong to the target event envelope; the RabbitMQ payloads in
production carry only the message id and the business fields (`09-microservices/services/*/events.md`).

**Open question:** whether to adopt a tracing standard (OpenTelemetry is the least
vendor-specific option) that propagates one id from the gateway through the internal REST calls
and into the outbox messages.

---

## Centralized configuration

**Decided by practice, not by ADR:** there is no configuration server. Each service has its own
`application.yaml` in its repository, and the environment-specific values arrive as environment
variables:

- **Full local stack and production:** Docker Compose reads `infra/.env` (local) or
  `infra/.env.prod` (server) and passes each container its variables.
- **Development mode:** each service on the PC reads the `.env` of its own folder plus an
  `application-local.yaml` copied from its `.example`, with the shared values taken from
  `infra/.env`.
- **Values shared by several services:** `INTERNAL_API_TOKEN` and `RABBITMQ_*` must be the same in
  every service; `MYSQL_PASSWORD` is the same for all four components that use the database.
- **Secrets:** `.env`, `.env.prod`, the RSA key pair (`keys/`) and the broker certificates
  (`mosquitto/certs/`) are git-ignored and never versioned. Only the security service receives
  `private.pem`; every other service receives `public.pem` only. GitHub Actions keeps only the
  deployment secrets (`DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS`).

The full variable list is in `10-devops/local-setup.md`.

---

## Feature flags

**Not decided.** No feature-flag mechanism exists. A few behaviours are switched by configuration
rather than by a flag system — for example `MEASUREMENT_MQTT_ENABLED` (only one measurement
subscriber may store readings; off by default, on in the infra stack), and optional integrations
that turn themselves off when their variables are empty (Google sign-in, VAPID web push, Expo
push, e-mail).

---

## Error handling

**One shared error shape.** Every service renders its errors with one exception handler, and the
gateway uses the same JSON shape for the answers it produces itself:

```json
{ "timestamp": "...", "status": 503, "error": "...", "message": "...", "path": "/api/v1/..." }
```

The mapping from domain or use-case exception to HTTP status lives in each service's inbound web
adapter, as Hexagonal Architecture requires (`01-context/glossary.md`). Answers produced by the
gateway: 401 (missing or invalid token), 404 (no route), 429 (rate limit, with `Retry-After`),
503 (the service refused the connection) and 504 (no answer within 30 s).

`LoginErrorLog` additionally records failed-login error types (`INVALID_PASSWORD`,
`USER_NOT_FOUND`, `ACCOUNT_BLOCKED`, `ACCOUNT_INACTIVE`, `EMAIL_NOT_VERIFIED`) as queryable data
(`06-data/models.md`, Security).

---

## Retry and degradation policies

**Async consumers (RabbitMQ, `ADR-008`):**

| Setting | Value |
|---|---|
| Publishing | Outbox relay every second, oldest first; a row is published only after the broker confirms it and a queue took it; the first failure stops the run and the row is retried on the next one |
| Delivery | At least once, with the message id; consumers tolerate duplicates |
| Consumer attempts before the dead-letter queue | 3 (1, 2 and 4 s apart in the device and measurement services) |
| Queues | Durable, each with its `.dlq` dead-letter queue |

**Synchronous internal calls:** no generic retry; each caller combines short timeouts, a cache
and a defined answer when the other side does not respond:

| Caller → callee | Cache | When the callee does not answer |
|---|---|---|
| notification → security (permissions, administrators / profiles), home → security (profiles) | 30 s / 60 s | Names left out, alert mail skipped (push still goes), administrator notice routes answer 503 |
| security → home (deletion check) | — | 1 s to connect, 2 s to read; the deletion is refused with 503 |
| measurement, alert, recommendation, notification → device | 30 s, unknown devices included | Stale copy served up to 1 h; with nothing kept, 503 (a reading is dropped) |
| alert, recommendation → measurement (consumption levels / energy) | Levels cached 10 min, stale copy up to 24 h; energy never cached | Service-specific "unavailable" exception |
| device, measurement, alert, recommendation → home (home access) | Never cached | 503 |
| notification → home (name and members) | — | Not documented in this repository yet (see the service's README) |
| gateway → any service | — | 2 s to connect, 30 s to read; 503 or 504 |
| Mosquitto → device (MQTT auth/ACL) | 5 min per decision | New MQTT logins refused while the device service is down |

There is no circuit breaker library in use.

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
| Seguridad                        | Technical, human, and administrative measures to prevent alteration, loss, unauthorized access         | **Partially met.** HTTPS and MQTT over TLS are in production (`05-architecture/security-threat-model.md`, T3/T4); home isolation inside `energy-monitor-backend` is still not documented (I4) |
| Confidencialidad                 | Everyone who handles the data must keep it reserved, even after their involvement ends                 | **Not documented.** No confidentiality/NDA expectation for team members handling production data is written down anywhere — relevant given this is a two-person team (`01-context/scope.md`) with direct database access, not an abstract policy question                                                                                        |

**What this adds to the open questions below:** a consent-evidence field on `user` (Libertad), a
consolidated "my data" capability (Transparencia), and a documented data-handling confidentiality
expectation for the team (Confidencialidad) are three concrete, new gaps — not restatements of
the TLS/encryption gaps already tracked elsewhere.

---

## Open questions — summary

| Concern | Status |
|---|---|
| Standard logging | Spring Boot defaults, read with `docker compose logs`; no standard or aggregation decided |
| Distributed tracing | Not in place; no correlation id across REST calls or messages |
| Centralized configuration | Per-service `application.yaml` + environment variables from Compose; secrets git-ignored |
| Feature flags | None; a few behaviours switched by configuration |
| Error handling | One shared error shape in every service and the gateway |
| Retry policies | Fixed for RabbitMQ (outbox relay, three attempts, dead-letter queue); synchronous calls use timeouts, caches and defined fallbacks, no generic retry |
| Habeas Data (Law 1581) | Partially met structurally (RBAC, correction, TLS); consent evidence, "my data" access, and team confidentiality expectations are not yet in place |

---

## Correlations

- Event transport, outbox and dead-letter queues → `ADR-008-rabbitmq-transactional-outbox.md`
- Target domain event envelope (correlationId/causationId) → `02-domain/domain-events.md`
- Idempotent Consumer and Dead Letter Queue patterns → `05-architecture/pattern-guide.md`
- Communication patterns these concerns apply to → `05-architecture/overview.md`
- Deployment (Docker Compose, no Kubernetes) → `05-architecture/deployment.md`, `01-context/scope.md`
- Environment variables → `10-devops/local-setup.md`
- Security-relevant audit trail (distinct from application logging) → `06-data/models.md` (`audit_log`, `login_error_log`)
- The requirement this section's new compliance table traces back to → `04-requirements/non-functional.md` (ERNF3.1)
- Personal data fields referenced above → `06-data/models.md` (`person`, `user`, `user_session`)
- Security gaps this Habeas Data table reuses rather than duplicates → `05-architecture/security-threat-model.md` (T3, T4, I4)
