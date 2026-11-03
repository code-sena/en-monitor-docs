# Runbook — Alert Service

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra`, with `emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }`.
> While this service is down no alert is raised, and the alert inbox answers 503.

**Service:** `en-monitor-alert-service` (container `alert`)
**Repository:** https://github.com/angelgarcia0/en-monitor-alert-service
**Version:** `main` @ `0e65430` (28-Oct-2026)
**Last updated:** 2026-11-03

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8088, inside the Docker network only |
| Production URL | `https://energy-monitor.tech/api/v1/alerts` (through the gateway) |
| Health | `/actuator/health` |
| Switches | `ALERT_CONSUMERS_ENABLED`, `ALERT_OUTBOX_RELAY_ENABLED` (both `true`); `ALERT_LIMIT_ZONE` (`America/Bogota`) |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |

---

## 2. Verify the service is healthy

```bash
emc ps alert                                                          # (healthy)
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/alerts   # 401: route reaches the alert service
```

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| Readings arrive but no alert is raised | `alert-service.measurement.recorded` in its `.dlq`, or `measurement.recorded` waiting in `measurement_outbox` | Check RabbitMQ through an SSH tunnel; check `emc logs alert` |
| No unusual-consumption alert for an appliance | Less than 24 h of history, a 7-day mean below 5 W, a power below 200 W or below 2.5 times the usual one — the rule is working as designed | None; the thresholds are constants (DT-27) |
| Alerts are stored but nobody is notified | `alert.raised` waiting in `alert_outbox` because the notification service's queue is missing | Deploy the notification service first |
| Alert routes answer 503 | The home service did not answer the access check | `emc ps home` |

---

## 4. Rollback

`infra/deploy/deploy.sh alert` (GitHub Actions, after approval of the `production` environment)
backs up MySQL, pulls `main`, rebuilds only `alert` and waits for every Java service to be healthy;
if that fails it rebuilds the previous commit and exits with an error. A code rollback does not roll
back an applied Liquibase changeset.

---

## 5. Database migrations

Liquibase runs on startup and applies only `db/changelog/alert/` (`alert-001` … `alert-004`).
Never rename or move a changelog file.

---

## 6. Alerts

No automated alerting is configured; failures show up in `emc ps`, the deploy job and user
reports.
