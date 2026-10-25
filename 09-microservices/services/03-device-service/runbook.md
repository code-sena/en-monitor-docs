# Runbook — Device Service

> Operating procedures for whoever is on call. Commands are run on the production host from
> `/opt/energy-monitor/infra`, with `emc() { docker compose -f docker-compose.prod.yml --env-file .env.prod "$@"; }`.
> If this service is down, no module can connect to the broker once the cached decisions expire.

**Service:** `en-monitor-device-service` (container `device`)
**Repository:** https://github.com/Juan-Cortes2045/en-monitor-device-service
**Version:** `main` @ `f8b8526` (24-Oct-2026)
**Last updated:** 2026-10-24

---

## 1. Quick service information

| Field | Value |
|-------|-------|
| Port | 8084, inside the Docker network only |
| Production URL | `https://energy-monitor.tech/api/v1/homes/{homeId}/devices`, `/api/v1/appliance-types` (through the gateway) |
| MQTT | `mqtt.energy-monitor.tech:8883` (modules); this service uses `tcp://mosquitto:1883` |
| Health | `/actuator/health`, includes MySQL and RabbitMQ |
| Escalation | Project team: Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |

---

## 2. Verify the service is healthy

```bash
emc ps device                                    # (healthy)
curl -s -o /dev/null -w '%{http_code}\n' https://energy-monitor.tech/api/v1/appliance-types   # 401: route reaches the device service
emc logs mosquitto | grep "New client connected"   # em-device-service and em-measurement-service connected
```

End-to-end check on the local stack: `infra/e2e/device-flow.sh` (21/21 on 24-Oct-2026).

---

## 3. Known symptoms

| Symptom | Cause | Action |
|---|---|---|
| Broker logs `context deadline exceeded` calling `device:8084` | This service is down (or, in development mode, not running on the host / missing `ufw` rule for 8084) | `emc ps device`, `emc logs device` |
| An MQTT client is refused with `rc=5` (not authorised) | This service does not recognise its credentials | `MQTT_DEVICE_SERVICE_USER/PASS` and `MQTT_MEASUREMENT_SERVICE_USER/PASS` must be the same in the connecting container and in this service |
| A module does not connect: `X509 - Certificate verification failed` | The broker certificate does not cover the address the module uses | Production: check the Let's Encrypt certificate of `mqtt.energy-monitor.tech` (`deploy/renew-mqtt-cert.sh`); local: `DEVICE_MQTT_PUBLIC_HOST` + `./prepare.sh` |
| Device routes answer 503 | This service is down, or the backend did not answer the home access check | `emc ps device backend`; check `INTERNAL_API_TOKEN` is the same everywhere |
| Devices stay `OFFLINE` | No telemetry for 150 s, or this service is not subscribed | `emc logs device`; check the module's Wi-Fi |
| A deleted home's modules are still linked | `home.deleted` waiting in `home_outbox` or in `device-service.device.home-deleted.dlq` | RabbitMQ console through an SSH tunnel (`ssh -L 15672:127.0.0.1:15672 <server>`) |

---

## 4. Rollback

`infra/deploy/deploy.sh device` (GitHub Actions, after approval) backs up MySQL, pulls `main`,
rebuilds only `device` and waits for every Java service to be healthy; if that fails it rebuilds
the previous commit and exits with an error. To return to an earlier version on purpose, revert
the change in the repository and let CD redeploy `main`. A code rollback does not roll back an
applied Liquibase changeset.

---

## 5. Database migrations

Liquibase runs on startup and applies only `db/changelog/device/` (`device-001` … `device-009`).
Never rename or move a changelog file.

---

## 6. Scaling note

Several instances would need different `MQTT_CLIENT_ID` values, and the outbox relay assumes one
instance per service.

---

## 7. Alerts

No automated alerting is configured; failures show up in `emc ps`, the broker log, the deploy job
and user reports.
