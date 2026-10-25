# Pattern Guide

> This document catalogs the design patterns already adopted in the project (per
> `01-context/glossary.md` and `02-domain/entities-and-rules.md`) and the ones that have been
> discussed but not yet decided. Each entry states when to use the pattern, when not to, and a
> concrete example from this project — not a generic textbook example.

---

## Patterns in use

### Hexagonal Architecture (Ports and Adapters)

**When to use it:** when the domain's business rules need to stay testable and framework-free,
and the same business logic must be reachable from more than one entry point (REST today,
possibly a scheduled job or a message consumer later) without duplicating rules.

**When NOT to use it:** for a throwaway script or a service so small that the indirection of
ports/adapters costs more than the coupling it prevents — this project does not have that case,
since every service and backend module has real business rules and at least two adapters (a web
adapter and a persistence adapter).

**Project example:** every microservice separates a **domain model** (framework-free business
state and rules), **use cases** (application-layer coordinators of one business operation, not
thin wrappers around a single entity method), **inbound ports** (what a use case exposes), and
**outbound ports** (what a use case needs from the outside, such as persistence) — each
implemented by an adapter. `security-service`'s `PasswordHasher` is an outbound port: the domain
depends on the interface, never on a specific hashing library, so the concrete algorithm can
change without touching a use case. Full definitions: `01-context/glossary.md`.

---

### Repository (outbound port + persistence adapter)

**When to use it:** whenever a use case needs to load or save an aggregate and should not know
whether that aggregate lives in MySQL, a cache, or anywhere else.

**When NOT to use it:** for read-only, cross-context queries that intentionally bypass the
aggregate boundary (a reporting/statistics query spanning several tables) — those are better as a
dedicated query, not forced through a single-aggregate repository interface.

**Project example:** `Measurement`'s lifecycle is `[CREATED] → (stored in the repository) →
[STORED]` (`02-domain/entities-and-rules.md`). The repository is an outbound port; its concrete
implementation is a **persistence adapter** built on **persistence entities** (classes that map a
database table and never leak into the domain) plus a **mapper** that translates between the two,
per the Port/Adapter/Persistence Entity/Mapper definitions in `01-context/glossary.md`.

---

### Use Case (Application Service)

**When to use it:** to coordinate one meaningful business operation end-to-end (validate,
apply business rules, call outbound ports), as the single entry point a controller or event
consumer calls into.

**When NOT to use it:** for a trivial getter that does not coordinate anything — exposing it
directly through a query port is enough; wrapping it in a "use case" that does nothing but call
a repository method is the anti-pattern this pattern exists to avoid (`01-context/glossary.md`:
_"Not a wrapper around one entity method"_).

**Project example:** `RegisterLoginErrorUseCase`, referenced from `data-dictionary.md`'s note on
`user.failed_login_attempts` — it is the one place that increments the failed-attempt counter and
flips `status` to `BLOCKED` once the configured maximum is reached, rather than that rule being
duplicated wherever a login is attempted.

---

### Value Object

**When to use it:** for a concept defined entirely by its value, with no identity of its own,
that should be immutable and validate itself at construction so an invalid instance can never
exist.

**When NOT to use it:** for anything the business needs to track individually over time, even if
two instances currently hold the same data — that calls for an Entity instead (an `AccessCode`
value is a VO; the `Home` it belongs to is an Entity, tracked by identity even if its address
changes).

**Project example:** `Threshold` (`02-domain/entities-and-rules.md`, System Value Objects) wraps
`daily_limit`/`monthly_limit` and enforces `daily_limit ≤ monthly_limit` in its constructor — a
rule `06-data/data-dictionary.md` notes is **not** a database constraint, precisely because it
lives in this Value Object instead. `Email`, `AccessCode`, `DeviceCode`, `ApiKey`, `ActivePower`,
`Energy`, and `MessageKey` follow the same pattern (`02-domain/entities-and-rules.md`, §3).

---

### Observer

**When to use it:** when one thing happening should notify other parts of the system without the
source knowing or caring who is listening, or how many listeners there are.

**When NOT to use it:** when the "notification" is really a synchronous question that needs an
answer before continuing — that is a direct call, not an Observer relationship.

**Project example:** a new `Measurement` notifies subscribers without coupling ingestion to the
reaction it triggers (`01-context/glossary.md`). Across processes this is realized as the
`measurement.recorded` RabbitMQ message published by `en-monitor-measurement-service`; inside
`energy-monitor-backend` the `measurement` proxy republishes it as the in-memory
`MeasurementRecorded` Spring event, which the `alert` module observes
(`02-domain/domain-events.md`) — the same decoupling purpose at two different scopes.

---

### Factory Method

**When to use it:** when the caller needs an object built for it (here, a notification) without
knowing or choosing the concrete type — the creation logic picks the type based on the situation.

**When NOT to use it:** when there is only ever one concrete type to build — a plain constructor
call is clearer than a factory that always returns the same thing.

**Project example:** building notifications from measurements or device status without the caller
knowing the concrete type (`01-context/glossary.md`) — an `Alert` raised from a `THRESHOLD`
condition and one raised from a `CONNECTIVITY` condition are both notifications, built through the
same factory entry point, without the alert-raising code branching on which case it is.

---

### Singleton

**When to use it:** for a value that is genuinely system-wide and should have exactly one source
of truth — not per-request, per-user, or per-home.

**When NOT to use it:** for anything that varies per home or per user, such as a specific home's
own thresholds — using a Singleton there would leak one home's configuration into every other
home's evaluation.

**Project example:** system-wide default consumption thresholds. `06-data/data-dictionary.md`
documents `home_thresholds.use_system_defaults`: when `true`, the home's own `daily_limit` /
`monthly_limit` are ignored and the platform-wide defaults are used instead — those platform-wide
defaults are exposed from the single Singleton instance referenced in `01-context/glossary.md`.

---

### Idempotent Consumer

**When to use it:** for any consumer of an at-least-once delivery channel, where the same message
can legitimately arrive more than once.

**When NOT to use it:** as a substitute for exactly-once delivery guarantees the business
actually needs at the broker level — idempotency handles duplicate _processing_, not duplicate
_delivery_; if duplicates must never even reach the consumer, that is a different (and usually
more expensive) guarantee to buy from the broker, not something this pattern promises.

**Project example:** every RabbitMQ consumer tolerates a second delivery, since the outbox relay
delivers at least once with the message id: the device service's `home.deleted` consumer finds no
module left to unlink the second time, and the measurement service's `device.linked/unlinked`
consumer only drops a cache entry again (`02-domain/domain-events.md`, "At-least-once delivery +
Idempotency"). This is the same underlying property as the `Idempotence` glossary term
that also governs raw MQTT measurement ingestion (`01-context/glossary.md`).

---

### Dead Letter Queue (DLQ)

**When to use it:** for an async consumer that can fail on a specific message in a way retries
will not fix (malformed payload, a business rule that will never pass) — so one bad message
cannot block every message behind it.

**When NOT to use it:** as a place to silently lose messages — a DLQ with no alert on non-zero
depth and no runbook for draining it is a queue nobody will ever look at again.

**Project example:** every RabbitMQ consumer queue is durable and has a dead-letter queue; a
message that fails three attempts (1, 2 and 4 s apart in the device and measurement services) is
moved there, for example to `energy-monitor-backend.measurement.recorded.dlq` or
`device-service.device.home-deleted.dlq`. There is no automated alert on a non-empty DLQ yet; the
end-to-end test `infra/e2e/measurement-flow.sh` checks that the backend's queue and its `.dlq` end
empty.

---

### Transactional Outbox

**When to use it:** to guarantee a domain event is published if and only if the database
transaction that caused it also commits — avoiding the dual-write problem of writing to the
database and publishing to the broker as two separate, non-atomic operations.

**When NOT to use it:** if the event's business impact of being occasionally lost or duplicated is
low enough that the added table, relay process, and operational overhead are not worth it.

**Project example:** **in use since October 2026** (`ADR-008-rabbitmq-transactional-outbox.md`).
Each producer has its outbox table — `security_outbox`, `home_outbox`, `device_outbox`,
`measurement_outbox` (`06-data/models.md`, "Outbox tables") — written in the same transaction as
the change; a relay publishes the pending rows to RabbitMQ every second, oldest first, with
correlated confirms and mandatory routing, and stops at the first failure.

---

## Patterns considered but not yet adopted

These are referenced elsewhere in the documentation as candidates for solving a specific problem,
but no decision has been made to actually implement them. Listed here so the option is not
rediscovered from scratch, and so nobody mistakes a mention for an implementation.

### Saga

**When to use it:** when one business operation needs to modify two or more aggregates and a
single database transaction cannot span them — exactly the situation `02-domain/entities-and-rules.md`
describes as the Aggregate's golden rule: _"Transactions do not cross aggregate boundaries. If you
need to modify two aggregates in one operation, use a Domain Event and a Saga."_

**When NOT to use it:** when the two aggregates involved can tolerate eventual consistency without
a compensating action — a plain domain event with an idempotent consumer (already in use, see
above) is simpler and is what this project already does for `MeasurementRecorded` and
`DeviceConnectivityLost`. A Saga adds compensating-transaction complexity that is only worth it
when a step can fail _after_ an earlier step already committed and that earlier step needs to be
undone.

**Project example:** **not yet implemented.** No cross-aggregate operation in the current scope
(`01-context/scope.md`) has been identified that needs compensation rather than plain eventual
consistency. If one is identified, it should be documented here with the concrete steps and their
compensations — not assumed to already exist because the golden rule mentions it.

---

## Correlations

- Ports, adapters, and the other hexagonal-architecture terms used throughout this document →
  `01-context/glossary.md`
- Entities, Value Objects, and the Aggregate golden rule → `02-domain/entities-and-rules.md`
- Domain events, idempotency, and DLQ configuration → `02-domain/domain-events.md`
- Table-level schema → `06-data/models.md`
- Event transport and outbox decision → `ADR-008-rabbitmq-transactional-outbox.md` (`05-architecture/decisions/records/`, supersedes `ADR-002`)
