# Notification Service — `en-monitor-notification-service`

> The Notification bounded context, extracted from the `notification` module of the former
> `energy-monitor-backend` and in production since 28-Oct-2026. It is the only service that sends
> e-mail and push messages.

---

## Responsibility

Delivers alerts, recommendations and activity to the members of a home by e-mail, Web Push and
mobile push according to each member's preferences, mails the security service's code messages,
and keeps the in-app inbox, the administrator notices and the delivery log.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 06 |
| Local port | 8087 |
| Repository | https://github.com/angelgarcia0/en-monitor-notification-service |
| Technology | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP, JavaMail, Web Push (VAPID), Expo push |
| DB engine | MySQL 8.4 — shared schema `energy_monitor`, notification tables only (`ADR-003`) |
| Architecture | Hexagonal: `api`, `domain`, `application`, `adapter`, `infrastructure` (package `com.energymonitor.notification`) |
| Communicates with | security, home and device services (internal API), RabbitMQ, an SMTP server, browser push services and Expo |
| Consumed by | Web and mobile apps (through the gateway); RabbitMQ events of the other services |
| Status | 🟢 In production |

---

## Responsibilities (what this service DOES)

- **Public endpoints** (through the gateway):

  | Path | What it does |
  |---|---|
  | `GET /api/v1/notifications/activity`, `PUT …/{id}/read`, `PUT …/read`, `DELETE …/{id}` | Inbox of activity notifications (account, homes and devices) |
  | `GET`, `PUT /api/v1/notifications/preferences` | E-mail and push preferences of the user |
  | `GET /api/v1/notifications/push/public-key` | VAPID public key (404 when Web Push is not configured) |
  | `POST /api/v1/notifications/push/subscriptions`, `…/remove` | Browser push subscriptions |
  | `POST /api/v1/notifications/push/devices`, `…/remove`; `GET …/devices/available` | Mobile app installations (Expo) and whether app push is configured |
  | `GET /api/v1/admin/notifications`, `PUT …/{id}/read`, `DELETE …/{id}` | In-app notices of the administrators (`USER.READ`) |

- **Sends the code mails** requested by the security service (`mail.email-verification`,
  `mail.password-reset`, `mail.account-deletion`); it is the only service that sends e-mail.
- **Delivers** each `alert.raised` and `recommendation.created` to the members of the home through
  the channels they enabled, and records the status (`PENDING`, `SENT`, `FAILED`); a sweeper takes
  care of abandoned deliveries.
- **Records activity** (welcome, homes and devices) with the text key and the names at that
  moment (`message_params`), so the client translates the text and the history never changes.
- **Records administrator notices** when an account registers or another administrator changes the
  status or roles of an account.
- **Cleans up** preferences and push devices of deleted accounts (`account.deleted`).
- The alert of the unusual-consumption rule arrives with `messageKey = alert.anomaly.device` and
  is delivered like any other alert; the notification text for it is built by this service.

### Who it asks

| Service | Call | For |
|---|---|---|
| Home service | `GET /internal/homes/{homeId}` | Name and members of the home |
| Security service | `POST /internal/users/profiles`, `GET /internal/users/{id}/permissions`, `GET /internal/administrators` | Names and e-mails, `USER.READ`, administrators |
| Device service | `GET /internal/devices/{id}` | Name of the device in a message |

## Out of scope (what it does NOT do)

- **Does not decide when something is an alert or a recommendation** — it delivers what the alert
  and recommendation services announce.
- **Does not own accounts or homes** — it asks their owners.

---

## How to run it locally

```bash
# MySQL and RabbitMQ from energy-monitor-infra (development mode, see 10-devops/local-setup.md)
cd backend/en-monitor-notification-service
cp .env.example .env
cp src/main/resources/application-local.yaml.example src/main/resources/application-local.yaml
# set MYSQL_PASSWORD, SECURITY_JWT_PUBLIC_KEY_PATH, INTERNAL_API_TOKEN, RABBITMQ_*; MAIL_*,
# NOTIFICATION_VAPID_* and EXPO_ACCESS_TOKEN only if you want real deliveries
./mvnw spring-boot:run

# Verify
curl http://localhost:8087/actuator/health
```

The consumers of the security and alert events are **off by default**
(`SECURITY_EVENTS_CONSUMER_ENABLED=false`, `ALERT_EVENTS_CONSUMER_ENABLED=false`) so a run from the
IDE never competes with production; the infra stack turns them on.

Tests: `SPRING_DATASOURCE_PASSWORD=... ./mvnw test` (235 tests green on 28-Oct-2026).

---

## Related documents

- [data-model.md](./data-model.md) — Notification tables
- [events.md](./events.md) — Events it consumes
- [runbook.md](./runbook.md) — Operation in production
- API contract → `07-api/contracts/openapi/notification-service.yaml`
- Code: `README.md` in the repository
