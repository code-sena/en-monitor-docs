# Device Service — `en-monitor-device-service`

> The Devices bounded context, extracted from the `device` module of `energy-monitor-backend` in
> October 2026 and in production since 24-Oct-2026. It is the authoritative owner of devices,
> their link to a home, their credentials and their connectivity history.

---

## Responsibility

Links ESP32 + PZEM-004T modules to homes, issues and rotates their MQTT credentials,
authenticates and authorises every MQTT client for the broker, sends commands to the modules and
tracks their connectivity.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 03 |
| Local port | 8084 |
| Repository | https://github.com/Juan-Cortes2045/en-monitor-device-service |
| Technology | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP, Paho MQTT |
| DB engine | MySQL 8.4 — shared schema `energy_monitor`, device tables only (`ADR-003`) |
| Architecture | Hexagonal: `api`, `domain`, `application`, `adapter`, `infrastructure` (package `com.energymonitor.device`) |
| Communicates with | `energy-monitor-backend` (`GET /internal/homes/{homeId}/access`); Mosquitto (MQTT client); RabbitMQ |
| Consumed by | Web and mobile apps (through the gateway); Mosquitto go-auth; `energy-monitor-backend` and the measurement service (internal API, events) |
| Status | 🟢 In production |

---

## Responsibilities (what this service DOES)

- **Device management** (through the gateway; membership and ownership asked to the backend):

  | Path | What it does |
  |---|---|
  | `GET /api/v1/homes/{homeId}/devices` | Devices of a home (any member) |
  | `POST /api/v1/homes/{homeId}/devices` | Links a module; returns its API key and the broker address (owner) |
  | `PUT /api/v1/homes/{homeId}/devices/{deviceId}` | Edits name, appliance and room (owner) |
  | `POST /api/v1/homes/{homeId}/devices/{deviceId}/credentials` | New API key, for a new Wi-Fi network (owner) |
  | `DELETE /api/v1/homes/{homeId}/devices/{deviceId}` | Unlinks the module and tells it over MQTT (owner) |
  | `GET /api/v1/appliance-types` | Appliance catalogue |

- **API keys** are stored only as a SHA-256 hash.
- **Broker authentication and ACL:** Mosquitto (go-auth) calls `POST /internal/mqtt/auth/user` on
  every connection and `POST /internal/mqtt/auth/acl` on every publish/subscribe, and caches each
  decision 5 minutes. Clients it knows:

  | Client | Credentials | May |
  |---|---|---|
  | This service | `MQTT_DEVICE_SERVICE_USER` | Read telemetry and status; the only client that publishes commands |
  | Measurement service | `MQTT_MEASUREMENT_SERVICE_USER` | Read telemetry only |
  | Each ESP32 module | `device_code` + API key | Its own topics only |

- **Commands** to the modules on `energy-monitor/devices/{deviceId}/command` (for example
  `unlink`), sent after the transaction commits.
- **Connectivity:** subscribed to `energy-monitor/devices/+/telemetry` and `/status`. Telemetry
  marks a device `ONLINE` and refreshes `last_seen`; the retained `online`/`offline` status
  (including the last will) opens a row in `device_status_log`; an inactivity check every 30 s
  marks a device `OFFLINE` after 150 s without data (`DEVICE_OFFLINE_AFTER`).
- **Events** through its outbox (`events.md`), and unlinking every module of a deleted home on
  `home.deleted`.
- **Internal API** for the backend and the measurement service:

  | Path | Answer |
  |---|---|
  | `GET /internal/devices/{deviceId}` | `{idDevice, name, applianceTypeId, location, description, installationDate, deviceCode, homeId}` (`homeId` null when unlinked), or 404 |
  | `GET /internal/homes/{homeId}/devices` | The devices linked to the home (empty list when none) |

  Accepted only from `INTERNAL_API_ALLOWED_NETWORKS` **and** with `X-Internal-Token`; with no token
  configured every call is refused.

## Out of scope (what it does NOT do)

- **Does not store readings** — the measurement service is the only subscriber that stores
  telemetry; this service only uses telemetry to track connectivity.
- **Does not know home membership** — it asks `energy-monitor-backend`, which owns homes; without
  an answer the request gets 503.
- **Does not raise alerts** — it publishes `device.connectivity-lost/restored`; the backend's
  `alert` module raises CONNECTIVITY alerts.

---

## How to run it locally

```bash
# MySQL, RabbitMQ and Mosquitto from energy-monitor-infra (development mode, see 10-devops/local-setup.md)
cd backend/en-monitor-device-service
cp .env.example .env
cp src/main/resources/application-local.yaml.example src/main/resources/application-local.yaml
# set MYSQL_PASSWORD of infra/.env, SECURITY_JWT_PUBLIC_KEY_PATH, INTERNAL_API_TOKEN, RABBITMQ_*,
# MQTT_DEVICE_SERVICE_USER/PASS, MQTT_MEASUREMENT_SERVICE_USER/PASS, DEVICE_MQTT_PUBLIC_HOST
./mvnw spring-boot:run

# Verify
curl http://localhost:8084/actuator/health
```

In development mode Mosquitto reaches this service on the host (`host-gateway`); on Linux with
`ufw`, allow the Docker networks to port 8084 once (see `10-devops/local-setup.md`).

Tests: `SPRING_DATASOURCE_PASSWORD=... ./mvnw test` (against the MySQL of infra; 130 tests green
on 24-Oct-2026). They never reach a running backend or broker.

---

## Related documents

- [data-model.md](./data-model.md) — Device tables
- [events.md](./events.md) — Events it publishes and consumes
- [decisions.md](./decisions.md) — Design decisions of the extraction
- [runbook.md](./runbook.md) — Operation in production
- MQTT broker → `05-architecture/decisions/records/ADR-007-mqtt-broker-selection.md`; broker
  authentication → `ADR-011-mqtt-authentication-by-device-service.md`
- Internal API → `05-architecture/decisions/records/ADR-012-internal-service-api.md`
- API contract → `07-api/contracts/openapi/device-service.yaml`
- Code design → `docs/Architecture.md` and `README.md` in the repository
