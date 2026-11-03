# Runbook — Notification Service

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra`, with `emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }`.
> While this service is down nobody receives verification codes, alert e-mails or push messages.

**Service:** `en-monitor-notification-service` (container `notification`)
**Repository:** https://github.com/angelgarcia0/en-monitor-notification-service
**Version:** `main` @ `ecdbc0a` (28-Oct-2026)
**Last updated:** 2026-11-03

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8087, inside the Docker network only |
| Production URL | `https://energy-monitor.tech/api/v1/notifications/**`, `/api/v1/admin/notifications/**` (through the gateway) |
| Health | `/actuator/health` |
| Switches | `SECURITY_EVENTS_CONSUMER_ENABLED`, `ALERT_EVENTS_CONSUMER_ENABLED`, `ACTIVITY_CONSUMER_ENABLED` (all `true` in the infra stack) |
| Secrets (never in the repository) | `MAIL_*`, `NOTIFICATION_VAPID_PRIVATE_KEY`, `EXPO_ACCESS_TOKEN`, `INTERNAL_API_TOKEN`, in `.env.prod` |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |

---

## 2. Verify the service is healthy

```bash
emc ps notification        # (healthy)
emc logs notification | tail -50
```

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| Users do not receive the verification code | SMTP credentials wrong or the mail provider blocked the login; message waiting in `notification-service.notification.mail` | `emc logs notification`; check `MAIL_*` |
| No push in the browser | `NOTIFICATION_VAPID_*` empty (`GET /api/v1/notifications/push/public-key` answers 404), or the user's browser uses Brave without "Use Google services for push messaging" | Check the keys; ask the user to enable that setting (`brave://settings/privacy`) |
| No push in the app | `EXPO_ACCESS_TOKEN` missing (`GET /api/v1/notifications/push/devices/available` answers 404) | Set the token |
| Alerts are raised but not delivered | `alert.raised` waiting in `alert_outbox`, or this service's queue missing | Deploy this service before the alert service; check RabbitMQ through an SSH tunnel |

---

## 4. Rollback

`infra/deploy/deploy.sh notification` (GitHub Actions, after approval of the `production`
environment) backs up MySQL, pulls `main`, rebuilds only `notification` and waits for every Java
service to be healthy; if that fails it rebuilds the previous commit and exits with an error. A code
rollback does not roll back an applied Liquibase changeset.

---

## 5. Database migrations

Liquibase runs on startup and applies only `db/changelog/notification/` (`notification-001` …
`notification-007`). Never rename or move a changelog file.

---

## 6. Alerts

No automated alerting is configured; failures show up in `emc ps`, the deploy job, the
`delivery_status = FAILED` rows and user reports.
