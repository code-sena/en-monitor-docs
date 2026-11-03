# Microservices Catalog

> The complete inventory of EnergyMonitor's backend components, as they run in production at
> `https://energy-monitor.tech` since 28-Oct-2026. For the detail of each service, go to
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
   ┌──────────┬──────────┬────────┴─┬──────────┬──────────┬──────────┬──────────────┐
   ▼          ▼          ▼          ▼          ▼          ▼          ▼              
┌────────┐┌────────┐┌─────────┐┌────────┐┌────────────┐┌────────┐┌──────────────┐
│security││ device ││measure- ││  home  ││notification││ alert  ││recommendation│
│ :8082  ││ :8084  ││  ment   ││ :8086  ││   :8087    ││ :8088  ││    :8089     │
│        ││        ││  :8085  ││        ││            ││        ││              │
└───┬────┘└───┬────┘└────┬────┘└───┬────┘└─────┬──────┘└───┬────┘└──────┬───────┘
    │ /internal/** (X-Internal-Token) between them, inside Docker       │
    └─────────┴──────────┴─────────┼───────────┴───────────┴────────────┘
                ┌───────────────────┴─────────┬───────────────────────┐
                ▼                             ▼                       ▼
         MySQL 8.4 (one schema)      RabbitMQ 4.1 (outbox)    Mosquitto (go-auth)
                                                              ◄── ESP32 modules, TLS 8883
```

---

## Service registry

| # | Component | Main responsibility | Port | Repository | DB | Status |
|---|---|---|---|---|---|---|
| 01 | `en-monitor-api-gateway` | Single API entry point: CORS, token validation, rate limit, routing | 8080 | [en-monitor-api-gateway](https://github.com/Juan-Cortes2045/en-monitor-api-gateway) | None | 🟢 In production |
| 02 | `en-monitor-security-service` | Security context: accounts, registration with e-mail verification, terms, login, Google, JWT, sessions, RBAC, audit | 8082 | [en-monitor-security-service](https://github.com/Juan-Cortes2045/en-monitor-security-service) | Shared MySQL 8.4 | 🟢 In production |
| 03 | `en-monitor-device-service` | Devices context: linking, credentials, MQTT auth/ACL, commands, connectivity | 8084 | [en-monitor-device-service](https://github.com/Juan-Cortes2045/en-monitor-device-service) | Shared MySQL 8.4 | 🟢 In production |
| 04 | `en-monitor-measurement-service` | Monitoring and Measurements context: readings, history, statistics, consumption | 8085 | [en-monitor-measurement-service](https://github.com/Juan-Cortes2045/en-monitor-measurement-service) | Shared MySQL 8.4 | 🟢 In production |
| 05 | `en-monitor-home-service` | Home Management context: homes, members, roles, favourites, thresholds, deletion, ownership transfer | 8086 | [en-monitor-home-service](https://github.com/angelgarcia0/en-monitor-home-service) | Shared MySQL 8.4 | 🟢 In production |
| 06 | `en-monitor-notification-service` | Notification context: e-mail, Web Push, mobile push, preferences, in-app inbox, administrator notices | 8087 | [en-monitor-notification-service](https://github.com/angelgarcia0/en-monitor-notification-service) | Shared MySQL 8.4 | 🟢 In production |
| 07 | `en-monitor-alert-service` | Alerts context: threshold, unusual-consumption, limit, connectivity and linked-device alerts | 8088 | [en-monitor-alert-service](https://github.com/angelgarcia0/en-monitor-alert-service) | Shared MySQL 8.4 | 🟢 In production |
| 08 | `en-monitor-recommendation-service` | Recommendations context: five saving rules evaluated every hour, inbox | 8089 | [en-monitor-recommendation-service](https://github.com/angelgarcia0/en-monitor-recommendation-service) | Shared MySQL 8.4 | 🟢 In production |

**Statuses:** 🟢 In production | 🟡 In progress | 🔴 Planned | ⏸ Deprecated

The migration from the modular monolith finished on 28-Oct-2026: every bounded context is now its
own service. The repository `energy-monitor-backend` is **retired**; it only keeps the ESP32
firmware (`firmware/`) and the history of the monolith. The port `:8083` no longer exists.

### Version in production (branch `main`, close of 28-Oct-2026)

| Component | Repository | Commit | Tests (all green) |
|---|---|---|---|
| API gateway | `en-monitor-api-gateway` | `0b2c164` | 69 |
| Security service | `en-monitor-security-service` | `de52c95` | 774 |
| Device service | `en-monitor-device-service` | `06c1bdc` | 129 |
| Measurement service | `en-monitor-measurement-service` | `8a9b544` | 151 |
| Home service | `en-monitor-home-service` | `6a4e396` | 260 |
| Notification service | `en-monitor-notification-service` | `ecdbc0a` | 235 |
| Alert service | `en-monitor-alert-service` | `0e65430` | 128 |
| Recommendation service | `en-monitor-recommendation-service` | `aeeeec5` | 71 |
| Web (React 19 + Vite 8) | [Energy-Monitor-application](https://github.com/Juan-Cortes2045/Energy-Monitor-application) | `27ad9f0` | 253 (44 files) |
| Android (Expo 57 / React Native) | [Energy-Monitor-Mobile](https://github.com/angelgarcia0/Energy-Monitor-Mobile) | `58f9cbc` | 78 (8 suites) |
| Infrastructure | [energy-monitor-infra](https://github.com/Juan-Cortes2045/energy-monitor-infra) | `ceb1543` | `e2e/*.sh` flows |

The backend services add up to 1,817 tests; with the web and the app, 2,148. The commits are the
ones recorded in `energy-monitor-docs-final/15-enlaces-repositorios-github/enlaces.md`; a few of
the latest ones (unusual consumption, member photos, help in Settings) were merged into `main` and
their deployment depended on the approval of each repository's `production` environment.

Every repository has `main` (production) and `dev` (integration); pull requests go to `dev`, and
from `dev` to `main`. All the services, the web and the infra repository have a
`.github/workflows/ci-cd.yml` pipeline (tests, Docker image, deployment from `main` with approval);
the app runs type check, lint and tests on every push and queues an EAS build from `main`.

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
| **Communicates with** | the seven services below |
| **Externally exposed** | Yes — the only API entry point |

**Routes:** see "Route ownership" below.

---

### 02 — `en-monitor-security-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/02-security-service/` |
| **Responsibility** | Identity and access: accounts, registration with e-mail verification and acceptance of the terms, login, Sign in with Google, JWT RS256 (access 15 min) + rotating refresh token, HttpOnly session cookies for the web, password recovery, profile (with photo), account deletion, RBAC and user administration, support contacts |
| **Type** | Supporting service |
| **Port** | 8082 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Security (OAuth2 resource server), Spring Data JPA, Liquibase, Spring AMQP |
| **DB** | Shared MySQL 8.4 — security tables |
| **Bounded Context** | `02-domain/domain-map.md` → Security |
| **Externally exposed** | Only through the API gateway |

**Key endpoints:** `/api/v1/auth/**`, `/api/v1/admin/users/**`, `/api/v1/admin/roles/**`,
`GET /api/v1/support/contacts` (public).

**Events published** (`energy-monitor.security`, outbox `security_outbox`): `account.registered`,
`account.administered`, `account.deleted`, `account.first-signed-in`, `mail.email-verification`,
`mail.password-reset`, `mail.account-deletion`.

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

**Events consumed:** `home.deleted` from the home service (`energy-monitor.home`).

---

### 04 — `en-monitor-measurement-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/04-measurement-service/` |
| **Responsibility** | Measurements: the only subscriber that stores telemetry; history, latest reading, statistics, home consumption (summary and history), consumption levels; internal API for the alert and recommendation services |
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

### 05 — `en-monitor-home-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/05-home-service/` |
| **Responsibility** | Homes: creation, access code, members, roles, favourites, thresholds (daily or monthly limit), deletion, ownership transfer; answers who may act on a home |
| **Type** | Core service |
| **Port** | 8086 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP |
| **DB** | Shared MySQL 8.4 — home tables |
| **Bounded Context** | `02-domain/domain-map.md` → Home Management |
| **Externally exposed** | Only through the API gateway |

**Key endpoints:** `/api/v1/homes/**` (except devices and consumption), `/api/v1/home-types/**`.

**Events published** (`energy-monitor.home`, outbox `home_outbox`): `home.created`,
`home.member-joined`, `home.member-left`, `home.member-removed`, `home.thresholds-updated`,
`home.ownership-transferred`, `home.deleted`.

**Events consumed:** `account.deleted` from the security service.

---

### 06 — `en-monitor-notification-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/06-notification-service/` |
| **Responsibility** | Delivery of alerts, recommendations and activity by e-mail, Web Push and mobile push; code mails; preferences; in-app inbox; administrator notices; delivery log |
| **Type** | Supporting service |
| **Port** | 8087 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP, Web Push (VAPID), Expo push |
| **DB** | Shared MySQL 8.4 — notification tables |
| **Bounded Context** | `02-domain/domain-map.md` → Notification |
| **Externally exposed** | Only through the API gateway |

**Key endpoints:** `/api/v1/notifications/**`, `/api/v1/admin/notifications/**`.

**Events published:** none (no outbox).

**Events consumed:** `account.*`, `mail.*`, `home.*`, `device.linked`, `device.unlinked`,
`alert.raised`, `recommendation.created`.

---

### 07 — `en-monitor-alert-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/07-alert-service/` |
| **Responsibility** | Alerts: threshold, unusual consumption, limit, connectivity and linked-device; inbox (list, read, delete) |
| **Type** | Core service |
| **Port** | 8088 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP |
| **DB** | Shared MySQL 8.4 — alert tables |
| **Bounded Context** | `02-domain/domain-map.md` → Alerts |
| **Externally exposed** | Only through the API gateway |

**Key endpoints:** `/api/v1/alerts/**`.

**Events published** (`energy-monitor.alert`, outbox `alert_outbox`): `alert.raised`.

**Events consumed:** `measurement.recorded`; `device.linked`, `device.unlinked`,
`device.connectivity-lost`, `device.connectivity-restored`.

---

### 08 — `en-monitor-recommendation-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/08-recommendation-service/` |
| **Responsibility** | Recommendations: five saving rules evaluated every hour per home; inbox (list, read, delete) |
| **Type** | Core service |
| **Port** | 8089 |
| **Technology** | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP |
| **DB** | Shared MySQL 8.4 — recommendation tables |
| **Bounded Context** | `02-domain/domain-map.md` → Recommendations |
| **Externally exposed** | Only through the API gateway |

**Key endpoints:** `/api/v1/recommendations/**`.

**Events published** (`energy-monitor.recommendation`, outbox `recommendation_outbox`):
`recommendation.created`.

**Events consumed:** none (it polls the internal APIs every hour).

---

## Route ownership

| Paths | Owner |
|---|---|
| `/api/v1/auth/**`, `/api/v1/admin/users/**`, `/api/v1/admin/roles/**`, `/api/v1/support/**` | security :8082 |
| `/api/v1/homes/{homeId}/devices/**`, `/api/v1/appliance-types/**` | device :8084 |
| `/api/v1/measurements/**`, `/api/v1/consumption-levels/**`, `/api/v1/homes/{homeId}/consumption/**` | measurement :8085 |
| the rest of `/api/v1/homes/**`, `/api/v1/home-types/**` | home :8086 |
| `/api/v1/notifications/**`, `/api/v1/admin/notifications/**` | notification :8087 |
| `/api/v1/alerts/**` | alert :8088 |
| `/api/v1/recommendations/**` | recommendation :8089 |
| `/internal/**` | never forwarded by the gateway |

The device and consumption routes are matched before the general home route, so the clients call
the same paths as before the migration.

---

## Service communication matrix

| Source | Destination | Channel | Type | Description |
|---|---|---|---|---|
| gateway | the seven services | HTTP | Synchronous | Forwards client requests after CORS, token check and rate limit |
| security | home | `/internal/accounts/{id}/deletion-check` | Synchronous | May the account be deleted? |
| home | security | `/internal/users/profiles` | Synchronous | Names (and photos) of the members |
| device, measurement, alert, recommendation | home | `/internal/homes/{homeId}/access?userId=` | Synchronous | Member / owner of the home? (never cached) |
| measurement, alert, recommendation | home | `/internal/homes/{homeId}/thresholds` | Synchronous | Daily and monthly limits |
| notification | home | `/internal/homes/{homeId}` | Synchronous | Name and members of a home |
| notification | security | `/internal/users/profiles`, `/internal/users/{id}/permissions`, `/internal/administrators` | Synchronous | Names and e-mails, `USER.READ`, administrators (cached) |
| measurement, alert, recommendation, notification | device | `/internal/devices/{id}`, `/internal/homes/{homeId}/devices` | Synchronous | Device reads (cached 30 s; stale copy up to 1 h) |
| alert, recommendation | measurement | `/internal/consumption-levels`, `/internal/measurements/hourly-energy`, `/internal/measurements/reporting-devices` | Synchronous | Levels (cached), hourly energy, reporting devices |
| Mosquitto | device | `/internal/mqtt/auth/user`, `/internal/mqtt/auth/acl` | Synchronous | Authenticate and authorise every MQTT client (decisions cached 5 min) |
| security | notification, home | RabbitMQ `energy-monitor.security` | Asynchronous | `account.*`, `mail.*`: administrator notices, code mails, account clean-up |
| home | device, notification | RabbitMQ `energy-monitor.home` | Asynchronous | `home.deleted` unlinks the modules; `home.*` activity |
| device | alert, measurement, notification | RabbitMQ `energy-monitor.device` | Asynchronous | CONNECTIVITY alerts, caches (measurement: linked/unlinked only), activity |
| measurement | alert | RabbitMQ `energy-monitor.measurement` (`measurement.recorded`) | Asynchronous | Threshold, limit, unusual-consumption and "device linked" alerts |
| alert | notification | RabbitMQ `energy-monitor.alert` (`alert.raised`) | Asynchronous | Delivery of the alert |
| recommendation | notification | RabbitMQ `energy-monitor.recommendation` (`recommendation.created`) | Asynchronous | Delivery of the recommendation |

Internal REST is accepted only from loopback or the Docker network (`172.16.0.0/12`) **and** with
`X-Internal-Token` (`INTERNAL_API_TOKEN`), compared in constant time; Mosquitto's
`/internal/mqtt/auth/**` calls are restricted by network only.

---

## Data ownership matrix

Each table has a single owner. Others access the data through the owner's internal API or its
events, never directly in the database. All tables live in the one shared schema (`ADR-003`).

| Owner | Tables |
|---|---|
| `en-monitor-security-service` | `person`, `user`, `user_configuration`, `user_session`, `refresh_token`, `password_reset_token`, `password_policy`, `security_configuration`, `login_error_log`, `system_role`, `permission`, `system_role_permission`, `user_system_role`, `audit_log`, `pending_registration`, `security_outbox` |
| `en-monitor-device-service` | `appliance_type`, `device`, `device_home`, `device_status_log`, `device_outbox` |
| `en-monitor-measurement-service` | `measurement`, `consumption_level`, `measurement_outbox` |
| `en-monitor-home-service` | `home`, `home_type`, `home_thresholds`, `user_home`, `home_outbox` |
| `en-monitor-notification-service` | `notification`, `notification_preference`, `push_subscription`, `push_device` |
| `en-monitor-alert-service` | `alert`, `alert_outbox` |
| `en-monitor-recommendation-service` | `recommendation`, `recommendation_outbox` |

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
- API contracts → `07-api/contracts/openapi/` (one file per service plus `api-gateway.yaml`)
- Dependency map and failure effects → `09-microservices/dependency-map.md`
- System events → `02-domain/domain-events.md`
- Tables → `06-data/models.md`
- Template for creating services → `09-microservices/_template/service/`
