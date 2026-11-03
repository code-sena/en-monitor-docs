# Events — Notification Service

> This service **publishes no events**. It consumes the events of the other services from four
> durable queues, each declared at start-up, with its `.dlq` (`ADR-008`).

---

## Events this service CONSUMES

| Event | Published by | Queue | What this service does |
|---|---|---|---|
| `account.registered`, `account.administered`, `account.deleted` | Security service | `notification-service.notification.account-events` | In-app notices for administrators; clean-up of the preferences and push devices of deleted accounts |
| `mail.email-verification`, `mail.password-reset`, `mail.account-deletion` | Security service (they expire with `expires_at`) | `notification-service.notification.mail` | Sends the code e-mail |
| `account.first-signed-in`; `home.created`, `home.member-joined`, `home.member-left`, `home.member-removed`, `home.thresholds-updated`, `home.ownership-transferred`, `home.deleted`; `device.linked`, `device.unlinked` | Security, home and device services | `notification-service.notification.activity` | Activity notifications (welcome and changes of homes and devices) in the inbox and through the chosen channels |
| `alert.raised` | Alert service | `notification-service.notification.alert-raised` | Delivers the alert to the members of the home by e-mail and push |
| `recommendation.created` | Recommendation service | `notification-service.notification.recommendation-created` | Delivers the recommendation to the members of the home |

Switches: `SECURITY_EVENTS_CONSUMER_ENABLED`, `ALERT_EVENTS_CONSUMER_ENABLED` and
`ACTIVITY_CONSUMER_ENABLED`. Each queue must exist **before** its producer is deployed, because an
outbox stops at a message no queue takes.

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | A repeated message is dropped by its message id |
| Ordering | Per queue | — |
| Exactly-once | No | — |

---

## Error handling

- **Dead Letter Queue:** each queue has its `.dlq`.
- **Retries:** three attempts, 1, 2 and 4 s apart, then the dead-letter queue.
- **Delivery failures** (SMTP, browser push, Expo) do not lose the notification: it stays in the
  inbox with `delivery_status = FAILED` and the reason; a sweeper closes abandoned `PENDING` rows.
