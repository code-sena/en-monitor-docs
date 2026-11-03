# Events — Recommendation Service

> Published on the RabbitMQ topic exchange `energy-monitor.recommendation` through the
> transactional outbox `recommendation_outbox` (`ADR-008`).

---

## Events this service PUBLISHES

| Routing key | When it is emitted | Consumer |
|---|---|---|
| `recommendation.created` | A rule matches and a new recommendation is stored | Notification service (`notification-service.notification.recommendation-created`): delivery to the members of the home |

The message is written in the transaction of the recommendation. A relay publishes the pending
messages every second, oldest first, and keeps them pending until a queue takes them (switch
`RECOMMENDATION_OUTBOX_RELAY_ENABLED`).

---

## Events this service CONSUMES

None. It works by polling: every hour it asks the home, device and measurement services through
their internal APIs.

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | A repeated message is dropped by its message id |
| Ordering | Oldest first | Order kept |
| Exactly-once | No | — |

---

## Error handling

- **Broker down:** recommendations are still stored and their messages wait in the outbox; health
  is DOWN.
- **A service it asks is down:** that evaluation is skipped for the affected home and repeats at
  the next hour.
