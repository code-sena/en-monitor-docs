# Events — Measurement Service

> Published on the RabbitMQ topic exchange `energy-monitor.measurement` through the transactional
> outbox `measurement_outbox` (`ADR-008`).

---

## Events this service PUBLISHES

| Routing key | Payload | When it is emitted | Consumer |
|---|---|---|---|
| `measurement.recorded` | `{idMeasurement, deviceId, dateTime, voltage, current, activePower, storedEnergy}` | A reading is stored | Alert service (queue `alert-service.measurement.recorded`): evaluates the threshold, limit, unusual-consumption and "device linked" rules |

- The message is written in the transaction of the reading (`TransactionalRegisterMeasurement`):
  one never commits without the other.
- A relay publishes up to 50 pending messages per run, every second, oldest first, with
  correlated confirms and mandatory routing; until the alert service declares its queue they stay
  pending.

---

## Events this service CONSUMES

| Event | Published by | Queue / exchange | What this service does |
|---|---|---|---|
| `device.linked`, `device.unlinked` | Device service | `measurement-service.device.events` on `energy-monitor.device` | Drops the cached device and the cached devices of its home, so a link or an unlink shows at once |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers drop duplicates by message id; forgetting a cache entry twice only costs one more request |
| Ordering | Oldest first; the relay stops at the first failure | Order kept |
| Exactly-once | No | — |

---

## Error handling

- **Dead Letter Queue:** `measurement-service.device.events.dlq` on `measurement-service.dlx`; the
  alert service's `alert-service.measurement.recorded` queue has its own `.dlq`.
- **Retries:** three attempts, 1, 2 and 4 s apart, then the dead-letter queue. A message without a
  `deviceId`, or of another type, is dropped.
- **Broker down:** readings keep being stored and their messages wait in the outbox; without the
  device events the device cache still expires after 30 s; health is DOWN.
- **Tests:** the relay, the purge and the consumers are off in the test profile;
  `RabbitMeasurementOutboxPublisherBrokerTest` and `DeviceEventsBrokerTest` run against the
  RabbitMQ of infra when reachable.
