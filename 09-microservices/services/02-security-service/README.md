# Security Service — `en-monitor-security-service`

> **Identity authority** of EnergyMonitor: the Security bounded context, extracted from
> `energy-monitor-backend` in October 2026 and in production since 24-Oct-2026. It is the
> authoritative owner of accounts (`Person`, `User`), sessions, global roles and the audit trail.

---

## Responsibility

Registers and authenticates users, issues and renews their tokens, and decides global (platform)
permissions; it owns every security table.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 02 |
| Local port | 8082 |
| Repository | https://github.com/Juan-Cortes2045/en-monitor-security-service |
| Technology | Java 21, Spring Boot 4.1.1, Spring Security (OAuth2 resource server, JWT RS256), Spring Data JPA, Liquibase, Spring AMQP, springdoc-openapi |
| DB engine | MySQL 8.4 — shared schema `energy_monitor`, security tables only (`ADR-003`) |
| Architecture | Hexagonal: `api`, `domain`, `application`, `adapter`, `infrastructure` (package `com.energymonitor.security`) |
| Communicates with | `energy-monitor-backend` (`GET /internal/accounts/{id}/deletion-check`); RabbitMQ (publishes) |
| Consumed by | Web and mobile apps (through the gateway); `energy-monitor-backend` (internal API, events) |
| Status | 🟢 In production |

---

## Responsibilities (what this service DOES)

- Registration with e-mail domain check and e-mail verification code; sign-in is refused (403)
  until the address is verified. Accounts created with Google start verified.
- Login with e-mail and password, and Sign in with Google (`POST /api/v1/auth/google`).
- Sessions: JWT **RS256** access token (**15 minutes**, `sub` = account, no roles) and an opaque
  refresh token (7 days, stored as a hash, rotated on every use, reuse detection by family). It is
  the only service holding the private key.
- **HttpOnly session cookies for the web app** when the request sends
  `X-Session-Transport: cookie`: `em_access` (`Path=/api`) and `em_refresh`
  (`Path=/api/v1/auth`), `SameSite=Strict`, `Secure` unless `SECURITY_SESSION_COOKIE_SECURE=false`.
  The mobile app gets the tokens in the body.
- Password recovery with a six-digit code (stored as `HMAC-SHA256(code, pepper)`, valid 15
  minutes, ten redemption attempts per address per 15 minutes).
- Profile and account deletion (soft delete, after asking the backend whether the account may go).
- RBAC: global role `ADMIN` with `USER.READ`, `USER.MANAGE`, `ROLE.MANAGE`; user directory, status
  changes and role grants under `/api/v1/admin/users/**` and `/api/v1/admin/roles/**`. Permissions
  are read from the database on every request.
- Support contacts: `GET /api/v1/support/contacts` (public) lists the active administrators.
- Audit trail (`audit_log`) and failed-login trail (`login_error_log`).
- Publishes account events and mail requests through its outbox (`events.md`).
- Internal API for the backend: permissions, profiles, administrators.

## Out of scope (what it does NOT do)

- **Does not send e-mail** — it publishes `mail.*` requests; `energy-monitor-backend`
  (`notification`) sends them.
- **Does not know homes** — home roles (`OWNER`, `MEMBER`) belong to the backend's `home`
  module; the global `ADMIN` role grants no access to any home.
- **Does not answer CORS** — the gateway does, and drops `Origin` before forwarding.

---

## Main endpoints

| Path | What it does |
|---|---|
| `POST /api/v1/auth/register`, `/login`, `/google`, `/refresh`, `/logout` | Account and session lifecycle |
| `POST /api/v1/auth/email/verify`, `/email/verification/resend` | E-mail verification |
| `POST /api/v1/auth/password/forgot`, `/password/reset`, `/password/change` | Password recovery and change |
| `GET /api/v1/auth/account`, `PUT /api/v1/auth/profile` | Read the caller's account, update the profile |
| `POST /api/v1/auth/account/delete` | Delete the caller's own account |
| `GET /api/v1/auth/permissions` | The caller's permission codes |
| `GET /api/v1/admin/users`, `PUT /api/v1/admin/users/{idUser}/status`, `PUT`/`DELETE /api/v1/admin/users/{idUser}/roles/{roleName}`, `GET /api/v1/admin/roles` | User administration |
| `GET /api/v1/support/contacts` | Public list of administrators to contact |

### Internal API (for `energy-monitor-backend`)

| Route | Answer |
|---|---|
| `GET /internal/users/{id}/permissions` | `{permissions: [...]}`; 404 without an active account |
| `POST /internal/users/profiles` `{userIds}` (≤ 500) | `{profiles: [{userId, name, lastName, email}]}` |
| `GET /internal/administrators` | `{userIds: [...]}` |

Accepted only from `internal-api.allowed-networks` **and** with `X-Internal-Token`
(`INTERNAL_API_TOKEN`), compared in constant time; the gateway never forwards `/internal/**`.

---

## How to run it locally

```bash
# MySQL and RabbitMQ from energy-monitor-infra (development mode, see 10-devops/local-setup.md)
cd backend/en-monitor-security-service
cp .env.example .env    # SPRING_DATASOURCE_PASSWORD = MYSQL_PASSWORD of infra/.env, SECURITY_RESET_PEPPER, RABBITMQ_*, INTERNAL_API_TOKEN
cp src/main/resources/application-local.yaml.example src/main/resources/application-local.yaml
./mvnw spring-boot:run

# Verify
curl http://localhost:8082/actuator/health     # Swagger UI: http://localhost:8082/swagger-ui.html
```

It does not start without the RSA key pair (`SECURITY_JWT_PRIVATE_KEY_PATH`,
`SECURITY_JWT_PUBLIC_KEY_PATH`, absolute paths) or the pepper. Without RabbitMQ it starts, but
its messages wait in the outbox and the health is DOWN.

Tests: `SPRING_DATASOURCE_PASSWORD=... ./mvnw test` (against the MySQL of infra; 660 tests green
on 22-Oct-2026).

---

## Related documents

- [data-model.md](./data-model.md) — Security tables
- [events.md](./events.md) — Events and mail requests it publishes
- [decisions.md](./decisions.md) — Design decisions
- [runbook.md](./runbook.md) — Operation in production
- Authentication decision → `05-architecture/decisions/records/ADR-010-session-tokens-and-web-cookies.md` (supersedes ADR-005)
- API contract → `07-api/contracts/openapi/security-service.yaml`
- Code design → `docs/Architecture.md` and `README.md` in the repository
