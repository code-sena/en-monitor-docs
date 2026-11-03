# Alert Service — `en-monitor-alert-service`

> The Alerts bounded context, extracted from the `alert` module of the former
> `energy-monitor-backend` and in production since 28-Oct-2026. It is the authoritative owner of
> the alerts of a home.

---

## Responsibility

Raises threshold, unusual-consumption, limit, connectivity and linked-device alerts from the
readings and the device events, resolves them, lets the members of the home list, read and delete
them, and announces every new alert to the notification service.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 07 |
| Local port | 8088 |
| Repository | https://github.com/angelgarcia0/en-monitor-alert-service |
| Technology | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP |
| DB engine | MySQL 8.4 — shared schema `energy_monitor`, alert tables only (`ADR-003`) |
| Architecture | Hexagonal: `api`, `domain`, `application`, `adapter`, `infrastructure` (package `com.energymonitor.alert`) |
| Communicates with | home, device and measurement services (internal API), RabbitMQ |
| Consumed by | Web and mobile apps (through the gateway); notification service (`alert.raised`) |
| Status | 🟢 In production |

---

## Responsibilities (what this service DOES)

- **Public endpoints** (through the gateway; only members of the home):

  | Path | What it does |
  |---|---|
  | `GET /api/v1/alerts?homeId=&status=` | Alerts of a home, `PENDING` or `RESOLVED` |
  | `GET /api/v1/alerts/{idAlert}` | One alert |
  | `PUT /api/v1/alerts/{idAlert}/read` | Marks it as read (409 for system alerts that resolve themselves) |
  | `DELETE /api/v1/alerts/{idAlert}` | Deletes a resolved alert |
  | `DELETE /api/v1/alerts?homeId=` | Deletes every resolved alert of a home |

- **Reacts to readings** (`measurement.recorded`) and **device events** (`device.*`):

  | Type | Rule |
  |---|---|
  | `THRESHOLD` (HIGH / CRITICAL) | The power of a reading exceeds the threshold of its consumption level; only one while active; resolved by a low reading |
  | `THRESHOLD` — unusual consumption | See below |
  | `LIMIT` | The daily or monthly limit of the home is reached (zone `ALERT_LIMIT_ZONE`, `America/Bogota` by default) |
  | `DEVICE` | First reading of a newly linked device |
  | `CONNECTIVITY` | Device offline; resolved when it reconnects |

- **Unusual-consumption rule** (`RegisterAnomalyAlertService`): a reading is an anomaly when its
  power is at least **200 W** (`MIN_POWER`) and at least **2.5 times** (`FACTOR`) the usual power
  of the appliance. The usual power is the mean of its hourly average over the previous **7 days**,
  excluding the last hour; it only counts with at least **24 hours** of history (`MIN_HOURS`) and a
  mean of at least **5 W** (`MIN_BASELINE`), so a new or almost idle appliance never raises it. The
  baseline is requested from the measurement service (`DeviceBaselineAdapter`). The alert is of
  type `THRESHOLD` with `messageKey = alert.anomaly.device`, resolves by itself when consumption
  returns to normal and is announced by e-mail and push like any alert. The numbers are constants
  in the code, not configuration (DT-27).
- **Publishes** `alert.raised` through its outbox (`events.md`).

### Who it asks

| Service | Call | For |
|---|---|---|
| Home service | `GET /internal/homes/{homeId}/access?userId=`, `GET /internal/homes/{homeId}/thresholds` | Membership; daily and monthly limits |
| Device service | `GET /internal/devices/{id}`, `GET /internal/homes/{homeId}/devices` | Home and name of a module |
| Measurement service | `GET /internal/consumption-levels`, `GET /internal/measurements/hourly-energy` | Level of a reading; hourly energy for the limit and the unusual-consumption rules |

## Out of scope (what it does NOT do)

- **Does not send anything** — the notification service delivers by e-mail and push.
- **Does not store readings** — the measurement service does.

---

## How to run it locally

```bash
# MySQL and RabbitMQ from energy-monitor-infra (development mode, see 10-devops/local-setup.md)
cd backend/en-monitor-alert-service
cp .env.example .env
cp src/main/resources/application-local.yaml.example src/main/resources/application-local.yaml
# set MYSQL_PASSWORD of infra/.env, SECURITY_JWT_PUBLIC_KEY_PATH, INTERNAL_API_TOKEN, RABBITMQ_*
./mvnw spring-boot:run

# Verify
curl http://localhost:8088/actuator/health
```

`ALERT_CONSUMERS_ENABLED` and `ALERT_OUTBOX_RELAY_ENABLED` are `true` by default; turn the
consumers off when you run the service against a broker other services are already consuming.

Tests: `SPRING_DATASOURCE_PASSWORD=... ./mvnw test` (128 tests green on 28-Oct-2026).

---

## Related documents

- [data-model.md](./data-model.md) — Alert tables
- [events.md](./events.md) — Events it publishes and consumes
- [runbook.md](./runbook.md) — Operation in production
- API contract → `07-api/contracts/openapi/alert-service.yaml`
- Code: `README.md` in the repository
