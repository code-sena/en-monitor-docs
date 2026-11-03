# Runbook — Home Service

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra`, with `emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }`.
> While this service is down no home route answers, and the device, measurement, alert,
> recommendation and notification services cannot check who belongs to a home.

**Service:** `en-monitor-home-service` (container `home`)
**Repository:** https://github.com/angelgarcia0/en-monitor-home-service
**Version:** `main` @ `6a4e396` (28-Oct-2026)
**Last updated:** 2026-11-03

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8086, inside the Docker network only |
| Production URL | `https://energy-monitor.tech/api/v1/homes`, `/api/v1/home-types` (through the gateway) |
| Health | `/actuator/health` |
| Switches | `ACCOUNT_DELETED_CONSUMER_ENABLED`, `HOME_OUTBOX_RELAY_ENABLED` (both `true`) |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |

---

## 2. Verify the service is healthy

```bash
emc ps home                                                          # (healthy)
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/home-types   # 401: route reaches the home service
```

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| Device, consumption, alert or recommendation routes answer 503 | They ask this service for access and it did not answer | `emc ps home`; check `INTERNAL_API_TOKEN` is the same in both |
| A deleted home keeps its devices | `home.deleted` waiting in `home_outbox` (broker down) or in the device service's `.dlq` | Check RabbitMQ through an SSH tunnel; the outbox publishes in order when it returns |
| Member names show as unknown | The security service did not answer `/internal/users/profiles` | `emc ps security` |

---

## 4. Rollback

`infra/deploy/deploy.sh home` (GitHub Actions, after approval of the `production` environment)
backs up MySQL, pulls `main`, rebuilds only `home` and waits for every Java service to be healthy;
if that fails it rebuilds the previous commit and exits with an error. A code rollback does not roll
back an applied Liquibase changeset.

---

## 5. Database migrations

Liquibase runs on startup and applies only `db/changelog/home/` (`home-001` … `home-007`). Never
rename or move a changelog file.

---

## 6. Alerts

No automated alerting is configured; failures show up in `emc ps`, the deploy job and user
reports.
