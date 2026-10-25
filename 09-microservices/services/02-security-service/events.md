# Events — Security Service

> Everything other services must hear from the security service goes to the RabbitMQ topic
> exchange `energy-monitor.security`, with the message type as routing key, through the
> transactional outbox `security_outbox` (`ADR-008`). The only consumer today is
> `energy-monitor-backend`.

---

## Events this service PUBLISHES

| Routing key | Body (JSON) | Consumer in `energy-monitor-backend` |
|---|---|---|
| `account.registered` | `userId`, `email`, `occurredAt` | `notification`: in-app notice to the administrators |
| `account.administered` | `actorId`, `userId`, `action`, `detail`, `occurredAt` | `notification`: in-app notice to the other administrators |
| `account.deleted` | `userId`, `deletedAt` | `home`: drops the memberships and the homes the account alone owned |
| `mail.email-verification` | `userId`, `recipient`, `code`, `validUntil` | `notification`: mails the code |
| `mail.password-reset` | `userId`, `recipient`, `idResetToken`, `code`, `validUntil` | `notification`: mails the code |

Every message carries its id (`id_message` of the outbox row) as the broker message id, so
consumers drop duplicates.

---

## Events this service CONSUMES

**None.**

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers must tolerate duplicates (message id) |
| Ordering | Oldest first; the relay stops at the first failure | Order kept per producer |
| Exactly-once | No | — |

- The outbox row is written in the same transaction as the change; a relay (`OutboxRelay`)
  publishes the pending rows every second. A row counts as published only when RabbitMQ confirms
  it and a queue took it.
- Mail requests: the payload is erased once confirmed, the message expires with the code (AMQP
  expiration), and the relay discards it unsent once the code is no longer valid.
- The relay assumes one instance of the service; a second one would keep delivery correct but
  could reorder messages.

---

## Error handling

- **Dead Letter Queue:** each consumer declares its own durable queue with a dead-letter queue.
- **Broker down:** messages wait in `security_outbox`; `/actuator/health` reports DOWN.
- **Tests:** the outbox relay is off in the `test` profile and messages are stored already
  discarded; `RabbitOutboxPublisherBrokerTest` runs against a real RabbitMQ when one answers.
