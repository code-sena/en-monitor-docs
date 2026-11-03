# Runbook — Recommendation Service

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra`, with `emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }`.
> While this service is down no new recommendation is created, and the inbox answers 503.

**Service:** `en-monitor-recommendation-service` (container `recommendation`)
**Repository:** https://github.com/angelgarcia0/en-monitor-recommendation-service
**Version:** `main` @ `aeeeec5` (28-Oct-2026)
**Last updated:** 2026-11-03

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8089, inside the Docker network only |
| Production URL | `https://energy-monitor.tech/api/v1/recommendations` (through the gateway) |
| Health | `/actuator/health` |
| Switches | `RECOMMENDATION_SCHEDULER_ENABLED`, `RECOMMENDATION_OUTBOX_RELAY_ENABLED` (both `true`); `RECOMMENDATION_EVALUATE_INTERVAL` (`PT1H`), `RECOMMENDATION_INITIAL_DELAY` (`PT2M`), `RECOMMENDATION_ZONE` (`America/Bogota`) |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |

---

## 2. Verify the service is healthy

```bash
emc ps recommendation                                                          # (healthy)
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/recommendations   # 401: route reaches the service
```

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| No recommendations after a deploy | The first evaluation runs `RECOMMENDATION_INITIAL_DELAY` after start and then hourly; a home needs readings in the last week | Wait one evaluation; check `emc logs recommendation` |
| A home never gets a recommendation | No rule matches, or the same advice was already given in the last 7 days | None; working as designed |
| Recommendations stored but nobody is notified | `recommendation.created` waiting in `recommendation_outbox` because the notification service's queue is missing | Deploy the notification service first |

---

## 4. Rollback

`infra/deploy/deploy.sh recommendation` (GitHub Actions, after approval of the `production`
environment) backs up MySQL, pulls `main`, rebuilds only `recommendation` and waits for every Java
service to be healthy; if that fails it rebuilds the previous commit and exits with an error. A code
rollback does not roll back an applied Liquibase changeset.

---

## 5. Database migrations

Liquibase runs on startup and applies only `db/changelog/recommendation/`
(`recommendation-001`, `recommendation-002`). Never rename or move a changelog file.

---

## 6. Alerts

No automated alerting is configured; failures show up in `emc ps`, the deploy job and user
reports.
