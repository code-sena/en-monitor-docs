# ADR-002 — Domain Event Transport

- **ID:** ADR-002
- **Date:** 2026-09-11
- **Status:** Superseded by ADR-008
- **Authors:** Juan Esteban Cortés Parra

> **Superseded on 2026-10-20 by `ADR-008-rabbitmq-transactional-outbox.md`.** Never implemented:
> no Kafka broker, topic or client exists in any repository. Kept unchanged below as the record of
> the decision taken on 2026-09-11.
---

## Context

`02-domain/domain-events.md` defines two domain events — `MeasurementRecorded` and
`DeviceConnectivityLost` — that must travel from the context that publishes them
(Monitoring and Measurements, Devices) to the contexts that react to them (Alerts,
Recommendations). That document left the transport mechanism as an open question, because the
only broker technology this project had committed to — MQTT — was chosen specifically for device
telemetry ingestion (`01-context/scope.md`, feature #11), not for service-to-service
communication.

Without this decision, `AlertEvaluationService` and `RecommendationGenerationService` cannot be
implemented: both depend on receiving these events, and the channel, delivery guarantee, and
consumer pattern they are built against depend directly on what this ADR decides.

**Known constraints:**
- Two-person team, fixed delivery deadline of the last week of October 2026
  (`01-context/scope.md`)
- No budget for paid cloud infrastructure or third-party services (`01-context/scope.md`) — a
  self-hosted option in Docker is required
- MQTT is already committed as the device-to-backend telemetry channel and is not being replaced;
  this decision is about a separate concern, service-to-service events

---

## Decision

Use **Apache Kafka** as a dedicated message broker for inter-service domain events, kept
separate from the MQTT channel used for device telemetry ingestion.

**We decided:** `MeasurementRecorded` and `DeviceConnectivityLost` (and any future domain event)
are published to and consumed from Kafka topics, not carried over the existing MQTT connection
and not handled with direct in-process calls between services.

**Justification:** Kafka provides materially higher data transmission throughput than RabbitMQ,
the other option evaluated. Given `Measurement` records are expected to be the highest-volume
data in the system (continuous readings from every registered device), the broker chosen for the
events derived from that stream needs headroom to grow with it. Kafka's native consumer-group
model also fits this project's actual fan-out shape directly: both `alert-service` and
`recommendation-service` need to consume every `MeasurementRecorded` event independently, which
is what consumer groups are for.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|-------------|------|------|------------------------|
| **Apache Kafka (CHOSEN)** | Higher throughput than RabbitMQ; built-in log retention enables replaying events for debugging; consumer groups match the project's fan-out pattern (Alerts and Recommendations both consuming `MeasurementRecorded`) | Heavier operational footprint (requires ZooKeeper or KRaft mode, more memory); steeper learning curve for a two-person team | — (chosen) |
| RabbitMQ | Mature, simpler day-to-day operation than Kafka; strong routing/queueing semantics; lower resource footprint, easier to run on a laptop | Lower data transmission throughput than Kafka | Throughput ceiling is the deciding factor given the volume `Measurement` data is expected to reach |
| Reuse the existing MQTT broker for domain events too | No new infrastructure to run; team is already working with MQTT for device ingestion | Conflates two different concerns (device telemetry vs. service-to-service events) under one channel; MQTT's pub/sub model does not provide the retention/replay or consumer-group semantics the resilience patterns in `02-domain/domain-events.md` already assume | Mixing the device channel with the inter-service channel makes both harder to reason about and evolve independently |
| In-process / synchronous calls between services | Zero infrastructure; simplest option while the six services are still early in development | Stops working the moment services are deployed independently, which is the explicit target architecture (`02-domain/domain-map.md`); would require a rework later instead of building the real integration once | Only postpones the decision this ADR needs to make now |

---

## Consequences

**Positive:**
- Alerts and Recommendations are decoupled from Monitoring and Measurements and Devices — they
  react to events instead of being called directly
- Kafka's retention makes it possible to replay `MeasurementRecorded` events when debugging
  `AlertEvaluationService` or `RecommendationGenerationService` without needing new test data
- Consumer groups let both `alert-service` and `recommendation-service` independently consume the
  same event stream without either one blocking the other

**Negative / Trade-offs:**
- A second broker technology enters the stack — MQTT for device telemetry, Kafka for
  inter-service events — meaning two pieces of messaging infrastructure to run and operate
  instead of one
- Kafka's operational complexity (partitions, consumer group offsets, ZooKeeper/KRaft) is new
  ground for a two-person team working against a fixed deadline
- Self-hosting Kafka in Docker has no monetary cost but a real resource cost (memory, CPU) that
  needs to be confirmed against the team's actual development hardware
- A single self-hosted Kafka container is sufficient for development and the academic
  evaluation, but is not a production-grade deployment. If this project is ever operated beyond
  that scope — more homes, more devices, real users — a fault-tolerant Kafka setup typically
  needs a multi-broker cluster, which usually means either paying for a managed service or
  investing in infrastructure to operate that resilience level. This is outside the current
  no-budget constraint (`01-context/scope.md` scopes it to "development and evaluation"), not a
  contradiction of it — but it is a cost this decision defers, not one it avoids. What does carry
  over cleanly to that future stage is the client-side code: topics, partitions, and consumer
  group logic built against Kafka's API do not need to be rewritten to point at a managed Kafka
  service later — only the connection configuration changes

**Impact on the system:**
- Affected services: `measurement-service` and `device-service` (publishers), `alert-service` and
  `recommendation-service` (consumers)
- Documents that must be updated: `02-domain/domain-events.md` (the `Channel (topic)` and
  `Delivery guarantee` fields, currently "Not yet decided", can now be filled in with Kafka topic
  names); `02-domain/domain-map.md` (the relationship rows for `Monitoring and Measurements → Alerts`,
  `Monitoring and Measurements → Recommendations`, and `Devices → Alerts` can now name Kafka
  explicitly instead of pointing to this open question); `01-context/overview.md` (the technology
  stack table does not currently list a broker for inter-service events — Kafka should be added
  alongside MQTT, with each one's role distinguished)

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| The team's unfamiliarity with Kafka consumes disproportionate time against the fixed October 2026 deadline | Medium | High | Time-box a short spike to get a minimal Kafka setup running in Docker before committing further code to it; fall back to RabbitMQ if the spike shows the learning curve is too costly |
| Self-hosted Kafka's memory/CPU footprint is too heavy for the team's development machines | Low-Medium | Medium | Confirm minimum resource requirements against actual hardware before final adoption |
| Running two broker technologies (MQTT + Kafka) creates confusion about which one carries which kind of message | Medium | Low | Keep the distinction explicit in documentation — MQTT is device-to-backend telemetry only (`01-context/scope.md`, feature #11); Kafka is service-to-service domain events only (this ADR) |

---

## References

- Open question this ADR resolves → `02-domain/domain-events.md`
- MQTT's separate, unrelated role (device telemetry) → `01-context/scope.md`, feature #11
- Bounded contexts and relationships this transport connects → `02-domain/domain-map.md`
- Related to: `ADR-006` (MQTT broker product selection — a separate decision, for the device
  telemetry channel this ADR does not change)