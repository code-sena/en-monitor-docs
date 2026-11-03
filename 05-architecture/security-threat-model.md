# Security Threat Model

> Methodology: **STRIDE** (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of
> Service, Elevation of Privilege), applied to Energy Monitor's actual actors and assets: the
> three user-facing identities (`Person`/`User`, home membership, measuring device), the shared
> MySQL database, the MQTT and RabbitMQ channels, the API gateway, and the services behind it
> (the seven microservices: security, device, measurement, home, notification, alert and
> recommendation). Rows were reviewed against the system in production on 28-Oct-2026; the
> monolith `energy-monitor-backend` they used to name was retired that day.
>
> Each threat below states the concrete mitigation already implemented, sourced from
> `02-domain/entities-and-rules.md`, `01-context/glossary.md`, or `06-data/models.md` — not a
> generic textbook mitigation. Where nothing is implemented yet, that is stated as a gap, not
> glossed over. Threats without a citation are not yet mitigated.

---

## Spoofing (identity)

_Can an attacker convincingly pretend to be a user, a device, or a service?_

| #   | Threat                                                                                 | Implemented mitigation                                                                                                                                                                                                                                                                                                                               |
| --- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| S1  | Attacker authenticates as another user with a stolen or guessed password               | Passwords are never stored or compared in plain text — `User` is constructed only from an already-hashed `password_hash` (INV-004), produced by the `PasswordHasher` outbound port (`06-data/data-dictionary.md`), so the concrete hashing algorithm is swappable without touching a use case                                                        |
| S2 | Attacker reuses a stolen access token indefinitely | `Access Token` is short-lived (15 minutes, RS256) and stateless (`01-context/glossary.md`, `ADR-010`) — its blast radius is bounded by its own expiration. It carries no roles: the security service reads permissions from the database on every request, so a revoked role or a blocked account takes effect on the next request |
| S3 | Attacker steals a refresh token and keeps renewing access forever | `Refresh Token` is opaque, stored only as a hash, rotated on every use, with reuse detection by token family, and bound to a persisted, revocable `UserSession` (INV-011). For the web app it travels only in an HttpOnly, `SameSite=Strict` cookie scoped to `/api/v1/auth` (`ADR-010`), so page scripts cannot read it |
| S4  | Attacker brute-forces a user's password                                                | `failed_login_attempts` increments on each failure and flips `status` to `BLOCKED` once a configured maximum is reached (INV-005: a `BLOCKED` account must not authenticate again), enforced by `RegisterLoginErrorUseCase`                                                                                                                          |
| S5 | A fake or cloned device publishes measurements as if it were a real, registered device | Each module authenticates to the MQTT broker with its `device_code` and its own API key, never a user credential (INV-032); the broker (Mosquitto go-auth) asks `en-monitor-device-service`, which stores the key only as a SHA-256 hash, and the ACL limits each module to its own topics. The topic, not the payload, identifies the device, and the measurement service stores a reading only for a device the device service knows (`09-microservices/services/04-measurement-service/`) |
| S6 | One microservice calls another pretending to be a different, more trusted service | **Partially mitigated.** `/internal/**` routes accept a call only from loopback or the Docker network (`172.16.0.0/12`) **and** with `X-Internal-Token`, compared in constant time; the gateway and the web nginx never forward `/internal/**`, and the services trust `X-Forwarded-For` only from loopback or the Docker network. The token is one shared value (`INTERNAL_API_TOKEN`), so it proves "an internal service", not *which* one; there is no mTLS |
| S7  | Attacker without any account gains a session through a missing multi-factor step       | **Not mitigated / not considered.** No MFA mechanism appears in any document reviewed                                                                                                                                                                                                                                                                |

---

## Tampering (integrity)

_Can data be modified in transit or at rest without authorization or detection?_

| #   | Threat                                                                                        | Implemented mitigation                                                                                                                                                                                                                                                                                               |
| --- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T1  | Malformed or out-of-range values enter the domain through the API (e.g. a negative threshold) | Value Objects validate themselves at construction — `Threshold` enforces `daily_limit ≤ monthly_limit` in its constructor rather than trusting the caller (`06-data/data-dictionary.md`); `Email` is validated and normalized before use (`02-domain/entities-and-rules.md`, §3)                                     |
| T2 | A measurement is altered between the device and the measurement service | The broker authenticates each module and restricts it to its own topics (see S5), and the connection from the module is TLS (see T3). A genuine device's payload is not signed end to end |
| T3 | Traffic between devices and the MQTT broker is intercepted or altered | **Mitigated.** The modules connect over TLS on 8883 (`mqtt.energy-monitor.tech:8883`, Let's Encrypt certificate in production, a development CA locally). The 1883 listener without TLS is reachable only inside Docker (and on `127.0.0.1` in the local stack) (`ADR-007`) |
| T4 | Traffic between clients and the backend, or between services, is intercepted or altered | **Mitigated at the edge.** Nginx on the host terminates HTTPS (Let's Encrypt via Certbot) for `energy-monitor.tech`; neither the gateway nor the services publish a port. Traffic between containers inside the Docker network is plain HTTP/AMQP (`05-architecture/deployment.md`) |
| T5 | A domain event is altered in the message broker before a consumer processes it | **Not mitigated by the message itself.** No message signing is used on the RabbitMQ events (`ADR-008`); the broker is reachable only inside Docker (AMQP on `127.0.0.1` in the local stack) and needs its own credentials (`RABBITMQ_*`) |
| T6  | SQL injection through user-supplied input                                                     | **Likely mitigated by the stack's defaults (Spring Data JPA/Hibernate uses parameterized queries), but this is inherited, not an explicit documented decision.** No input-sanitization or query-construction convention is written down anywhere in this repository                                                  |

---

## Repudiation (accountability)

_Can a user or device deny having done something they actually did?_

| #   | Threat                                                                                                                                                                                   | Implemented mitigation                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | A user denies performing a security-relevant action (login, role change, etc.)                                                                                                           | `AuditLog` is an immutable record of who did what, when, and from where (`01-context/scope.md`, feature #4; `06-data/models.md`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| R2  | A user denies a failed login attempt was theirs, or claims the account was locked without cause                                                                                          | `LoginErrorLog` records every failed attempt with its `error_type`, `ip_address`, and timestamp — independent of `AuditLog`, so failed attempts are traceable even before an account is ever successfully accessed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| R3  | A user denies a session (and the actions taken during it) was theirs                                                                                                                     | `UserSession` records `ip_address`, `user_agent`, `created_at`, and `closed_at` for every session, not just the currently active one                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| R4  | **A security-relevant record is deleted, removing the evidence it was meant to preserve**                                                                                                | **Gap, not a mitigation.** The universal soft-delete convention added to every table (`06-data/models.md`, principle #3) applies to `audit_log` and `login_error_log` exactly like any other table — a `deleted_at` value hides an audit record from normal queries just as it would hide any other row. This directly undermines the purpose of R1–R3 above: an audit trail that can be "soft-deleted" is not non-repudiation, it is repudiation with an extra step. This table should very likely be excluded from the standard soft-delete convention, or given a stricter access rule for who can set `deleted_at` on it — currently nothing distinguishes it from `home_type` or any other catalog table |
| R5  | It is unclear whether `AuditLog` covers business actions outside Security (e.g. who resolved an `Alert`, who changed a `Home`'s thresholds), or only authentication/authorization events | **Open question.** `AuditLog`'s `action` enum (`CREATE`, `UPDATE`, `DELETE`, `LOGIN`, `LOGIN_FAILED`, `LOGOUT`) is generic enough to represent any entity's lifecycle, but `id_user` is its only structured reference — there is no column identifying _which_ entity or table was affected, only a free-text `description`. Whether every microservice is expected to write to this same table, or only `security-service`, is not documented                                                                                                                                                                                                                                                                |

---

## Information Disclosure (confidentiality)

_Can an attacker see data they should not be able to see?_

| #   | Threat                                                                                          | Implemented mitigation                                                                                                                                                                                                                                                                                                                      |
| --- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| I1  | A device's `ApiKey` leaks through a log line or an API response                                 | The `ApiKey` Value Object must be stored as a secret, must never be exposed in responses or logs, and must expose a masked representation (e.g. `toString()`) so accidental logging cannot leak the real value (`02-domain/entities-and-rules.md`, §3)                                                                                      |
| I2  | A password hash, or the plain password itself, leaks through a log or an error message          | `password_hash` is never stored or logged in any form other than the hash itself, and is produced only by the `PasswordHasher` port (`06-data/data-dictionary.md`)                                                                                                                                                                          |
| I3  | Internal persistence/schema details leak to API clients through response bodies                 | Hexagonal Architecture keeps **Persistence Entities** in the persistence adapter only, never exposed to the domain or the client; **DTOs** are what the web boundary actually returns, so a database column is never a direct 1:1 mirror of an API field (`01-context/glossary.md`)                                                         |
| I4 | A user reads another home's data by guessing or manipulating an ID (IDOR) | **Partially documented.** The device, measurement, alert and recommendation services ask the home service (`GET /internal/homes/{homeId}/access`) whether the caller is a member or the owner of the home before answering, and a device of another home answers 404 like one that does not exist (membership is never cached). The enforcement point inside the home service's own endpoints and the notification service's inbox is not written down in this repository |
| I5 | A device's `ApiKey` or a measurement's contents are sniffed over an unencrypted MQTT connection | **Mitigated** — TLS on 8883, see T3 |
| I6 | An unhandled exception returns a stack trace or internal error detail to the client | **Partially mitigated.** Errors are rendered in one shared JSON shape (`timestamp`, `status`, `error`, `message`, `path`) by each service's exception handler and by the gateway (`05-architecture/cross-cutting.md`, "Error handling") |

---

## Denial of Service (availability)

_Can the system, or a piece of it, be made unavailable?_

| #   | Threat                                                                                       | Implemented mitigation                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| --- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| D1 | A single malformed or "poison" domain event blocks every event behind it in a consumer | Every RabbitMQ queue is durable with a dead-letter queue: a message that fails three attempts (1, 2 and 4 s apart) is moved there instead of blocking the queue (`ADR-008`) |
| D2  | Brute-force login attempts exhaust service resources or lock out real users                  | `failed_login_attempts` → `BLOCKED` bounds how many attempts a single account tolerates (see S4) — **but this is also a new attack surface**: nothing prevents an attacker from deliberately entering wrong passwords for a _victim's_ account purely to force it into `BLOCKED`, a denial-of-service against that specific user. No distinction is documented between "the account owner is failing to log in" and "someone else is attacking this account" |
| D3 | A compromised or malfunctioning device floods the measurement service with publishes | **Not mitigated.** No per-device publish rate limit or broker-level quota is configured; the ACL only limits a module to its own topics |
| D4 | A client or attacker floods the REST endpoints | **Partially mitigated.** The gateway limits the public account `POST`s (sign in, registration, Google, refresh, recovery, verification) to 20 per client address and path per minute, then answers 429; the security service also limits password-recovery redemption to ten attempts per caller address per 15 minutes. Both counters are in memory, per instance. Authenticated endpoints have no rate limit |
| D5  | Infrastructure-level flooding (network-layer DoS)                                            | **Not applicable to mitigate directly at this project's scale.** `01-context/scope.md`'s budget constraint rules out most managed DDoS-protection services; this is accepted as an unaddressed risk given the project's academic scope, not a decision to revisit soon                                                                                                                                                                                       |

---

## Elevation of Privilege (authorization)

_Can an actor do something they are not authorized to do?_

| #   | Threat                                                                                                                                                                    | Implemented mitigation                                                                                                                                                                                                                                                                                                                                               |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| E1 | A user without the right permission calls a protected endpoint | RBAC via `SystemRole` → `SystemRolePermission` → `Permission`, assigned through `UserSystemRole`; the security service checks the permission code (`USER.READ`, `USER.MANAGE`, `ROLE.MANAGE`) against the database on every admin request, and the notification service asks the same codes over `GET /internal/users/{id}/permissions` (cached 30 s) |
| E2  | A privilege check is bypassed because it used the human-readable permission name instead of its stable code                                                               | `Permission.code` is the only identifier checked in authorization logic; `06-data/data-dictionary.md` states explicitly that `code` must never be used for anything user-facing and `name` must never be used in an authorization check — closing a common class of bug where a display label silently drifts and breaks the check it was never meant to be used for |
| E3  | A `Home Role` (`MEMBER`/`OWNER`, scoped to one home) is confused with a global `System Role`, letting a home-scoped permission leak into system-wide access or vice versa | `02-domain/domain-map.md` explicitly documents these as two different meanings of "role" in two different bounded contexts, never to be conflated: `System Role` in Security is global, `Home Role` in Home Management is scoped to exactly one home                                                                                                                 |
| E4 | A compromised measuring device is used as a channel to send commands back into the home network or the services | Commands exist (`energy-monitor/devices/{deviceId}/command`, for example `unlink`), but only `en-monitor-device-service` may publish them; each module may publish and subscribe only on its own topics (broker ACL decided by the device service). No other service is an MQTT client |
| E5  | A `Home Member` performs an action reserved for the `Home Owner` (e.g. inviting/removing members, changing thresholds)                                                    | **Not documented as enforced.** `01-context/scope.md` (feature #5) describes member management as something the Home Owner does, implying `OWNER`-only enforcement is intended, but no use case or authorization rule confirms this check is actually implemented anywhere in `02-domain/entities-and-rules.md`                                                      |
| E6 | A valid, authenticated user with no `SystemRole` assigned at all reaches a protected endpoint | **Default deny in the security service:** its filter chain denies everything not permitted on purpose, and any `/api/v1/admin/**` route without its permission code answers 403. The gateway also requires a valid token on every route except the public account endpoints and `GET /api/v1/support/contacts`. The `ADMIN` role grants no access to any home |

---

## Regulatory requirement: Habeas Data (Law 1581 of 2012)

STRIDE is a threat taxonomy; it does not have a category for "does this system comply with a
specific data-protection law." `04-requirements/non-functional.md` (ERNF3.1) commits this project
to Colombia's Law 1581 of 2012 anyway, so it is checked here as its own short pass, mapped back to
the STRIDE threats above where they overlap rather than treated as unrelated.

| #   | Requirement (Art. 4 principle)                                                                             | Status                                                                                                                                                                                                                                                                                                            |
| --- | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| H1  | Libertad — prior, express, informed consent before collecting personal data                                | **Not mitigated.** `HU1` requires accepting "terms, conditions, and license permissions" at registration, but no field on `user` (`06-data/models.md`) records that consent was given — there is no evidence trail for something the product's own acceptance criteria require                                    |
| H2 | Seguridad — technical measures preventing unauthorized access to personal data | **Partially mitigated.** Transport is encrypted at the edge (T3/T4) and the device and measurement services check home membership (I4); home isolation inside the home service is still not documented (I4) |
| H3  | Transparencia — the data subject can obtain what data is held about them, on request                       | **Not mitigated.** No consolidated "my data" capability exists; overlaps with **I3** (DTOs prevent _accidental_ over-exposure) but does not satisfy it — I3 is about not leaking too much, H3 is about being able to deliberately show everything on request, which is a different, unbuilt capability            |
| H4  | Confidencialidad — everyone who handles the data keeps it reserved, including after their involvement ends | **Not documented.** No confidentiality expectation is written down for the two people with direct database access (`01-context/scope.md`) — closer to a **Repudiation**-adjacent gap than an Information Disclosure one: if it is never written down who agreed to what, there is nothing to hold anyone to later |

Full principle-by-principle detail (all eight, including the ones already satisfied structurally
by RBAC and the correction use case) is in `05-architecture/cross-cutting.md`, "Regulatory
compliance — Habeas Data." This table only lists what is **not** mitigated, cross-referenced back
into STRIDE.

---

## Summary

| STRIDE category | Implemented | Gaps found |
|---|---|---|
| Spoofing | 5, plus 1 partial (S6) | 2 (shared internal token instead of per-service identity, MFA) |
| Tampering | 3 (T1, T3, T4), plus 1 partial (T2) | 2 (event integrity, explicit input handling) |
| Repudiation | 3 | 2 (soft-delete on audit tables, cross-context audit coverage) |
| Information Disclosure | 4, plus 2 partial (I4, I6) | 1 (home isolation inside the home service not documented) |
| Denial of Service | 1, plus 1 partial (D4) | 3 (device flood, infra DoS; D2 side-effect risk) |
| Elevation of Privilege | 5 | 1 (home-role enforcement in the home service not documented) |
| Habeas Data (Law 1581) | 0, plus 1 partial (H2) | 3 (consent evidence, "my data" access, team confidentiality) |

The most consequential single gap is **R4** (soft delete applied to `audit_log`): it does not just
leave a threat unmitigated, it quietly weakens a mitigation this document initially credited as
implemented. It is worth resolving before the others; it does not depend on any infrastructure
decision (the MQTT broker, the API gateway and TLS are all in place since 24-Oct-2026; the migration to eight Java services finished on 28-Oct-2026).

---

## Correlations

- Entities, invariants, and Value Objects referenced above → `02-domain/entities-and-rules.md`
- Term definitions (Access Token, Refresh Token, API Key, DTO, Persistence Entity) → `01-context/glossary.md`
- Table-level schema, including the soft-delete convention behind R4 → `06-data/models.md`
- Domain event delivery guarantees and DLQ behind D1 → `02-domain/domain-events.md`
- Infrastructure this model depends on (MQTT broker, API gateway, TLS termination) → `05-architecture/deployment.md`, `05-architecture/overview.md`
- Error handling and retry behaviour → `05-architecture/cross-cutting.md`; rate limiting → `ADR-004-api-gateway.md`
- Full eight-principle Habeas Data table (H1–H4 above only list what is unmitigated) → `05-architecture/cross-cutting.md`, "Regulatory compliance — Habeas Data"
- The requirement H1–H4 trace back to → `04-requirements/non-functional.md` (ERNF3.1)
