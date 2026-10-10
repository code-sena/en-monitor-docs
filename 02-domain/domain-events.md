# Domain Events

---

## What is a domain event?

A **Domain Event** communicates that something important occurred in the business.
It is an immutable message that describes the fact in past tense.

```
✓ MeasurementRecorded
✓ DeviceConnectivityLost

✗ RecordMeasurement (this is a command, not an event)
✗ MeasurementUpdated (too generic — what changed?)
✗ MeasurementEvent (does not indicate what occurred)
```

### Difference between Command and Event

| Concept | Intent | Tense | Can fail? |
|---------|--------|-------|-----------|
| **Command** | Instruction to do something | Present | Yes |
| **Event** | Notification of something that occurred | Past | No (it already happened) |

```
Device → [PublishReading] → Monitoring and Measurements → [MeasurementRecorded] → Alerts, Recommendations
              (Command, over MQTT)                              (Event)
```

---

## Event catalog

### Event: MeasurementRecorded

| Field | Value |
|-------|-------|
| **Name** | `MeasurementRecorded` |
| **Bounded Context** | Monitoring and Measurements |
| **Aggregate** | `Measurement` |
| **Trigger** | A device publishes a reading over MQTT and it is accepted and persisted |
| **Consumers** | Alerts (threshold evaluation), Recommendations (pattern update) |
| **Channel (topic)** | Kafka topic `monitoring.measurement-recorded` |
| **Partition key** | `deviceId` — preserves per-device ordering; consumers that need per-home ordering must not assume it across devices |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "MeasurementRecorded",
  "aggregateId": "550e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Measurement",
  "occurredAt": "2026-09-09T14:30:00Z",
  "version": 1,
  "payload": {
    "deviceId": "string — the device that produced the reading",
    "voltage": "number — volts (V)",
    "current": "number — amperes (A)",
    "activePower": "number — watts (W)",
    "storedEnergy": "number — accumulated kilowatt-hours (kWh)"
  },
  "metadata": {
    "correlationId": "550e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The payload does not include a `homeId`. `Measurement` only knows the device that produced it
> (see `id_device` in `02-domain/entities-and-rules.md`) — it does not know which home that
> device belongs to. A consumer that needs the home must resolve it from the `deviceId` through
> the Devices context, never by assuming Measurements can supply it.

> `metadata.userId` is intentionally omitted. This event is device-initiated, not user-initiated —
> no user action caused it.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Alerts | Runs `ConsumptionClassificationService`; if the result is `HIGH` or `CRITICAL`, raises a `THRESHOLD` alert | Yes — uses `eventId` as the idempotency key before creating a new `Alert` |
| Recommendations | Updates the home's consumption pattern; may generate a new `Recommendation` if a pattern threshold is met | Yes — uses `eventId` as the idempotency key |

---

### Event: DeviceConnectivityLost

| Field | Value |
|-------|-------|
| **Name** | `DeviceConnectivityLost` |
| **Bounded Context** | Devices |
| **Aggregate** | `Device` (recorded through `DeviceStatusLog`) |
| **Trigger** | A device stops publishing measurements for longer than the configured timeout |
| **Consumers** | Alerts (connectivity evaluation) |
| **Channel (topic)** | Kafka topic `devices.device-connectivity-lost` |
| **Partition key** | `deviceId` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "660e8400-e29b-41d4-a716-446655440000",
  "eventType": "DeviceConnectivityLost",
  "aggregateId": "660e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Device",
  "occurredAt": "2026-09-09T14:35:00Z",
  "version": 1,
  "payload": {
    "deviceId": "string — the device that stopped reporting",
    "homeId": "string — the home the device belongs to",
    "lastSeen": "ISO 8601 — the last timestamp the device was seen"
  },
  "metadata": {
    "correlationId": "770e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> Unlike `MeasurementRecorded`, this payload can include `homeId` directly. The relationship
> between a device and its home (`DeviceHome`) is owned by the Devices context itself, so it does
> not need to be resolved by the consumer.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Alerts | Raises a `CONNECTIVITY` alert for the affected home and device | Yes — uses `eventId` as the idempotency key |

> **Open question, not yet resolved:** nothing in the current domain model automatically resolves
> a `CONNECTIVITY` alert when the device reconnects. Today it would stay `PENDING` until someone
> resolves it manually. A symmetrical `DeviceReconnected` event and an auto-resolution policy are
> a reasonable addition, but they are not documented here because no business rule for this was
> ever stated — this note exists so the gap is visible instead of silently assumed away.

---
---
---
### Event: DeviceConnectivityRestored

| Field | Value |
|-------|-------|
| **Name** | `DeviceConnectivityRestored` |
| **Bounded Context** | Devices |
| **Aggregate** | `Device` (recorded through `DeviceStatusLog`) |
| **Trigger** | A device that was offline resumes publishing measurements within the timeout |
| **Consumers** | Alerts (may auto-resolve a `CONNECTIVITY` alert) |
| **Channel (topic)** | Kafka topic `devices.device-connectivity-restored` |
| **Partition key** | `deviceId` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "880e8400-e29b-41d4-a716-446655440000",
  "eventType": "DeviceConnectivityRestored",
  "aggregateId": "880e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Device",
  "occurredAt": "2026-09-09T14:40:00Z",
  "version": 1,
  "payload": {
    "deviceId": "string — the device that resumed reporting",
    "homeId": "string — the home the device belongs to",
    "rssi": "number — optional signal strength at reconnection"
  },
  "metadata": {
    "correlationId": "990e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The `homeId` is included because the Devices context owns the `DeviceHome` relationship,
> just like `DeviceConnectivityLost`. Consumers in the Alerts context may use this event to
> auto-resolve a previously-raised `CONNECTIVITY` alert for the same device and home.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Alerts | May auto-resolve a `CONNECTIVITY` alert if one exists for this device and home | Yes — uses `eventId` as the idempotency key |

---
---
---
### Event: DeviceLinked

| Field | Value |
|-------|-------|
| **Name** | `DeviceLinked` |
| **Bounded Context** | Devices |
| **Aggregate** | `Device` |
| **Trigger** | A device is successfully linked to a home via the access code flow |
| **Consumers** | Home Management (updates `DeviceHome` relationship), Alerts (device now reports to home) |
| **Channel (topic)** | Kafka topic `devices.device-linked` |
| **Partition key** | `deviceId` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "AA0e8400-e29b-41d4-a716-446655440000",
  "eventType": "DeviceLinked",
  "aggregateId": "AA0e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Device",
  "occurredAt": "2026-09-09T14:45:00Z",
  "version": 1,
  "payload": {
    "deviceId": "string — the device that was linked",
    "homeId": "string — the home the device was linked to",
    "accessCode": "string — the code used to link the device"
  },
  "metadata": {
    "correlationId": "BB0e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The `homeId` and `accessCode` are included because the linkage is a home-management event;
> the Devices context does not store the home association itself — it is the `DeviceHome`
> relationship in the Home Management context that records the link.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Home Management | Records the `DeviceHome` relationship (idempotent if already linked) | Yes — uses `eventId` as the idempotency key |
| Alerts | Device is now associated with a home for alert evaluation | Yes |

---
---
---
### Event: DeviceUnlinked

| Field | Value |
|-------|-------|
| **Name** | `DeviceUnlinked` |
| **Bounded Context** | Devices / Home Management |
| **Aggregate** | `Device` (aggregate root), `UserHome` (relationship) |
| **Trigger** | A device is unlinked from a home (either by the owner or by account deletion) |
| **Consumers** | Home Management (removes `DeviceHome` relationship), Alerts (device no longer reports to home) |
| **Channel (topic)** | Kafka topic `devices.device-unlinked` |
| **Partition key** | `deviceId` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "CC0e8400-e29b-41d4-a716-446655440000",
  "eventType": "DeviceUnlinked",
  "aggregateId": "CC0e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Device",
  "occurredAt": "2026-09-09T14:50:00Z",
  "version": 1,
  "payload": {
    "deviceId": "string — the device that was unlinked",
    "homeId": "string — the home the device was unlinked from",
    "reason": "string — optional: MANUAL, ACCOUNT_DELETED, etc."
  },
  "metadata": {
    "correlationId": "DD0e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The `reason` field is optional; if absent, the unlink is assumed to be manual via the UI.
> If `ACCOUNT_DELETED`, the `AccountDeleted` event will also be emitted.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Home Management | Removes the `DeviceHome` relationship; device is disassociated from the home | Yes — uses `eventId` as the idempotency key |
| Alerts | Device is no longer associated with a home; related alerts may be resolved or removed | Yes |

---
---
---
### Event: DeviceRegistered

| Field | Value |
|-------|-------|
| **Name** | `DeviceRegistered` |
| **Bounded Context** | Devices |
| **Aggregate** | `Device` |
| **Trigger** | A new device is registered in the system (creates the `Device` entity) |
| **Consumers** | None directly — this is an internal state change event; other services may query the device registry |
| **Channel (topic)** | Kafka topic `devices.device-registered` (optional; may be internal-only) |
| **Partition key** | `deviceId` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "EE0e8400-e29b-41d4-a716-446655440000",
  "eventType": "DeviceRegistered",
  "aggregateId": "EE0e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Device",
  "occurredAt": "2026-09-09T15:00:00Z",
  "version": 1,
  "payload": {
    "deviceId": "string — the newly registered device",
    "deviceCode": "string — the short code identifying the device",
    "apiKeyHash": "string — hash of the device's API key (one-way, for verification)"
  },
  "metadata": {
    "correlationId": "FF0e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The `apiKeyHash` is included for security verification; the raw `api_key` is never stored or transmitted in events.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| None (internal registry update) | — | N/A |

---
---
---
### Event: AlertRaised

| Field | Value |
|-------|-------|
| **Name** | `AlertRaised` |
| **Bounded Context** | Alerts |
| **Aggregate** | `Alert` |
| **Trigger** | An alert transitions from `PENDING` to `ACTIVE` state |
| **Consumers** | Notification (delivers the alert via email/push) |
| **Channel (topic)** | Kafka topic `alerts.alert-raised` |
| **Partition key** | `idAlert` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "GG0e8400-e29b-41d4-a716-446655440000",
  "eventType": "AlertRaised",
  "aggregateId": "GG0e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Alert",
  "occurredAt": "2026-09-09T15:05:00Z",
  "version": 1,
  "payload": {
    "idAlert": "string — the alert identifier",
    "idHome": "string — the affected home",
    "idDevice": "string — the related device, if any (null for LIMIT alerts)",
    "type": "string — one of: THRESHOLD, CONNECTIVITY, DEVICE, LIMIT",
    "messageKey": "string — translatable message key",
    "consumptionLevelId": "string — the consumption level, if THRESHOLD alert",
    "dateTime": "string — when the alert was raised (ISO 8601)"
  },
  "metadata": {
    "correlationId": "HH0e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The `type` field determines which consumers react: `THRESHOLD` and `CONNECTIVITY` are delivered
> to Notification; `DEVICE` and `LIMIT` may have different handling. The `consumptionLevelId` is
> only present for `THRESHOLD` alerts and references a level owned by Monitoring and Measurements.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Notification | `AlertRaisedListener` delivers the alert via configured channels (email, Web Push) | Yes — uses `eventId` as the idempotency key; delivers each alert once |

---
---
---
### Event: RecommendationCreated

| Field | Value |
|-------|-------|
| **Name** | `RecommendationCreated` |
| **Bounded Context** | Recommendations |
| **Aggregate** | `Recommendation` |
| **Trigger** | A new recommendation is generated from consumption pattern analysis |
| **Consumers** | Notification (delivers the recommendation via email/push) |
| **Channel (topic)** | Kafka topic `recommendations.recommendation-created` |
| **Partition key** | `idRecommendation` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "II0e8400-e29b-41d4-a716-446655440000",
  "eventType": "RecommendationCreated",
  "aggregateId": "II0e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "Recommendation",
  "occurredAt": "2026-09-09T15:10:00Z",
  "version": 1,
  "payload": {
    "idRecommendation": "string — the recommendation identifier",
    "idHome": "string — the affected home",
    "messageKey": "string — translatable message key",
    "consumptionLevelId": "string — the consumption level that triggered the recommendation, if any",
    "patternId": "string — the pattern ID that generated the recommendation",
    "dateTime": "string — when the recommendation was created (ISO 8601)"
  },
  "metadata": {
    "correlationId": "JJ0e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The `consumptionLevelId` and `patternId` are optional; some recommendations may be based on
> general behavioral patterns without a specific consumption level threshold.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Notification | `RecommendationCreatedListener` delivers the recommendation via configured channels (email, Web Push) | Yes — uses `eventId` as the idempotency key; delivers each recommendation once |

---
---
---
### Event: AccountDeleted

| Field | Value |
|-------|-------|
| **Name** | `AccountDeleted` |
| **Bounded Context** | Security |
| **Aggregate** | `User` |
| **Trigger** | A user account is soft-deleted (lifecycle: `ACTIVE` → `DELETED`) |
| **Consumers** | Home (`AccountDeletedListener`), Notification (may notify related parties) |
| **Channel (topic)** | Kafka topic `security.account-deleted` |
| **Partition key** | `idUser` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "KK0e8400-e29b-41d4-a716-446655440000",
  "eventType": "AccountDeleted",
  "aggregateId": "KK0e8400-e29b-41d4-a716-446655440001",
  "aggregateType": "User",
  "occurredAt": "2026-09-09T15:15:00Z",
  "version": 1,
  "payload": {
    "idUser": "string — the user whose account was deleted",
    "deletedEmail": "string — the email address, transformed to `deleted+<id>@deleted.invalid`",
    "reason": "string — optional reason for deletion",
    "permanent": "boolean — whether this is a permanent deletion"
  },
  "metadata": {
    "correlationId": "LL0e8400-e29b-41d4-a716-446655440002",
    "causationId": null
  }
}
```

> The `deletedEmail` follows the pattern `deleted+<id>@deleted.invalid` so that the address can be
> reused in the future without conflicts. The `home` context listens for this event to soft-delete
> related entities (`user_home`, `home`, `device_home`, etc.) within the same transaction.

**What do consumers do with this event?**

| Consuming service | Action | Idempotent? |
|--------------------|--------|-------------|
| Home | Soft-deletes the home and all related entities (`user_home`, `device_home`, thresholds, alerts, recommendations); releases the email address | Yes — uses `eventId` as the idempotency key |
| Notification | May send a final delivery confirmation or cleanup notification | Yes |

---

## Standard fields for all events

All events must include these fields in the envelope:

| Field | Type | Description |
|-------|------|--------------|
| `eventId` | UUID | Unique event ID (for idempotency) |
| `eventType` | string | Event name in PascalCase |
| `aggregateId` | UUID | ID of the aggregate that generated the event |
| `aggregateType` | string | Aggregate type |
| `occurredAt` | ISO 8601 | When the business fact occurred |
| `version` | integer | Schema version (for evolution) |
| `payload` | object | Event data (specific per type) |
| `metadata.correlationId` | UUID | For tracing a transaction across services |
| `metadata.causationId` | UUID | ID of the event or command that caused this event, when one exists |
| `metadata.userId` | UUID | User who initiated the chain, if applicable |

In this project, both catalogued events are triggered by hardware or by a timeout, not by a user
action — `metadata.userId` is expected to be absent from every event currently defined. It stays
in the envelope for future events that a user does initiate (for example, a user manually
resolving an alert).

**Partitioning (Kafka-specific):** the envelope fields above are broker-agnostic, but Kafka only
guarantees ordering *within* a partition, not across a topic. Every event in this catalog is
produced with a **partition key** — see the "Partition key" field on each event above — so that
events about the same `deviceId` are always processed in order by the same consumer instance.
The partition key is not part of the JSON payload; it is passed to the producer separately (the
Kafka record key), per the client library in use (Spring Kafka on this project's backend).

**Transport:** Domain events in this catalog are **Spring Framework events published in memory**
within the modular monolith. They are not sent over Apache Kafka or any external broker. The
`MeasurementRecorded` and `DeviceConnectivityLost` events are raised by the domain layer and
consumed by other modules within the same Spring application context. This design choice
reflects the modular monolith architecture (Spring Modulith) where in-process event delivery
is sufficient and avoids the complexity of a distributed event bus for the current deployment.

MQTT is used only for device-to-backend telemetry (device → backend), never for service-to-service
events. The transport mechanism is documented in `05-architecture/decisions/records/ADR-002-domain-event-transport.md`,
which previously evaluated Kafka as an alternative but the implementation chose in-memory Spring
events for the modular monolith.

---

## Event flow: Automatic measurement ingestion and reaction

```
Device (ESP32 + PZEM-004T V3)
  │
  │  PublishReading (MQTT)
  ▼
[Aggregate: Measurement]  (Monitoring and Measurements)
  │
  │  MeasurementRecorded (event)
  ├────────────────────────────────────────────┐
  │                                             ▼
  │                                  [Service: Alerts]
  │                                  ConsumptionClassificationService classifies the reading
  │                                  If HIGH or CRITICAL → raises a THRESHOLD Alert
  │
  │  MeasurementRecorded (event)
  └────────────────────────────────────────────┐
                                                 ▼
                                      [Service: Recommendations]
                                      Updates the home's consumption pattern
                                      May generate a Recommendation
```

## Event flow: Device connectivity loss

```
[Devices] — timeout reached since the device's last reading
  │
  │  DeviceConnectivityLost (event)
  ▼
[Service: Alerts]
  Raises a CONNECTIVITY Alert for the affected home and device
```

---

## Schema evolution strategy

Events are contracts. Changing them in an incompatible way breaks consumers.

### What is a compatible change (does not break)?

```
✓ Add a new optional field to the payload
✓ Add a new event type
✓ Change a required field → optional
```

### What is an incompatible change (breaks)?

```
✗ Remove a field from the payload
✗ Change the type of a field (string → number)
✗ Change an optional field → required
✗ Change the event name
```

### How to evolve a schema without breaking consumers

**Strategy: Version the event**

```
Step 1: Publish EventV2 (new type with incompatible changes)
Step 2: Publish both EventV1 and EventV2 during the migration period
Step 3: Migrate consumers to V2 one by one
Step 4: Deprecate EventV1 (announce in advance)
Step 5: Stop publishing EventV1
```

---

## Event summary table

| Event | Origin context | Topic | Consumers | Version |
|-------|-----------------|-------|-----------|---------|
| `MeasurementRecorded` | Monitoring and Measurements | `monitoring.measurement-recorded` | Alerts, Recommendations | v1 |
| `DeviceConnectivityLost` | Devices | `devices.device-connectivity-lost` | Alerts | v1 |
| `DeviceConnectivityRestored` | Devices | `devices.device-connectivity-restored` | Alerts (may auto-resolve) | v1 |
| `DeviceLinked` | Devices | `devices.device-linked` | Home Management, Alerts | v1 |
| `DeviceUnlinked` | Devices / Home Management | `devices.device-unlinked` | Home Management, Alerts | v1 |
| `DeviceRegistered` | Devices | (internal registry) | None (internal) | v1 |
| `AlertRaised` | Alerts | `alerts.alert-raised` | Notification | v1 |
| `RecommendationCreated` | Recommendations | `recommendations.recommendation-created` | Notification | v1 |
| `AccountDeleted` | Security | `security.account-deleted` | Home, Notification | v1 |

---

## Policies — Reactions to events

A **Policy** describes what happens automatically when an event arrives: "whenever X occurs, do Y".

| Trigger event | Policy | Emitted command | Service |
|----------------|--------|-------------------|---------|
| `MeasurementRecorded` | Whenever the classified consumption level is `HIGH` or `CRITICAL`, raise an alert | `RaiseThresholdAlert` | Alerts |
| `MeasurementRecorded` | Whenever the reading meaningfully changes the home's consumption pattern, generate a suggestion | `GenerateRecommendation` | Recommendations |
| `DeviceConnectivityLost` | Whenever a device stops reporting past the timeout, raise an alert | `RaiseConnectivityAlert` | Alerts |
| `DeviceConnectivityRestored` | Whenever a device resumes reporting, may auto-resolve a `CONNECTIVITY` alert | — | Alerts |
| `DeviceLinked` | Whenever a device is linked to a home, record the `DeviceHome` relationship | — | Home Management |
| `DeviceUnlinked` | Whenever a device is unlinked from a home, remove the `DeviceHome` relationship | — | Home Management |
| `AlertRaised` | Whenever an alert becomes active, deliver it via configured channels | — | Notification |
| `RecommendationCreated` | Whenever a recommendation is generated, deliver it via configured channels | — | Notification |
| `AccountDeleted` | Whenever an account is soft-deleted, cascade the deletion to related entities | — | Home, Notification |

---

## Resilience patterns for events

### At-least-once delivery + Idempotency

The broker guarantees the event is delivered **at least once** but it may be delivered more than
once (in case of retries). Consumers must be **idempotent**.

With Kafka, "at-least-once" comes from committing the consumer group's offset only *after* the
event is fully processed (not before). If the consumer crashes between processing and committing,
the same message is redelivered on restart — the `processedEventRepository.exists(eventId)` check
below is what makes that redelivery safe, not the broker itself.

```java
// Idempotent consumer — stores the processed eventId before acting on the event
public void handleMeasurementRecorded(MeasurementRecordedEvent event) {
    if (processedEventRepository.exists(event.getEventId())) {
        log.info("Event {} already processed, ignoring", event.getEventId());
        return;
    }

    alertEvaluationService.evaluate(event.getPayload());
    processedEventRepository.markProcessed(event.getEventId());
}
```

### Dead Letter Queue (DLQ)

When an event fails after N retries, it goes to the DLQ.

In Kafka, the DLQ is itself a topic, following the `<original-topic>.DLQ` naming pattern (e.g.
`monitoring.measurement-recorded.DLQ`), not a separate queue technology. A message published to
it keeps its original headers plus the failure reason, so it can be replayed once fixed.

| Configuration | Recommended value |
|----------------|--------------------|
| Retries before DLQ | 3-5 |
| Backoff | Exponential (1s → 2s → 4s → 8s) |
| DLQ retention | 7 days |
| Alert | When DLQ has > 0 messages |

> See DLQ runbook in `09-microservices/services/XX-service/runbook.md`