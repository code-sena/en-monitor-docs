# Runbook — Measurement Service

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra`, with `emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }`.
> While this service is down no reading is stored and the consumption screens answer 503.

**Service:** `en-monitor-measurement-service` (container `measurement`)
**Repository:** https://github.com/Juan-Cortes2045/en-monitor-measurement-service
**Version:** `main` @ `ee2cb81` (24-Oct-2026)
**Last updated:** 2026-10-24

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8085, inside the Docker network only |
| Production URL | `https://energy-monitor.tech/api/v1/measurements`, `/api/v1/consumption-levels`, `/api/v1/homes/{homeId}/consumption/...` (through the gateway) |
| MQTT | `tcp://mosquitto:1883`, telemetry only, `MEASUREMENT_MQTT_ENABLED=true` in the infra stack |
| Health | `/actuator/health` |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |

---

## 2. Verify the service is healthy

```bash
emc ps measurement                               # (healthy)
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/consumption-levels   # 401: route reaches the measurement service
emc logs mosquitto | grep "New client connected"   # em-measurement-service connected
```

End-to-end check on the local stack: `infra/e2e/measurement-flow.sh` (25/25 on 24-Oct-2026). It
checks each reading is stored once, announced as `measurement.recorded`, raises and resolves a
`THRESHOLD` alert in the backend, and that `energy-monitor-backend.measurement.recorded` and its
`.dlq` end empty.

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| Readings stored twice | Two subscribers with `MEASUREMENT_MQTT_ENABLED=true` (for example a service on a PC pointed at the same broker) | Leave the subscription on in one instance only |
| No new readings | Subscription off, MQTT user refused (`rc=5`: `MQTT_MEASUREMENT_SERVICE_USER/PASS` differ from the device service's copy), or the device service is down (unknown devices are dropped) | `emc logs measurement mosquitto device` |
| Consumption screens answer 503 | The backend or the device service did not answer and nothing was cached | `emc ps backend device`; check `INTERNAL_API_TOKEN` |
| Alerts no longer react to readings | `measurement.recorded` waiting in `measurement_outbox` (broker down) or in the backend's `.dlq` | RabbitMQ console through an SSH tunnel (`ssh -L 15672:127.0.0.1:15672 <server>`) |

---

## 4. Rollback

`infra/deploy/deploy.sh measurement` (GitHub Actions, after approval) backs up MySQL, pulls `main`,
rebuilds only `measurement` and waits for every Java service to be healthy; if that fails it
rebuilds the previous commit and exits with an error. To return to an earlier version on purpose,
revert the change in the repository and let CD redeploy `main`. A code rollback does not roll back
an applied Liquibase changeset.

---

## 5. Database migrations

Liquibase runs on startup and applies only `db/changelog/measurement/` (`measurement-001` …
`measurement-004`). Never rename or move a changelog file.

---

## 6. Alerts

No automated alerting is configured; failures show up in `emc ps`, the deploy job and user
reports.
