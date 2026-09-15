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

**Transport:** domain events in this catalog travel over **Apache Kafka**, kept separate from the
MQTT channel used for device-to-backend telemetry (`01-context/scope.md`, feature #11) — MQTT is
never used for service-to-service events. See `ADR-002-domain-event-transport.md` in
`05-architecture/decisions/records/` for the alternatives evaluated and why Kafka was chosen.

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

---

## Policies — Reactions to events

A **Policy** describes what happens automatically when an event arrives: "whenever X occurs, do Y".

| Trigger event | Policy | Emitted command | Service |
|----------------|--------|-------------------|---------|
| `MeasurementRecorded` | Whenever the classified consumption level is `HIGH` or `CRITICAL`, raise an alert | `RaiseThresholdAlert` | Alerts |
| `MeasurementRecorded` | Whenever the reading meaningfully changes the home's consumption pattern, generate a suggestion | `GenerateRecommendation` | Recommendations |
| `DeviceConnectivityLost` | Whenever a device stops reporting past the timeout, raise an alert | `RaiseConnectivityAlert` | Alerts |

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