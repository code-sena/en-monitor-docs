# Measurement Service — `en-monitor-measurement-service`

> The Monitoring and Measurements bounded context, extracted from the `measurement` module of
> `energy-monitor-backend` and in production since 24-Oct-2026. It is the authoritative owner of
> the readings and of the consumption level catalogue.

---

## Responsibility

Stores every reading the ESP32 + PZEM-004T modules publish over MQTT, and serves the
consumption history, statistics and levels of a device and of a home.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 04 |
| Local port | 8085 |
| Repository | https://github.com/Juan-Cortes2045/en-monitor-measurement-service |
| Technology | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP, MQTT client |
| DB engine | MySQL 8.4 — shared schema `energy_monitor`, measurement tables only (`ADR-003`) |
| Architecture | Hexagonal: `api`, `domain`, `application`, `adapter`, `infrastructure` (package `com.energymonitor.measurement`) |
| Communicates with | `energy-monitor-backend` (home access, thresholds), `en-monitor-device-service` (devices), Mosquitto (MQTT client), RabbitMQ |
| Consumed by | Web and mobile apps (through the gateway); `energy-monitor-backend` (internal API, `measurement.recorded`) |
| Status | 🟢 In production |

---

## Responsibilities (what this service DOES)

- **Readings over MQTT:** subscribed to `energy-monitor/devices/+/telemetry` with its own user
  (`MQTT_MEASUREMENT_SERVICE_USER`, read-only on telemetry) and client id
  (`${MQTT_CLIENT_ID}-telemetry`). It is **the only subscriber that stores telemetry**. Each
  reading is stored with the device timestamp, so samples replayed from a module's offline queue
  land at the time they were taken. The topic, not the payload, identifies the device; a reading
  is stored only for a device the device service knows.
- **Public endpoints** (through the gateway; only members of the device's home may read it — a
  device of another home answers 404):

  | Path | What it does |
  |---|---|
  | `GET /api/v1/measurements?deviceId=&from=&to=` | Readings of a device, oldest first |
  | `GET /api/v1/measurements/latest?deviceId=` | Latest reading of a device |
  | `GET /api/v1/measurements/statistics?deviceId=&from=&to=` | Count, average and maximum power and energy |
  | `GET /api/v1/homes/{homeId}/consumption/summary?zone=` | Current power, today and this month against the home's limits |
  | `GET /api/v1/homes/{homeId}/consumption/history?period=&zone=` | Energy of a home by day, week, month or year |
  | `GET /api/v1/consumption-levels` | Consumption level catalogue |

- **Publishes** every stored reading as `measurement.recorded` through its outbox (`events.md`).
- **Internal API** for `energy-monitor-backend`:

  | Path | Answer |
  |---|---|
  | `GET /internal/consumption-levels` | `[{idConsumptionLevel, name, description, minLimit, maxLimit}]`; a reading belongs to the level whose `[minLimit, maxLimit)` holds its power |
  | `GET /internal/measurements/hourly-energy?deviceId=..&from=..&to=..` | `[{deviceId, hourStart, averagePower, energy}]` per device and UTC hour |
  | `GET /internal/measurements/reporting-devices?since=..` | Ids of the devices with a reading since then |

  Accepted only from `INTERNAL_API_ALLOWED_NETWORKS` **and** with `X-Internal-Token`.

### Who it asks

| Service | Call | For |
|---|---|---|
| `energy-monitor-backend` | `GET /internal/homes/{homeId}/access?userId=` | Is the caller a member of the home? (never cached) |
| `energy-monitor-backend` | `GET /internal/homes/{homeId}/thresholds` | Daily and monthly limits of the summary (404 = none) |
| Device service | `GET /internal/devices/{deviceId}`, `GET /internal/homes/{homeId}/devices` | Home of a device, devices of a home (30 s cache; last answer served up to 1 h while the device service does not answer) |

No answer and nothing kept: the request gets 503 (and an incoming reading is dropped).

## Out of scope (what it does NOT do)

- **Does not raise alerts** — `energy-monitor-backend` hears `measurement.recorded` and its `alert`
  module raises threshold, limit and "device linked" alerts.
- **Does not own homes, thresholds or devices** — it asks their owners.
- **Does not publish to the devices** — only the device service sends commands.

---

## How to run it locally

```bash
# MySQL, RabbitMQ and Mosquitto from energy-monitor-infra (development mode, see 10-devops/local-setup.md)
cd backend/en-monitor-measurement-service
cp .env.example .env
cp src/main/resources/application-local.yaml.example src/main/resources/application-local.yaml
# set MYSQL_PASSWORD of infra/.env, SECURITY_JWT_PUBLIC_KEY_PATH, INTERNAL_API_TOKEN, RABBITMQ_*,
# MQTT_MEASUREMENT_SERVICE_USER/PASS and MEASUREMENT_MQTT_ENABLED=true
./mvnw spring-boot:run

# Verify
curl http://localhost:8085/actuator/health
```

The MQTT subscription is **off by default** (`MEASUREMENT_MQTT_ENABLED=false`), so a run from the
IDE or the tests never takes the readings by surprise; the infra stack turns it on. Only one
subscriber may store readings: two would store every reading twice in the shared table. The
backend and the device service must be running for homes and devices.

Tests: `SPRING_DATASOURCE_PASSWORD=... RABBITMQ_USERNAME=... RABBITMQ_PASSWORD=... ./mvnw test`
(151 tests green on 24-Oct-2026).

---

## Related documents

- [data-model.md](./data-model.md) — Measurement tables
- [events.md](./events.md) — Events it publishes and consumes
- [decisions.md](./decisions.md) — Design decisions of the extraction
- [runbook.md](./runbook.md) — Operation in production
- Events → `05-architecture/decisions/records/ADR-008-rabbitmq-transactional-outbox.md`; internal
  API → `ADR-012-internal-service-api.md`
- API contract → `07-api/contracts/openapi/measurement-service.yaml`
- Code: `README.md` in the repository
