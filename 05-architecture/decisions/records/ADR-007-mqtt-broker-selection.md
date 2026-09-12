# ADR-007 — MQTT Broker Selection

- **ID:** ADR-007
- **Date:** 2026-09-11
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra

---

## Context

Measuring devices (ESP32 + PZEM-004T V3) need to publish electrical readings to the backend
continuously and automatically, with no manual step involved. MQTT fits this constraint: it is
designed for constrained microcontrollers and unreliable Wi-Fi connections, and its
publish/subscribe model means a device never needs to know which service ultimately consumes its
readings. This ADR decides which broker product receives those publications — without one
running, `measurement-service` has nothing to connect to, and device ingestion cannot be
implemented or tested end to end.

This is a separate decision from `ADR-002-domain-event-transport.md`: that ADR covers how the six
backend services communicate with each other (Kafka); this one covers how the ESP32 devices
publish readings to the backend, over MQTT.

**Known constraints:**
- Two-person team, fixed delivery deadline of the last week of October 2026
  (`01-context/scope.md`)
- No budget for paid cloud infrastructure or third-party services (`01-context/scope.md`) — a
  self-hosted, free option is required
- The device side runs on an ESP32 (`01-context/overview.md`), a resource-constrained
  microcontroller — the broker itself does not need to handle that constraint, but the project's
  overall footprint should stay small given the team is also running MySQL, and now Kafka
  (`ADR-002-domain-event-transport.md`), on the same limited development hardware

---

## Decision

Use **Eclipse Mosquitto** as the MQTT broker for device telemetry ingestion.

**We decided:** every measuring device (ESP32 + PZEM-004T V3) publishes its readings to a
Mosquitto broker, which `measurement-service` subscribes to.

**Justification:** Mosquitto's resource footprint is low enough to run comfortably alongside
MySQL and Kafka on the same development environment, and it is well suited to prototypes and
small projects — this project's actual scale, at two developers and a fixed academic deadline,
not a production IoT fleet with thousands of devices. It does the one job this ADR needs done
(receive MQTT publications from devices, deliver them to a subscriber) without operational
overhead beyond that.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|-------------|------|------|------------------------|
| **Eclipse Mosquitto (CHOSEN)** | Low resource consumption; well suited to prototypes and small projects; simple to run self-hosted in Docker at no cost | Fewer built-in enterprise features (clustering, advanced access control, a management UI) than EMQX | — (chosen) |
| EMQX | More built-in features — clustering, a management dashboard, finer-grained access control; scales further for a larger device fleet | Heavier resource footprint than Mosquitto; most of its extra capability targets a scale and an operational team this project doesn't have | The extra capability is aimed at a scale this project isn't at, at a resource cost this team's development hardware doesn't need to pay |

---

## Consequences

**Positive:**
- Low resource consumption leaves more headroom for MySQL and Kafka on the same development
  environment
- Free and self-hosted in Docker, consistent with the no-budget constraint
- Minimal setup matches the two-person team's capacity to operate it without dedicated ops effort

**Negative / Trade-offs:**
- No built-in clustering or management dashboard — if the device fleet or the reliability
  requirements grow significantly beyond this project's current scope, migrating to something
  like EMQX (or adding tooling Mosquitto doesn't provide out of the box) becomes a real
  reconsideration, not a free upgrade
- Access control and monitoring around the broker have to be set up more manually than they would
  be with a product that bundles them in

**Impact on the system:**
- Affected: `measurement-service` (the subscriber), and every measuring device (the publishers)
- Documents that must be updated: `01-context/scope.md`'s feature #11 should be updated from "not
  yet designed" to reference this ADR; `01-context/overview.md`'s technology stack table should
  name Mosquitto specifically instead of "MQTT over Wi-Fi" generically

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Mosquitto's simpler access control makes it easier to misconfigure device authentication than a product with built-in fine-grained ACLs | Low-Medium | Medium | Follow Mosquitto's documented authentication setup explicitly (per-device credentials or certificates) rather than leaving the broker open |
| The device fleet grows beyond what a single Mosquitto instance comfortably handles | Low, at this project's current and expected scale | Low | Revisit only if load testing or real deployment shows an actual problem — do not pre-optimize for a scale this project hasn't reached |

---

## References

- Open item this ADR resolves → `01-context/scope.md`, feature #11
- The device-to-backend relationship this broker sits in → `02-domain/domain-map.md`,
  `Devices → Monitoring and Measurements`
- Related to: `ADR-002-domain-event-transport.md` (a separate broker, Kafka, for a separate
  concern — inter-service domain events, not device telemetry)