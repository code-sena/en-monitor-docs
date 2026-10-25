# API Gateway — `en-monitor-api-gateway`

> **Single entry point** of the EnergyMonitor API. The web and mobile clients call only this
> service; it forwards each request to the service that owns the path. It is the authoritative
> owner of the routing rules. In production since 24-Oct-2026.

---

## Responsibility

Route every `/api/v1/**` request to its owner service after answering CORS, validating the access
token and rate-limiting the public account endpoints. It owns no business data.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 01 |
| Local port | 8080 |
| Repository | https://github.com/Juan-Cortes2045/en-monitor-api-gateway |
| Technology | Java 21, Spring Boot 4.0.8, Spring Cloud Gateway Server Web MVC 5.0 (Spring Cloud 2025.1.3), Spring Security OAuth2 resource server, Apache HttpClient 5, Caffeine |
| DB engine | — (none) |
| Communicates with | `en-monitor-security-service` :8082, `en-monitor-device-service` :8084, `en-monitor-measurement-service` :8085, `energy-monitor-backend` :8083 |
| Consumed by | Web app (through its nginx, same origin), Android app |
| Status | 🟢 In production |

Spring Boot 4.0 rather than the 4.1 of the other services: Spring Cloud 2025.1 is the release
train that supports it.

---

## Responsibilities (what this service DOES)

- **Routes by path prefix, most specific first:**

  | Path | Service |
  |---|---|
  | `/api/v1/auth/**`, `/api/v1/admin/users/**`, `/api/v1/admin/roles/**`, `/api/v1/support/**` | security :8082 |
  | `/api/v1/homes/{homeId}/devices/**`, `/api/v1/appliance-types/**` | device :8084 |
  | `/api/v1/measurements/**`, `/api/v1/consumption-levels/**`, `/api/v1/homes/{homeId}/consumption/**` | measurement :8085 |
  | any other `/api/v1/**` | backend :8083 |
  | `/internal/**` | never forwarded (401 without a token, 403 with one) |
  | anything else | 404 |

- **Answers CORS** for the whole API and drops the `Origin` header before forwarding, so the
  services never add CORS headers of their own.
- **Validates the access token** (RS256 signature of the security service, issuer, expiry) on
  every route except the public account endpoints (register, login, Google, refresh, password
  forgot and reset, e-mail verify and resend) and `GET /api/v1/support/contacts`. A missing or
  invalid token is answered 401 here; a valid one is forwarded as is.
- **Reads the web app's token from its HttpOnly cookie** (`AccessTokenCookieFilter`): a request
  without `Authorization` but with the `em_access` cookie and `X-Requested-With: XMLHttpRequest`
  is validated and forwarded with `Authorization: Bearer <cookie>`. The public account endpoints
  are left alone, so an expired cookie never blocks the refresh that renews it.
- **Limits the public account endpoints:** 20 `POST`s per client address and path per minute,
  then 429 with `Retry-After`.
- **Answers 503** when a service refuses the connection and **504** when it takes longer than
  30 s. Every answer it produces itself uses the services' error shape (`timestamp`, `status`,
  `error`, `message`, `path`).
- Appends the caller's address to `X-Forwarded-For`; it trusts an incoming `X-Forwarded-For` only
  from loopback or the Docker network (`172.16.0.0/12`).

## Out of scope (what it does NOT do)

- **No business logic and no permissions** — each service validates the token again and decides
  the permissions itself.
- **No data** — no database; the rate-limit counters are in memory.
- **No internal traffic** — `/internal/**` calls between services and Mosquitto's calls to the
  device service never pass through it.
- **No TLS** — Nginx on the host terminates HTTPS in production.

---

## How to run it locally

```bash
# Development mode (MySQL, RabbitMQ and Mosquitto from energy-monitor-infra; see 10-devops/local-setup.md)
cd backend/en-monitor-api-gateway
cp .env.example .env    # set SECURITY_JWT_PUBLIC_KEY_PATH (absolute) and SECURITY_CORS_ALLOWED_ORIGINS=http://localhost:5173
./mvnw spring-boot:run

# Verify
curl http://localhost:8080/actuator/health
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/api/v1/homes   # 401 without a session
```

It starts without the services behind it; their routes answer 503 until they are up. In the
full Docker stack it publishes no port: the web container's nginx forwards `/api/` to
`gateway:8080`.

Tests: `./mvnw test` — they need nothing running (four fake services on free local ports, key
pair generated for the run).

---

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `SERVER_PORT` | `8080` | HTTP port |
| `SECURITY_JWT_PUBLIC_KEY_PATH` | — (required) | Public key of the security service |
| `SECURITY_CORS_ALLOWED_ORIGINS` | empty | Browser origins allowed, comma separated |
| `SECURITY_CORS_ALLOWED_ORIGIN_PATTERNS` | empty | Origin patterns |
| `SECURITY_SERVICE_URL` | `http://localhost:8082` | Security service |
| `DEVICE_SERVICE_URL` | `http://localhost:8084` | Device service |
| `MEASUREMENT_SERVICE_URL` | `http://localhost:8085` | Measurement service |
| `BACKEND_URL` | `http://localhost:8083` | Backend |
| `GATEWAY_CONNECT_TIMEOUT` | `2s` | Connection to a service |
| `GATEWAY_READ_TIMEOUT` | `30s` | Answer of a service |
| `GATEWAY_RATE_LIMIT_REQUESTS` | `20` | `POST`s per address and path in a window |
| `GATEWAY_RATE_LIMIT_WINDOW` | `1m` | Length of the window |

---

## Related documents

- [data-model.md](./data-model.md) — Not applicable (no database)
- [events.md](./events.md) — Not applicable (no events)
- [decisions.md](./decisions.md) — Gateway design decisions
- [runbook.md](./runbook.md) — Operation in production
- Platform decision → `05-architecture/decisions/records/ADR-004-api-gateway.md`
- Sessions and tokens → `05-architecture/decisions/records/ADR-010-session-tokens-and-web-cookies.md`
- API contract → `07-api/contracts/openapi/api-gateway.yaml`
- Code design → `docs/Architecture.md` in the repository (packages `routing`, `security`,
  `ratelimit`, `error`)
