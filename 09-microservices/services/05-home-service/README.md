# Home Service — `en-monitor-home-service`

> The Home Management bounded context, extracted from the `home` module of the former
> `energy-monitor-backend` and in production since 28-Oct-2026. It is the authoritative owner of
> homes, memberships and consumption thresholds.

---

## Responsibility

Creates homes, joins members with an 8-character access code, keeps their roles, favourites and
the consumption thresholds, hands a home over to another member, deletes it, and answers the other
services whether a user may act on a home.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 05 |
| Local port | 8086 |
| Repository | https://github.com/angelgarcia0/en-monitor-home-service |
| Technology | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP |
| DB engine | MySQL 8.4 — shared schema `energy_monitor`, home tables only (`ADR-003`) |
| Architecture | Hexagonal: `api`, `domain`, `application`, `adapter`, `infrastructure` (package `com.energymonitor.home`) |
| Communicates with | `en-monitor-security-service` (names and emails), RabbitMQ |
| Consumed by | Web and mobile apps (through the gateway); device, measurement, alert, recommendation, notification and security services (internal API) |
| Status | 🟢 In production |

---

## Responsibilities (what this service DOES)

- **Public endpoints** (through the gateway; the identity is the `sub` claim of the JWT):

  | Path | What it does |
  |---|---|
  | `POST /api/v1/homes` | Creates a home; the caller is its owner and gets the access code |
  | `GET /api/v1/homes` | Homes of the caller, with the role and whether each is a favourite |
  | `POST /api/v1/homes/join` | Joins a home with its access code (there are no e-mail invitations) |
  | `GET /api/v1/homes/{homeId}/members` | Members with names (asked to the security service) and roles |
  | `PUT /api/v1/homes/{homeId}/favorite` | Marks or unmarks the home as favourite |
  | `DELETE /api/v1/homes/{homeId}/members/me` | Leaves the home (the last owner cannot) |
  | `DELETE /api/v1/homes/{homeId}/members/{userId}` | Removes a member (OWNER) |
  | `PUT /api/v1/homes/{homeId}/owner` | Transfers ownership to a member, an atomic role swap (OWNER) |
  | `DELETE /api/v1/homes/{homeId}` | Soft-deletes the home and announces `home.deleted` (OWNER) |
  | `GET`, `PUT /api/v1/homes/{homeId}/thresholds`, `POST …/defaults` | Daily or monthly limit (6 kWh and 180 kWh by default); changing it is OWNER only |
  | `GET /api/v1/home-types`, `GET /api/v1/home-types/{idHomeType}` | Home type catalogue |

- **Internal API** (`/internal/**`, never forwarded by the gateway):

  | Path | Answer |
  |---|---|
  | `GET /internal/homes/{homeId}/access?userId=` | Is the user a member or the owner of the home? |
  | `GET /internal/homes/{homeId}/thresholds` | Daily and monthly limits (404 = no limits) |
  | `GET /internal/homes/{homeId}` | Name and members of the home, for the notification service |
  | `GET /internal/accounts/{id}/deletion-check` | 204 if the account may be deleted; 409 when it is the only owner of a home with other members |

  Accepted only from `INTERNAL_API_ALLOWED_NETWORKS` **and** with `X-Internal-Token`.

- **Publishes** `home.created`, `home.member-joined`, `home.member-left`, `home.member-removed`,
  `home.thresholds-updated`, `home.ownership-transferred` and `home.deleted` through its outbox
  (`events.md`).
- **Consumes** `account.deleted`: drops the memberships of the deleted account and deletes the
  homes it solely owned.

### Who it asks

| Service | Call | For |
|---|---|---|
| Security service | `POST /internal/users/profiles` | Names (and photos) of the members of a home |

## Out of scope (what it does NOT do)

- **Does not own users** — identity, profile and roles belong to the security service.
- **Does not own devices** — it only announces `home.deleted`; the device service unlinks the
  modules.
- **Does not send notifications** — the notification service listens to its `home.*` events.

---

## How to run it locally

```bash
# MySQL and RabbitMQ from energy-monitor-infra (development mode, see 10-devops/local-setup.md)
cd backend/en-monitor-home-service
cp .env.example .env
cp src/main/resources/application-local.yaml.example src/main/resources/application-local.yaml
# set MYSQL_PASSWORD of infra/.env, SECURITY_JWT_PUBLIC_KEY_PATH, INTERNAL_API_TOKEN, RABBITMQ_*
./mvnw spring-boot:run

# Verify
curl http://localhost:8086/actuator/health
```

Tests: `SPRING_DATASOURCE_PASSWORD=... ./mvnw test` (260 tests green on 28-Oct-2026).

---

## Related documents

- [data-model.md](./data-model.md) — Home tables
- [events.md](./events.md) — Events it publishes and consumes
- [runbook.md](./runbook.md) — Operation in production
- Events → `05-architecture/decisions/records/ADR-008-rabbitmq-transactional-outbox.md`; internal
  API → `ADR-012-internal-service-api.md`
- API contract → `07-api/contracts/openapi/home-service.yaml`
- Code: `README.md` and `docs/Architecture.md` in the repository
