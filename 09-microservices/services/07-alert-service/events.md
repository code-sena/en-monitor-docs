# Events — Alert Service

> Published on the RabbitMQ topic exchange `energy-monitor.alert` through the transactional outbox
> `alert_outbox` (`ADR-008`).

---

## Events this service PUBLISHES

| Routing key | When it is emitted | Consumer |
|---|---|---|
| `alert.raised` | A new alert is stored (any type, including unusual consumption) | Notification service (`notification-service.notification.alert-raised`): delivery by e-mail and push to the members of the home |

The message is written in the transaction of the alert. A relay publishes the pending messages
every second, oldest first; until the notification service declares its queue they stay pending
(switch `ALERT_OUTBOX_RELAY_ENABLED`).

---

## Events this service CONSUMES

| Event | Published by | Queue / exchange | What this service does |
|---|---|---|---|
| `measurement.recorded` | Measurement service | `alert-service.measurement.recorded` on `energy-monitor.measurement` | Evaluates the threshold, limit, unusual-consumption and linked-device rules |
| `device.linked`, `device.unlinked`, `device.connectivity-lost`, `device.connectivity-restored` | Device service | `alert-service.device.events` on `energy-monitor.device` | Raises and resolves `CONNECTIVITY` alerts; raises the linked-device alert |

Switch: `ALERT_CONSUMERS_ENABLED`.

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | A repeated reading cannot raise a second alert while one is active |
| Ordering | Oldest first in the outbox | Order kept |
| Exactly-once | No | — |

---

## Error handling

- **Dead Letter Queue:** `alert-service.measurement.recorded.dlq` and `alert-service.device.events.dlq`.
- **Retries:** three attempts, 1, 2 and 4 s apart, then the dead-letter queue.
- **Broker down:** alerts already stored stay; new readings are not evaluated; health is DOWN.
