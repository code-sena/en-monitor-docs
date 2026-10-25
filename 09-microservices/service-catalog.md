# Microservices Catalog

> The complete inventory of EnergyMonitor's backend components, as they run in production at
> `https://energy-monitor.tech` on 24-Oct-2026. For the detail of each service, go to
> `09-microservices/services/NN-service-name/`. This catalog is the executive view — one row per
> component.

---

## Services map

```
  Web (React, nginx :80) ──/api/──┐
  Android app (Expo) ─────────────┤
                                  ▼
                       ┌──────────────────────┐
                       │ en-monitor-api-gateway│  CORS · token check · rate limit · routing
                       │        :8080          │  /internal/** refused
                       └──────────┬───────────┘
        ┌───────────────┬─────────┴────────┬──────────────────────┐
        ▼               ▼                  ▼                      ▼
 ┌─────────────┐ ┌─────────────┐ ┌──────────────────┐ ┌───────────────────────────┐
 │ security    │ │ device      │ │ measurement      │ │ energy-monitor-backend    │
 │ :8082       │ │ :8084       │ │ :8085            │ │ :8083 (Spring Modulith)   │
 │             │ │             │ │                  │ │ home · alert ·            │
 │             │ │             │ │                  │ │ recommendation ·          │
 │             │ │             │ │                  │ │ notification              │
 └──────┬──────┘ └──────┬──────┘ └────────┬─────────┘ └─────────────┬─────────────┘
        │  /internal/** (X-Internal-Token) between them, inside Docker │
        └───────────────┴──────┬───────────┴──────────────────────────┘
                 ┌─────────────┴──────────────┬───────────────────────┐
                 ▼                            ▼                       ▼
          MySQL 8.4 (one schema)     RabbitMQ 4.1 (outbox)    Mosquitto (go-auth)
                                                              ◄── ESP32 modules, TLS 8883
```

---

## Service registry

| # | Component | Main responsibility | Port | Repository | DB | Status |
|---|---|---|---|---|---|---|
| 01 | `en-monitor-api-gateway` | Single API entry point: CORS, token validation, rate limit, routing | 8080 | [en-monitor-api-gateway](https://github.com/Juan-Cortes2045/en-monitor-api-gateway) | None | 🟢 In production |
| 02 | `en-monitor-security-service` | Security context: accounts, login, JWT, sessions, RBAC, audit | 8082 | [en-monitor-security-service](https://github.com/Juan-Cortes2045/en-monitor-security-service) | Shared MySQL 8.4 | 🟢 In production |
| 03 | `en-monitor-device-service` | Devices context: linking, credentials, MQTT auth/ACL, commands, connectivity | 8084 | [en-monitor-device-service](https://github.com/Juan-Cortes2045/en-monitor-device-service) | Shared MySQL 8.4 | 🟢 In production |
| 04 | `en-monitor-measurement-service` | Monitoring and Measurements context: readings, history, statistics, consumption | 8085 | [en-monitor-measurement-service](https://github.com/Juan-Cortes2045/en-monitor-measurement-service) | Shared MySQL 8.4 | 🟢 In production |
| — | `energy-monitor-backend` | Modular monolith (Spring Modulith): Home Management, Alerts, Recommendations, Notification | 8083 | [energy-monitor-backend](https://github.com/Juan-Cortes2045/energy-monitor-backend) | Shared MySQL 8.4 | 🟢 In production — 🟡 its four contexts are being migrated to microservices |

**Statuses:** 🟢 In production | 🟡 In progress | 🔴 Planned | ⏸ Deprecated

### Contexts still in the monolith

| Bounded context | Module in `energy-monitor-backend` | Status |
|---|---|---|
| Home Management | `home` | 🟡 Migration to a microservice in progress (teammate: Miguel Ángel García Artunduaga) |
| Alerts | `alert` | 🟡 Migration to a microservice in progress (teammate) |
| Recommendations | `recommendation` | 🟡 Migration to a microservice in progress (teammate) |
| Notification | `notification` | 🟡 Migration to a microservice in progress (teammate) |

Service names, ports and repositories for these four are not defined yet; they will get a row in
the registry above, and a folder under `services/`, when that work lands.

### Version in production (branch `main`, 24-Oct-2026)

| Component | Repository | Commit | Tests (all green) |
|---|---|---|---|
| API gateway | `en-monitor-api-gateway` | `c070cec` | 57 |
| Security service | `en-monitor-security-service` | `334691a` | 660 (22-Oct) |
| Device service | `en-monitor-device-service` | `f8b8526` | 130 |
| Measurement service | `en-monitor-measurement-service` | `ee2cb81` | 151 |
| Backend | `energy-monitor-backend` | `aa9a713` | 481, `ModularityTests` green |
| Web (React 19 + Vite 8) | [Energy-Monitor-application](https://github.com/Juan-Cortes2045/Energy-Monitor-application) | `e59bece` | — |
| Android (Expo 57 / React Native) | [Energy-Monitor-Mobile](https://github.com/angelgarcia0/Energy-Monitor-Mobile) | `894b65a` | — |
| Infrastructure | [energy-monitor-infra](https://github.com/Juan-Cortes2045/energy-monitor-infra) | `eaf961b` | `e2e/device-flow.sh` 21/21, `e2e/measurement-flow.sh` 25/25 |

Every repository has `main` (production) and `dev` (integration); pull requests go to `dev`, and
from `dev` to `main`. The legacy repository `energy-monitor-devices` (old firmware and broker) is
no longer used to run anything.

---

## Detail per service

### 01 — `en-monitor-api-gateway`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/01-api-gateway/` |
| **Responsibility** | Single entry point of the API; routes each path to the service that owns it |
| **Type** | Infrastructure service (no domain) |
| **Port** | 8080 (no published port in Docker; the web container's nginx forwards `/api/` to it) |
| **Technology** | Java 21, Spring Boot 4.0.8, Spring Cloud Gateway Server Web MVC (Spring Cloud 2025.1.3) |
| **DB** | None |
| **Communicates with** | security :8082, device :8084, measurement :8085, backend :8083 |
| **Externally exposed** | Yes — the only API entry point |

**Routes:** see "Route ownership" below.

---

### 02 — `en-monitor-security-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/02-security-service/` |
| **Responsibility** | Identity and access: accounts, registration with e-mail verification, login, Sign in with Google, JWT RS256 (access 15 min) + rotating refresh token, HttpOnly session cookies for the web, password recovery, profile, account deletion, RBAC and user administration, support contacts |
| **Type** | Supporting service |
| **Port** | 8082 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Security (OAuth2 resource server), Spring Data JPA, Liquibase, Spring AMQP |
| **DB** | Shared MySQL 8.4 — security tables |
| **Bounded Context** | `02-domain/domain-map.md` → Security |
| **Externally exposed** | Only through the API gateway |

**Key endpoints:** `/api/v1/auth/**`, `/api/v1/admin/users/**`, `/api/v1/admin/roles/**`,
`GET /api/v1/support/contacts` (public).

**Events published** (`energy-monitor.security`, outbox `security_outbox`): `account.registered`,
`account.administered`, `account.deleted`, `mail.email-verification`, `mail.password-reset`.

**Events consumed:** none.

---

### 03 — `en-monitor-device-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/03-device-service/` |
| **Responsibility** | Devices: linking, editing, credential reissue (API key stored as SHA-256 hash), unlinking, appliance catalogue, MQTT authentication and ACL, MQTT commands, connectivity (OFFLINE after 150 s without data) |
| **Type** | Core service |
| **Port** | 8084 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP, Paho MQTT |
| **DB** | Shared MySQL 8.4 — device tables |
| **Bounded Context** | `02-domain/domain-map.md` → Devices |
| **Externally exposed** | Only through the API gateway; Mosquitto calls its `/internal/mqtt/auth/**` over the Docker network |

**Key endpoints:** `/api/v1/homes/{homeId}/devices/**`, `/api/v1/appliance-types/**`.

**Events published** (`energy-monitor.device`, outbox `device_outbox`): `device.linked`,
`device.unlinked`, `device.connectivity-lost`, `device.connectivity-restored`.

**Events consumed:** `home.deleted` from `energy-monitor-backend` (`energy-monitor.home`).

---

### 04 — `en-monitor-measurement-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/04-measurement-service/` |
| **Responsibility** | Measurements: the only subscriber that stores telemetry; history, latest reading, statistics, home consumption (summary and history), consumption levels; internal API for the backend |
| **Type** | Core service |
| **Port** | 8085 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP, MQTT client |
| **DB** | Shared MySQL 8.4 — measurement tables |
| **Bounded Context** | `02-domain/domain-map.md` → Monitoring and Measurements |
| **Externally exposed** | Only through the API gateway |

**Key endpoints:** `/api/v1/measurements/**`, `/api/v1/consumption-levels/**`,
`/api/v1/homes/{homeId}/consumption/**`.

**Events published** (`energy-monitor.measurement`, outbox `measurement_outbox`):
`measurement.recorded`.

**Events consumed:** `device.linked`, `device.unlinked` from the device service (cache
invalidation).

---

### — `energy-monitor-backend` (modular monolith)

| Field | Value |
|-------|-------|
| **Folder** | None under `services/` — it is not a microservice. See `05-architecture/overview.md` and its repository's `docs/Architecture.md` |
| **Responsibility** | Modules `home` (homes, members, thresholds), `alert`, `recommendation`, `notification` (e-mail, Web Push with VAPID, Expo push, in-app administrator notices), plus the proxy packages `security`, `device`, `measurement` (no domain, no tables) |
| **Port** | 8083 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Modulith |
| **DB** | Shared MySQL 8.4 — home, alert, recommendation and notification tables |
| **Externally exposed** | Only through the API gateway (every `/api/v1/**` path not routed elsewhere) |
| **MQTT** | Not a broker client |

**Events published** (`energy-monitor.home`, outbox `home_outbox`): `home.deleted`.

**Events consumed:** everything on `energy-monitor.security`, the four `device.*` keys and
`measurement.recorded`.

---

## Route ownership

| Paths | Owner |
|---|---|
| `/api/v1/auth/**`, `/api/v1/admin/users/**`, `/api/v1/admin/roles/**`, `/api/v1/support/**` | security :8082 |
| `/api/v1/homes/{homeId}/devices/**`, `/api/v1/appliance-types/**` | device :8084 |
| `/api/v1/measurements/**`, `/api/v1/consumption-levels/**`, `/api/v1/homes/{homeId}/consumption/**` | measurement :8085 |
| any other `/api/v1/**` (homes, members, thresholds, alerts, recommendations, notifications, administrator notices) | backend :8083 |
| `/internal/**` | never forwarded by the gateway |

The device and measurement routes are matched before the backend's catch-all. The clients call
the same paths as before the migration.

---

## Service communication matrix

| Source | Destination | Channel | Type | Description |
|---|---|---|---|---|
| gateway | security, device, measurement, backend | HTTP | Synchronous | Forwards client requests after CORS, token check and rate limit |
| backend | security | `/internal/users/{id}/permissions`, `/internal/users/profiles`, `/internal/administrators` | Synchronous | Permissions, profiles, administrators (cached) |
| security | backend | `/internal/accounts/{id}/deletion-check` | Synchronous | May the account be deleted? |
| backend | device | `/internal/devices/{id}`, `/internal/homes/{homeId}/devices` | Synchronous | Device reads (cached 30 s; stale copy up to 1 h) |
| device | backend | `/internal/homes/{homeId}/access?userId=` | Synchronous | Member / owner of the home? |
| measurement | backend | `/internal/homes/{homeId}/access?userId=`, `/internal/homes/{homeId}/thresholds` | Synchronous | Home access and limits |
| measurement | device | `/internal/devices/{id}`, `/internal/homes/{homeId}/devices` | Synchronous | Home of a device, devices of a home (cached) |
| backend | measurement | `/internal/consumption-levels`, `/internal/measurements/hourly-energy`, `/internal/measurements/reporting-devices` | Synchronous | Levels (cached 10 min), hourly energy for limit alerts and recommendations |
| Mosquitto | device | `/internal/mqtt/auth/user`, `/internal/mqtt/auth/acl` | Synchronous | Authenticate and authorise every MQTT client (decisions cached 5 min) |
| security | backend | RabbitMQ `energy-monitor.security` | Asynchronous | Administrator notices, account clean-up, e-mail delivery |
| backend | device | RabbitMQ `energy-monitor.home` (`home.deleted`) | Asynchronous | Unlink the home's modules and send them `unlink` |
| device | backend, measurement | RabbitMQ `energy-monitor.device` | Asynchronous | CONNECTIVITY alerts and caches (measurement: linked/unlinked only) |
| measurement | backend | RabbitMQ `energy-monitor.measurement` (`measurement.recorded`) | Asynchronous | Threshold, limit and "device linked" alerts |

Internal REST is accepted only from loopback or the Docker network (`172.16.0.0/12`) **and** with
`X-Internal-Token` (`INTERNAL_API_TOKEN`), compared in constant time; Mosquitto's
`/internal/mqtt/auth/**` calls are restricted by network only.

---

## Data ownership matrix

Each table has a single owner. Others access the data through the owner's internal API or its
events, never directly in the database. All tables live in the one shared schema (`ADR-003`).

| Owner | Tables |
|---|---|
| `en-monitor-security-service` | `person`, `user`, `user_configuration`, `user_session`, `refresh_token`, `password_reset_token`, `password_policy`, `security_configuration`, `login_error_log`, `system_role`, `permission`, `system_role_permission`, `user_system_role`, `audit_log`, `security_outbox` |
| `en-monitor-device-service` | `appliance_type`, `device`, `device_home`, `device_status_log`, `device_outbox` |
| `en-monitor-measurement-service` | `measurement`, `consumption_level`, `measurement_outbox` |
| `energy-monitor-backend` | `home`, `home_type`, `home_thresholds`, `user_home`, `home_outbox`, `alert`, `recommendation`, `notification`, `notification_preference`, `push_subscription`, `push_device` |

---

## How to add a new service

1. Assign the next available number in this catalog
2. Create the folder: `cp -r 09-microservices/_template/service 09-microservices/services/NN-new-service`
3. Fill in the README, data-model, events, decisions and runbook of the new service
4. Create the OpenAPI contract in `07-api/contracts/openapi/` if it exposes endpoints
5. Add the service to this catalog (registry, route ownership, communication and data ownership)
6. Add the ADR for any architecture decision the new service needs in `05-architecture/decisions/`
7. Update the diagrams in `05-architecture/overview.md` and `05-architecture/deployment.md`

---

## Correlations

- Architecture and communication patterns → `05-architecture/overview.md`
- Deployment → `05-architecture/deployment.md`
- API contracts → `07-api/contracts/openapi/` (`api-gateway.yaml`, `security-service.yaml`,
  `device-service.yaml`, `measurement-service.yaml`)
- Dependency map and failure effects → `09-microservices/dependency-map.md`
- System events → `02-domain/domain-events.md`
- Tables → `06-data/models.md`
- Template for creating services → `09-microservices/_template/service/`
