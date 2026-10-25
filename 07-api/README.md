# 07 — API Contracts

> **What is this?** The formal specification of how microservices communicate with each other
> and with the outside world. The contract is the promise a service makes to its consumers.

## Why contracts are critical in microservices

In a monolith, changing a function is easy: the compiler tells you what you break.
In microservices, a change to an API can silently break other services in production.

**Contract-first development:** design the API before implementing it. This way the team
can work in parallel (frontend, backend, other services) with an agreed contract.

---

## EnergyMonitor API surface (24-Oct-2026)

### Public API — route ownership

Clients call only `en-monitor-api-gateway` (`https://energy-monitor.tech`, or `:8080` in
development mode). The gateway forwards each path, most specific first, to its owner. The
migration to microservices did **not** change any public path.

| Paths | Owner |
|---|---|
| `/api/v1/auth/**`, `/api/v1/admin/users/**`, `/api/v1/admin/roles/**`, `/api/v1/support/**` | `en-monitor-security-service` :8082 |
| `/api/v1/homes/{homeId}/devices/**`, `/api/v1/appliance-types/**` | `en-monitor-device-service` :8084 |
| `/api/v1/measurements/**`, `/api/v1/consumption-levels/**`, `/api/v1/homes/{homeId}/consumption/**` | `en-monitor-measurement-service` :8085 |
| any other `/api/v1/**` (homes, members, thresholds, alerts, recommendations, notifications, administrator notices) | `energy-monitor-backend` :8083 |
| `/internal/**` | never forwarded |

- **Authentication:** access token RS256 (15 min), rotating refresh token; the mobile app sends
  `Authorization: Bearer`, the web app uses HttpOnly cookies (`em_access`, `em_refresh`) plus
  `X-Requested-With: XMLHttpRequest` (`ADR-010`). Public: the account endpoints (register, login,
  google, refresh, password forgot/reset, e-mail verify/resend) and `GET /api/v1/support/contacts`.
- **Errors:** one shape everywhere — `{timestamp, status, error, message, path}`; the gateway
  itself answers 401, 404, 429, 503 and 504 (`contracts/openapi/api-gateway.yaml`).
- **Contracts:** each service publishes its own OpenAPI with springdoc (Swagger UI at
  `/swagger-ui.html` on its port in development mode). In this folder,
  `contracts/openapi/api-gateway.yaml` describes the gateway, and `security-service.yaml`,
  `device-service.yaml` and `measurement-service.yaml` are generated from each service's springdoc
  output. The modules still in `energy-monitor-backend` get their contracts when they are
  extracted. The early `auth-service.yaml` draft is archived in `99-archive/deprecated/`.

### Internal API (`/internal/**`)

Service-to-service REST. Accepted only from loopback or the Docker network (`172.16.0.0/12`)
**and** with `X-Internal-Token` (= `INTERNAL_API_TOKEN`, the same in every service), compared in
constant time. The exception is `/internal/mqtt/auth/**`, which Mosquitto calls and which is
restricted to the allowed networks only. A user's access token does not open the internal API,
and the gateway never forwards it.

| Caller → callee | Endpoint | Purpose |
|---|---|---|
| backend → security | `GET /internal/users/{id}/permissions` | Permission codes of a user (cached 30 s) |
| backend → security | `POST /internal/users/profiles` `{userIds}` (≤ 500) | Names and e-mails (cached 60 s) |
| backend → security | `GET /internal/administrators` | Administrator ids (cached 30 s) |
| security → backend | `GET /internal/accounts/{id}/deletion-check` | 204 may delete; 409 `{message}` refused |
| backend → device, measurement → device | `GET /internal/devices/{id}` | A device and its `homeId` (cached 30 s; stale copy up to 1 h) |
| backend → device, measurement → device | `GET /internal/homes/{homeId}/devices` | Devices of a home |
| device → backend, measurement → backend | `GET /internal/homes/{homeId}/access?userId=` | Member / owner of the home? (never cached) |
| measurement → backend | `GET /internal/homes/{homeId}/thresholds` | Daily and monthly limits (404 = none) |
| backend → measurement | `GET /internal/consumption-levels` | Level catalogue (cached 10 min) |
| backend → measurement | `GET /internal/measurements/hourly-energy?deviceId=&from=&to=` | Energy per device and UTC hour (limit alerts, recommendations) |
| backend → measurement | `GET /internal/measurements/reporting-devices?since=` | Devices with a reading since then |
| Mosquitto → device | `POST /internal/mqtt/auth/user`, `POST /internal/mqtt/auth/acl` | Authenticate / authorise an MQTT client (only the allowed networks) |

---

## What is here and how to fill it in

### `guidelines.md` ⭐
The project's REST standards. **The entire team must follow this before designing an endpoint.**

**Key topics to define:**
```markdown
## Versioning
- URL: /api/v1/resources (version in the URL)
- Header: Accept: application/vnd.api+json;version=1

## Endpoint naming
- Plural for collections: GET /users
- Nouns, not verbs: GET /users/{id}/orders (not: GET /getUserOrders)
- snake_case or kebab-case for URLs

## Pagination
- ?page=1&limit=20 (offset-based)
- ?cursor=xyz (cursor-based for large volumes)

## Standard responses
| Code | When to use it |
|------|----------------|
| 200 | Success with body |
| 201 | Successful creation |
| 204 | Success without body |
| 400 | Client error (validation) |
| 401 | Not authenticated |
| 403 | Not authorized |
| 404 | Resource not found |
| 422 | Unprocessable entity |
| 500 | Server error |

## Error format
{
  "error": "VALIDATION_ERROR",
  "message": "The email field is required",
  "details": [{"field": "email", "message": "required"}]
}
```

### `authentication.md` ⭐
The system's authentication and authorization strategy.
**Fill in:** what mechanism (JWT, OAuth2, API Key), authentication flow, expiration policies,
refresh token handling, RBAC (roles and permissions).

### `contracts/openapi/` ⭐⭐
One `.yaml` file per microservice with the OpenAPI 3.0 contract.

**File name:** `service-name.yaml`

**Minimum structure for each contract:**
```yaml
openapi: 3.0.3
info:
  title: [Service Name] API
  version: 1.0.0
  description: [What this service does]

servers:
  - url: http://localhost:8080/api/v1
    description: Local

paths:
  /resources:
    get:
      summary: List resources
      tags: [Resources]
      parameters:
        - name: page
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: List of resources
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ResourceList'

components:
  schemas:
    Resource:
      type: object
      required: [id, name]
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
  
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

### `_shared.yaml`
Reusable schemas and components across all contracts (pagination, errors, etc.)

---

## Recommended tools

- **Swagger Editor** — Online editor for OpenAPI
- **Redocly** — Generates HTML documentation from the YAML (already configured in `redocly.yaml`)
- **Postman / Insomnia** — For testing the contracts
- **OpenAPI Generator** — Generates client/server code from the YAML

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `04-requirements/functional.md` → what it must do | Endpoints implementing each FR |
| `06-data/models.md` → what data exists | Schemas in the contracts |
| `02-domain/domain-events.md` | Event publishing endpoints |
| Contracts here | `09-microservices/[service]/` that implements them |
| Contracts here | `11-quality/testing-strategy.md` → contract tests |

---

## Questions this section must answer

- What is each endpoint called and what does it do?
- What data does it receive and what does it return?
- How does the client authenticate?
- What errors can the service return?
- How do I version the API without breaking consumers?
