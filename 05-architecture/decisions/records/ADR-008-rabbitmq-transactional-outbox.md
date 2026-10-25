# ADR-008 — Event Transport: RabbitMQ with a Transactional Outbox

- **ID:** ADR-008
- **Date:** 2026-10-20
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra

---

## Context

While the backend was a single modular monolith, domain events (`MeasurementRecorded`,
`DeviceConnectivityLost`, `AlertRaised`, ...) were Spring application events delivered in
memory, inside one process (`02-domain/domain-events.md`). `ADR-002` had chosen Kafka for the
moment the contexts became separate processes, but no Kafka code was ever written.

Extracting the Security context to `en-monitor-security-service` (October 2026) made the
question concrete: account events and mail requests now had to cross from one process to
another, and a change committed in the security service could not be allowed to lose its
message, nor a rolled-back change to send one. The device and measurement extractions that
followed had the same need.

**Known constraints:**
- Two-person team, delivery on 28-Oct-2026 (`01-context/scope.md`)
- No budget for paid infrastructure: the broker has to be self-hosted in the Docker stack of the
  `energy-monitor-infra` repository
- One shared MySQL schema (`ADR-003-shared-database.md`), so each service's outbox table lives in
  that schema next to its own tables
- MQTT stays the device telemetry channel (`ADR-007-mqtt-broker-selection.md`); this decision
  only covers service-to-service events

---

## Decision

**We decided:** service-to-service events go through **RabbitMQ 4.1** topic exchanges, one per
producing context, and every producer writes them through a **transactional outbox** table in
its own part of the schema.

| Exchange | Outbox table (producer) | Routing keys |
|---|---|---|
| `energy-monitor.security` | `security_outbox` (security service) | `account.registered`, `account.administered`, `account.deleted`, `mail.email-verification`, `mail.password-reset` |
| `energy-monitor.home` | `home_outbox` (backend) | `home.deleted` |
| `energy-monitor.device` | `device_outbox` (device service) | `device.linked`, `device.unlinked`, `device.connectivity-lost`, `device.connectivity-restored` |
| `energy-monitor.measurement` | `measurement_outbox` (measurement service) | `measurement.recorded` |

How it works in all four producers:

- The outbox row is written **in the same database transaction** as the change it announces.
- A relay publishes the pending rows **every second, oldest first**. A row counts as published
  only when the broker confirms it (correlated confirms) and a queue took it (mandatory routing);
  the first failure stops the run, so order is kept and the row is retried.
- Delivery is **at least once**. Every message carries its id (`id_message`), and consumers
  tolerate duplicates.
- Consumers declare their own **durable queues**, each with a dead-letter queue, and a message
  that fails three attempts goes to the dead-letter queue.
- Events between modules that still share the backend process (`AlertRaised`,
  `RecommendationCreated`, `HomeOwnershipTransferred`) stay Spring in-memory events. The backend's
  proxy packages republish `measurement.recorded` and `device.connectivity-*` in memory as
  `MeasurementRecorded` and `DeviceConnectivityLost/Restored` for the `alert` module.

**Justification:** the outbox gives the one guarantee the extraction needed: a change is never
committed without its message, and a rolled-back change never sends one. RabbitMQ instead of the
Kafka of `ADR-002`, for three reasons:

- `ADR-002` chose Kafka for its throughput, but the real volume is small: each module publishes one
  reading per minute, and the other events follow user actions. Kafka's retention and replay are
  not needed.
- Everything runs on one server (`05-architecture/deployment.md`) next to MySQL, Mosquitto and five
  Java services; RabbitMQ is one lighter container, while Kafka needs its own JVM and KRaft or
  ZooKeeper.
- `ADR-002` already named RabbitMQ as the fallback if Kafka proved too costly for the October
  deadline. Spring AMQP gives per-message publisher confirms, mandatory routing and dead-letter
  queues, which is what the outbox relay and the consumers need.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **RabbitMQ topic exchanges + transactional outbox (chosen)** | Change and message commit together; ordered, confirmed publishing; durable queues with dead-letter queues; one more container in the existing Compose stack | Consumers must be idempotent; one outbox table and relay per producer; messages wait in the outbox until the consumer has declared its queue | — (chosen) |
| Apache Kafka (`ADR-002`) | Higher throughput; retention and replay; consumer groups | A heavier broker (own JVM, KRaft or ZooKeeper) on the one server; throughput and replay are not needed at this volume | The throughput premise of `ADR-002` did not hold; never implemented |
| Publish to the broker right after the commit, without an outbox | No extra table or relay | A message is lost if the broker is down after the commit, and a failure between commit and publish cannot be retried | Breaks the guarantee the extraction needed |
| Keep Spring in-memory events only | No broker to run | Only works inside one process | Impossible once a context runs as its own service |

---

## Consequences

**Positive:**
- A change and its announcement are atomic in every producer
- A broker outage delays messages instead of losing them: they wait in the outbox and are sent
  when the broker is back
- The same pattern is repeated in four codebases, so a new extraction copies it rather than
  designing it again

**Negative / Trade-offs:**
- At-least-once delivery: every consumer has to tolerate a duplicate (for example, a second
  `home.deleted` finds no module left to unlink)
- The relay assumes one instance of each service; a second instance would keep delivery correct
  but could reorder messages (security service documentation)
- Outbox tables grow: `measurement_outbox` (one row per reading) purges published and discarded
  rows after 7 days; pending rows are never purged
- With RabbitMQ down, the services still start, but their messages wait in the outbox and
  `/actuator/health` reports DOWN

**Impact on the system:**
- Affected services: `en-monitor-security-service`, `en-monitor-device-service`,
  `en-monitor-measurement-service`, `energy-monitor-backend`; RabbitMQ in `energy-monitor-infra`
- Documents updated with this ADR: `05-architecture/overview.md`, `05-architecture/deployment.md`,
  `05-architecture/cross-cutting.md`, `09-microservices/`, `06-data/models.md` (outbox tables)
- Also updated: `02-domain/domain-events.md` (per-event channels)

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A consumer is not idempotent and a duplicate delivery repeats an effect | Low | Medium | Message id on every message; consumers written to tolerate a second delivery |
| A poison message blocks a queue | Low | Medium | Three attempts, then the dead-letter queue |
| The broker is down for a long time | Low | Medium | Outbox keeps the messages; health reports DOWN so `docker compose ps` and `deploy.sh` show it |

---

## References

- Supersedes → `ADR-002-domain-event-transport.md`
- Shared schema that holds the outbox tables → `ADR-003-shared-database.md`
- Device telemetry channel, unaffected by this decision → `ADR-007-mqtt-broker-selection.md`
- Outbox pattern → `05-architecture/pattern-guide.md`
- Per-service details → `09-microservices/services/`
- Code: `docs/Architecture.md` ("Messaging") of `en-monitor-security-service`,
  `en-monitor-device-service` and `energy-monitor-backend`; README of
  `en-monitor-measurement-service`
