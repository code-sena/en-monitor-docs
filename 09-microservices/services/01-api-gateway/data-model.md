# Data Model — API Gateway

> The API gateway has **no database** and owns no table in the shared schema.

---

## Database engine

**Engine:** none.

**Justification:** the gateway has no domain: it routes, validates and refuses. Its routes and
limits are code and configuration (`GatewayRoutes`, environment variables), not data.

---

## In-memory state

| Data | Storage | Lifetime | Purpose |
|------|---------|----------|---------|
| Rate-limit counters | Memory of the instance (fixed window) | `GATEWAY_RATE_LIMIT_WINDOW` (1 minute) | 20 `POST`s per client address and path on the public account endpoints |

The counters are per instance and start again on restart; running several instances would need a
shared store (see `decisions.md`).

---

## Configuration instead of tables

| What | Where |
|---|---|
| Which service owns each path | `routing` package (`GatewayRoutes`) of the repository |
| Service addresses | `SECURITY_SERVICE_URL`, `DEVICE_SERVICE_URL`, `MEASUREMENT_SERVICE_URL`, `HOME_SERVICE_URL`, `NOTIFICATION_SERVICE_URL`, `ALERT_SERVICE_URL`, `RECOMMENDATION_SERVICE_URL` |
| Public account endpoints (no token required) | `SecurityFilterChainConfiguration.PUBLIC_ENDPOINTS` |
| CORS origins | `SECURITY_CORS_ALLOWED_ORIGINS`, `SECURITY_CORS_ALLOWED_ORIGIN_PATTERNS` |

A routing change is a code change and a redeploy (`infra/deploy/deploy.sh gateway`).

---

## Schema migration

Not applicable: no Liquibase changelog.

---

## Correlations

- Environments → `10-devops/environments.md`
- Runbook → `runbook.md`
- Route ownership → `09-microservices/service-catalog.md`
