# Events — Home Service

> Published on the RabbitMQ topic exchange `energy-monitor.home` through the transactional outbox
> `home_outbox` (`ADR-008`).

---

## Events this service PUBLISHES

| Routing key | When it is emitted | Consumers |
|---|---|---|
| `home.created`, `home.member-joined`, `home.member-left`, `home.member-removed`, `home.thresholds-updated`, `home.ownership-transferred` | The matching change is committed; the payload carries the name of the home and its members | Notification service (`notification-service.notification.activity`): activity notifications in the inbox and through the chosen channels |
| `home.deleted` | A home is deleted (by its owner, or because the only owner's account was deleted) | Device service (`device-service.device.home-deleted`): unlinks the home's modules, revokes their keys and sends them the `unlink` command. Notification service (`…activity`) |

- The message is written in the same transaction as the change: one never commits without the
  other.
- A relay publishes the pending messages every second, oldest first, with confirms and mandatory
  routing; until a queue takes a message it stays pending.

---

## Events this service CONSUMES

| Event | Published by | Queue / exchange | What this service does |
|---|---|---|---|
| `account.deleted` | Security service | `home-service.home.account-deleted` on `energy-monitor.security` | Removes the account's memberships and deletes the homes it solely owned (switch `ACCOUNT_DELETED_CONSUMER_ENABLED`) |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers drop duplicates by message id |
| Ordering | Oldest first; the relay stops at the first failure | Order kept |
| Exactly-once | No | — |

---

## Error handling

- **Dead Letter Queue:** `home-service.home.account-deleted.dlq`.
- **Retries:** three attempts, 1, 2 and 4 s apart, then the dead-letter queue.
- **Broker down:** homes keep working and their messages wait in `home_outbox`; health is DOWN.
