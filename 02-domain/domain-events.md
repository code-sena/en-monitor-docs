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
| **Consumers** | Alerts (threshold, limit and "device linked" alerts). Recommendations do not consume it: they read hourly energy from the measurement service's internal API |
| **Channel (topic)** | RabbitMQ `energy-monitor.measurement` / `measurement.recorded` (outbox `measurement_outbox`), consumed by the alert service |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Channel (topic)** | RabbitMQ `energy-monitor.device` / `device.connectivity-lost` (outbox `device_outbox`), consumed by the alert service |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Channel (topic)** | RabbitMQ `energy-monitor.device` / `device.connectivity-restored` (outbox `device_outbox`), consumed by the alert service |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Consumers** Alert service (linked-device alert), measurement service (drops its device cache), notification service (activity). `DeviceHome` is owned by the device service itself |
| **Channel (topic)** | RabbitMQ `energy-monitor.device` / `device.linked` (outbox `device_outbox`) |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Consumers** Measurement service (drops its device cache), notification service (activity) |
| **Channel (topic)** | RabbitMQ `energy-monitor.device` / `device.unlinked` (outbox `device_outbox`) |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Channel (topic)** | Not published outside the device service (no RabbitMQ routing key exists for it) |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Channel (topic)** | RabbitMQ `energy-monitor.alert` / `alert.raised` (outbox `alert_outbox`) |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Channel (topic)** | RabbitMQ `energy-monitor.recommendation` / `recommendation.created` (outbox `recommendation_outbox`) |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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
| **Consumers** | Home Management (home service): drops the memberships and the homes the account alone owned; Notification: cleans up preferences and push devices |
| **Channel (topic)** | RabbitMQ `energy-monitor.security` / `account.deleted` (outbox `security_outbox`) |
| **Ordering** | Per producer: the outbox relay publishes oldest first and stops at the first failure (no partitions) |
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

**Ordering:** every producer writes its events to its own outbox table in the same transaction
as the change, and a relay publishes them to RabbitMQ oldest first, stopping at the first failure,
so order is kept per producer (`ADR-008`). The production payloads carry the message id (the
outbox `id_message`) and the business fields; the `metadata.*` envelope fields above are the
target format, not yet used on RabbitMQ.

**Transport (28-Oct-2026):**

- **Between services** (all seven contexts are microservices): RabbitMQ 4.1 topic exchanges
  `energy-monitor.security`, `energy-monitor.home`, `energy-monitor.device`,
  `energy-monitor.measurement`, `energy-monitor.alert` and `energy-monitor.recommendation`, each
  fed by a transactional outbox
  (`05-architecture/decisions/records/ADR-008-rabbitmq-transactional-outbox.md`, which
  supersedes the Kafka decision of `ADR-002`). Besides the events in this catalog, the security
  service publishes `account.registered`, `account.administered`, `account.first-signed-in`,
  `mail.email-verification`, `mail.password-reset` and `mail.account-deletion`, and the home
  service publishes `home.created`, `home.member-joined`, `home.member-left`,
  `home.member-removed`, `home.thresholds-updated`, `home.ownership-transferred` and
  `home.deleted`. The notification service only consumes.
- **In process:** none between contexts. The Spring in-memory events of the former monolith
  (`AlertRaised`, `RecommendationCreated`, `HomeOwnershipTransferred`, `MeasurementRecorded`,
  `DeviceConnectivityLost/Restored`) became RabbitMQ messages when the monolith was retired.

MQTT is used only for device traffic (telemetry and status from the modules, commands to them),
never for service-to-service events.

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

| Event | Origin context | Channel (24-Oct-2026) | Consumers | Version |
|-------|-----------------|-------|-----------|---------|
| `MeasurementRecorded` | Monitoring and Measurements | RabbitMQ `measurement.recorded` | Alerts | v1 |
| `DeviceConnectivityLost` | Devices | RabbitMQ `device.connectivity-lost` | Alerts | v1 |
| `DeviceConnectivityRestored` | Devices | RabbitMQ `device.connectivity-restored` | Alerts | v1 |
| `DeviceLinked` | Devices | RabbitMQ `device.linked` | Backend (device cache), measurement service (cache) | v1 |
| `DeviceUnlinked` | Devices | RabbitMQ `device.unlinked` | Backend (device cache), measurement service (cache) | v1 |
| `DeviceRegistered` | Devices | Not published outside the device service | None | v1 |
| `AlertRaised` | Alerts | RabbitMQ `alert.raised` | Notification | v1 |
| `RecommendationCreated` | Recommendations | RabbitMQ `recommendation.created` | Notification | v1 |
| `AccountDeleted` | Security | RabbitMQ `account.deleted` | Home Management | v1 |
| `home.deleted` | Home Management | RabbitMQ `energy-monitor.home` (outbox `home_outbox`) | Devices: unlinks the home's modules and sends them `unlink` | — |

---

## Policies — Reactions to events

A **Policy** describes what happens automatically when an event arrives: "whenever X occurs, do Y".

| Trigger event | Policy | Emitted command | Service |
|----------------|--------|-------------------|---------|
| `MeasurementRecorded` | Whenever the classified consumption level is `HIGH` or `CRITICAL`, raise an alert | `RaiseThresholdAlert` | Alerts |
| — | Recommendations are generated from the hourly energy read from the measurement service, not from this event | `GenerateRecommendation` | Recommendations |
| `DeviceConnectivityLost` | Whenever a device stops reporting past the timeout, raise an alert | `RaiseConnectivityAlert` | Alerts |
| `DeviceConnectivityRestored` | Whenever a device resumes reporting, may auto-resolve a `CONNECTIVITY` alert | — | Alerts |
| `DeviceLinked` | Whenever a device is linked to a home, drop the cached copy of the device | — | Backend, measurement service |
| `DeviceUnlinked` | Whenever a device is unlinked from a home, drop the cached copy of the device | — | Backend, measurement service |
| `AlertRaised` | Whenever an alert becomes active, deliver it via configured channels | — | Notification |
| `RecommendationCreated` | Whenever a recommendation is generated, deliver it via configured channels | — | Notification |
| `AccountDeleted` | Whenever an account is soft-deleted, drop its memberships and the homes it alone owned | — | Home Management |
| `home.deleted` | Whenever a home is deleted, unlink its modules, revoke their API keys and send them `unlink` | — | Devices |

---

## Resilience patterns for events

### At-least-once delivery + Idempotency

The broker guarantees the event is delivered **at least once** but it may be delivered more than
once (in case of retries). Consumers must be **idempotent**.

In this project "at-least-once" comes from the outbox relay: a row is marked published only after
RabbitMQ confirms it and a queue took it, so a crash in between publishes it again. Every message
carries its id, and every consumer tolerates a second delivery (for example, a repeated
`home.deleted` finds no module left to unlink). The snippet below is the generic form of an
idempotent consumer.

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

In RabbitMQ every consumer queue is durable and has its own dead-letter queue (`<queue>.dlq`, on
a `<service>.dlx` exchange), for example `device-service.device.home-deleted.dlq` or
`alert-service.measurement.recorded.dlq`.

| Configuration | Value in production |
|----------------|--------------------|
| Attempts before the DLQ | 3 |
| Backoff | 1 s, 2 s, 4 s (device and measurement services) |
| DLQ retention | Not documented in the service repositories |
| Alert | None automated; checked in the RabbitMQ console |

> See DLQ runbook in `09-microservices/services/XX-service/runbook.md`