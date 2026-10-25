# Events — Device Service

> Published on the RabbitMQ topic exchange `energy-monitor.device` through the transactional
> outbox `device_outbox` (`ADR-008`).

---

## Events this service PUBLISHES

| Routing key | Payload | When it is emitted | Consumers |
|---|---|---|---|
| `device.linked` | `{deviceId, homeId, deviceName, linkedBy, linkedAt}` | A module is linked, the first time or again | Backend (device cache), measurement service (cache) |
| `device.unlinked` | `{deviceId, homeId, unlinkedAt}` | A module is unlinked (also for every module of a deleted home) | Backend (cache), measurement service (cache) |
| `device.connectivity-lost` | `{deviceId, homeId, occurredAt}` | An online device goes offline (status, last will or inactivity) | Backend: CONNECTIVITY alert |
| `device.connectivity-restored` | `{deviceId, homeId, occurredAt}` | An offline device reports again | Backend: CONNECTIVITY alerts |

`homeId` is the home the device was linked to when the connectivity changed, or null. In the
backend, the `device` proxy package republishes the connectivity events in memory as
`DeviceConnectivityLost` / `DeviceConnectivityRestored` for the `alert` module (queue
`energy-monitor-backend.device.events`).

---

## Events this service CONSUMES

| Event | Published by | Queue / exchange | What this service does |
|---|---|---|---|
| `home.deleted` (`{homeId, deletedBy, deletedAt}`) | `energy-monitor-backend` (`home_outbox`) | `device-service.device.home-deleted` on `energy-monitor.home` | Unlinks every module of the home, revokes their API keys, sends each the MQTT `unlink` command after commit and announces `device.unlinked` |

Declaring this queue at startup is what lets the backend publish `home.deleted`: until it exists,
the message waits in `home_outbox` (mandatory routing).

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers must tolerate duplicates; a second `home.deleted` finds no module to unlink |
| Ordering | Oldest first; the relay stops at the first failure | Order kept |
| Exactly-once | No | — |

- The message is written in the transaction of the change; the connectivity use case runs one
  transaction per call (`TransactionalDeviceConnectivity`), so the status row and its message
  commit together.
- The relay publishes every second with correlated confirms and mandatory routing.

---

## Error handling

- **Dead Letter Queue:** `device-service.device.home-deleted.dlq` on `device-service.dlx`.
- **Retries:** three attempts, 1, 2 and 4 s apart, then the dead-letter queue. A message without a
  `homeId` is dropped.
- **Tests:** the test profile turns the relay off and does not start the consumers;
  `RabbitDeviceOutboxPublisherBrokerTest` runs against the RabbitMQ of infra when reachable.
