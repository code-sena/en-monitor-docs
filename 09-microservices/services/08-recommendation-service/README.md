# Recommendation Service — `en-monitor-recommendation-service`

> The Recommendations bounded context, the last module extracted from the former
> `energy-monitor-backend`; in production since 28-Oct-2026. It is the authoritative owner of the
> recommendations of a home.

---

## Responsibility

Reviews, every hour, the consumption of the homes that report, stores a recommendation when one of
its rules matches, lets the members of the home list, read and delete them, and announces each new
one to the notification service.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 08 |
| Local port | 8089 |
| Repository | https://github.com/angelgarcia0/en-monitor-recommendation-service |
| Technology | Java 21, Spring Boot 4.1.1, Spring Data JPA, Liquibase, Spring AMQP |
| DB engine | MySQL 8.4 — shared schema `energy_monitor`, recommendation tables only (`ADR-003`) |
| Architecture | Hexagonal: `api`, `domain`, `application`, `adapter`, `infrastructure` (package `com.energymonitor.recommendation`); the rules are pure domain code with no I/O (`RecommendationRules`) |
| Communicates with | home, device and measurement services (internal API), RabbitMQ |
| Consumed by | Web and mobile apps (through the gateway); notification service (`recommendation.created`) |
| Status | 🟢 In production |

---

## Responsibilities (what this service DOES)

- **Public endpoints** (through the gateway; only members of the home):

  | Path | What it does |
  |---|---|
  | `GET /api/v1/recommendations?homeId=` | Recommendations of a home |
  | `PUT /api/v1/recommendations/{idRecommendation}/read` | Marks one as read |
  | `DELETE /api/v1/recommendations/{idRecommendation}` | Deletes a read recommendation (409 when unread) |
  | `DELETE /api/v1/recommendations?homeId=` | Deletes every read recommendation of a home |

- **Evaluates five rules every hour** (`RecommendationScheduler`; first run `RECOMMENDATION_INITIAL_DELAY`
  after start, then every `RECOMMENDATION_EVALUATE_INTERVAL`, `PT1H`), for the homes with readings
  in the last week, in the zone `RECOMMENDATION_ZONE` (`America/Bogota`):

  | Rule (`type`) | Fires when |
  |---|---|
  | `PEAK_HOURS` | At least 40 % of the 7-day energy falls between 18:00 and 22:00 local time (minimum 2 kWh) |
  | `SAVING` | A device that is not always on draws 15 W or more between 01:00 and 05:00 on three nights in a row |
  | `HISTORICAL_COMPARISON` | The week used 25 % more than the previous one |
  | `THRESHOLD` | The current pace exceeds the limit of the home |
  | `HIGH_CONSUMPTION` | The mean power of a device is 30 % above the previous 21 days |

  The same advice is not repeated within 7 days.
- **Publishes** `recommendation.created` through its outbox (`events.md`).

### Who it asks

| Service | Call | For |
|---|---|---|
| Home service | `GET /internal/homes/{homeId}/access?userId=`, `GET /internal/homes/{homeId}/thresholds` | Membership; limits |
| Device service | `GET /internal/devices/{id}`, `GET /internal/homes/{homeId}/devices` | Devices of a home |
| Measurement service | `GET /internal/measurements/hourly-energy`, `GET /internal/measurements/reporting-devices` | Hourly energy; which devices reported |

## Out of scope (what it does NOT do)

- **Does not send anything** — the notification service delivers the recommendation.
- **Does not decide thresholds** — it reads the limits of the home service.

---

## How to run it locally

```bash
# MySQL and RabbitMQ from energy-monitor-infra (development mode, see 10-devops/local-setup.md)
cd backend/en-monitor-recommendation-service
cp .env.example .env
cp src/main/resources/application-local.yaml.example src/main/resources/application-local.yaml
# set MYSQL_PASSWORD of infra/.env, SECURITY_JWT_PUBLIC_KEY_PATH, INTERNAL_API_TOKEN, RABBITMQ_*
./mvnw spring-boot:run

# Verify
curl http://localhost:8089/actuator/health
```

Set `RECOMMENDATION_SCHEDULER_ENABLED=false` to run the service without the hourly evaluation.

Tests: `SPRING_DATASOURCE_PASSWORD=... ./mvnw test` (71 tests green on 28-Oct-2026).

---

## Related documents

- [data-model.md](./data-model.md) — Recommendation tables
- [events.md](./events.md) — Event it publishes
- [runbook.md](./runbook.md) — Operation in production
- API contract → `07-api/contracts/openapi/recommendation-service.yaml`
- Code: `README.md` in the repository
