# Domain Map — Bounded Contexts

---

## 1. Domain overview

Energy Monitor manages the full cycle of measuring, understanding, and reacting to a household's
electricity consumption. It starts when a home is registered and a measuring device is attached
to a specific appliance or load; from there, the system needs to know who owns that data, collect
real electrical readings automatically, classify how much a home is consuming, and react to that
behavior with alerts and recommendations.

Each of these responsibilities is kept in a separate area of the domain on purpose: they change
for different reasons and at different speeds. Registering a new home does not require touching
how consumption is classified. Changing how a recommendation is generated should never require
touching who is allowed to log in.

---

## 2. Identified Bounded Contexts

### Bounded Context: Security

| Field | Value |
|-------|-------|
| **Name** | Security |
| **Responsibility** | Knows who a user is, whether their credentials are valid, what they are allowed to do, and keeps a record of security-relevant actions |
| **Owning team** | Juan Cortes |
| **Microservice(s)** | `en-monitor-security-service` (:8082) — in production since 24-Oct-2026 |
| **Database** | Shared MySQL instance; owns the tables under the identity/access domain |
| **Ubiquitous Language** | `Person`, `User`, `Credentials`, `Session`, `Access Token`, `Refresh Token`, `Password Reset Token`, `Password Policy`, `System Role`, `Permission`, `RBAC`, `Audit Log`. Full definitions in `01-context/glossary.md`. |

**Context-specific terms:**

| Term | Meaning in THIS context | Different in another context? |
|------|--------------------------|-------------------------------|
| Role | `System Role` — a global permission grouping | Yes — in Home Management, "role" means `Home Role` (`OWNER`/`MEMBER`), scoped to one home only |
| User | System account with credentials, status, and login history | Distinct from `Person` (human identity) and from `UserHome` (membership) |

---

### Bounded Context: Home Management

| Field | Value |
|-------|-------|
| **Name** | Home Management |
| **Responsibility** | Groups devices, members, and thresholds under a home, and manages who belongs to which home and with what role |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | None yet — module `home` of `energy-monitor-backend` (:8083); migration to a microservice in progress |
| **Database** | Shared MySQL instance; owns the tables under the home domain |
| **Ubiquitous Language** | `Home`, `Home Type`, `Access Code`, `Home Role`, `Threshold`, `Home Thresholds`. Full definitions in `01-context/glossary.md`. |

**Context-specific terms:**

| Term | Meaning in THIS context | Different in another context? |
|------|--------------------------|-------------------------------|
| Role | `Home Role` — scoped to one specific home (`OWNER`/`MEMBER`) | Yes — in Security, "role" means `System Role`, global to the platform |
| Threshold | Consumption limits (daily/monthly) owned by the home | Not a user preference; thresholds belong to `HomeThresholds` aggregate |

---

### Bounded Context: Devices

| Field | Value |
|-------|-------|
| **Name** | Devices |
| **Responsibility** | Registers measuring devices, associates each one with a home, authenticates them via API key, and tracks their connectivity |
| **Owning team** | Juan Cortes |
| **Microservice(s)** | `en-monitor-device-service` (:8084) — in production since 24-Oct-2026 |
| **Database** | Shared MySQL instance; owns the tables under the device domain |
| **Ubiquitous Language** | `Device`, `Appliance`, `Appliance Type`, `Device Status`, `API Key`, `Device Code`. Full definitions in `01-context/glossary.md`. |

**Context-specific terms:**

| Term | Meaning in THIS context | Different in another context? |
|------|--------------------------|-------------------------------|
| Device | The physical measuring unit — the thing that measures | Distinct from `Appliance`, which is the load being measured |
| API Key | Secret assigned to a device to authenticate it against the MQTT broker | Not a user credential; the broker (go-auth) validates it via an HTTP callback to the device service, which stores only its SHA-256 hash |

---

### Bounded Context: Monitoring and Measurements

| Field | Value |
|-------|-------|
| **Name** | Monitoring and Measurements |
| **Responsibility** | Receives readings published by devices over MQTT, stores them, and makes consumption history and statistics available |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | `en-monitor-measurement-service` (:8085) — in production since 24-Oct-2026 |
| **Database** | Shared MySQL instance; owns the tables under the measurement domain |
| **Ubiquitous Language** | `Measurement`, `Voltage`, `Current`, `Active Power`, `Energy`, `Power Factor`, `Consumption`, `Threshold`, `Consumption Level`. Full definitions in `01-context/glossary.md`. |

---

### Bounded Context: Alerts

| Field | Value |
|-------|-------|
| **Name** | Alerts |
| **Responsibility** | Reacts to a measurement or a device going silent by raising and tracking an alert until it is resolved |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | None yet — module `alert` of `energy-monitor-backend`; migration to a microservice in progress |
| **Database** | Shared MySQL instance; owns the tables under the alerting domain |
| **Ubiquitous Language** | `Alert`, `Alert Severity`, `Alert Type` (THRESHOLD, CONNECTIVITY, DEVICE, LIMIT), `Consumption Level` (consumed, not owned). Full definitions in `01-context/glossary.md`. |

---

### Bounded Context: Recommendations

| Field | Value |
|-------|-------|
| **Name** | Recommendations |
| **Responsibility** | Generates suggestions from a home's own measured consumption behavior, and tracks whether the user has read them |
| **Owning team** | Juan Cortes |
| **Microservice(s)** | None yet — module `recommendation` of `energy-monitor-backend`; migration to a microservice in progress |
| **Database** | Shared MySQL instance; owns the tables under the recommendation domain |
| **Ubiquitous Language** | `Recommendation`, `Recommendation Type`, `Message Key`. Full definitions in `01-context/glossary.md`. |

---

### Bounded Context: Notification

| Field | Value |
|-------|-------|
| **Name** | Notification |
| **Responsibility** | Owns the delivery log: a record that an outbound message was attempted, over which channel, and whether it succeeded. Delivers alerts, recommendations, e-mail verification and password reset codes via email, Web Push (VAPID) and mobile push (Expo), plus in-app notices to administrators. |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | None yet — module `notification` of `energy-monitor-backend`; migration to a microservice in progress |
| **Database** | Shared MySQL instance; owns the tables under the notification domain |
| **Ubiquitous Language** | `Notification`, `Notification Channel` (EMAIL, PUSH), `Notification Source Type` (ALERT, RECOMMENDATION, PASSWORD_RESET), `Notification Status` (PENDING, SENT, FAILED), `Push Subscription`. Full definitions in `01-context/glossary.md`. |

---

## 3. Context Map

```
┌────────────┐   OHS / PL    ┌──────────────────┐
│  Security  │──────────────▶│  Home Management │
│ (identity) │               └────────┬─────────┘
│            │                        │ U → D
│            │   OHS / PL             ▼
│            │──────────────▶  ┌────────────────┐
│            │                 │    Devices     │
└─────┬──────┘                 └────────┬───────┘
      │ OHS / PL                        │ U → D
      │                                 ▼
      │                    ┌─────────────────────────┐
      └───────────────────▶│ Monitoring and          │
                           │ Measurements            │
                           └──────────┬──────────────┘
                                      │
                                      │    U → D (RabbitMQ + internal REST)
                          ┌───────────┴───────────┐
                          ▼                       ▼
                    ┌──────────┐            ┌────────────────┐
                    │  Alerts  │            │Recommendations │
                    └────┬─────┘            └───────┬────────┘
                         │                         │
                         │  U → D (in-process Spring event)
                         ▼                         ▼
                    ┌────────────────────────────────────┐
                    │         Notification               │
                    │  (delivers alerts, recommendations,│
                    │   password reset via email/push)   │
                    └────────────────────────────────────┘
```

> Deployment on 24-Oct-2026: Security, Devices and Monitoring and Measurements are
> microservices; Home Management, Alerts, Recommendations and Notification run in
> `energy-monitor-backend` (migration in progress). Channels are in the relationships table.
>
> This diagram draws the relationships that involve an active integration (an API call, a
> token claim, or a domain event). `Alert` and `Recommendation` also store `homeId` and,
> where applicable, `deviceId` as plain foreign references — Home Management and Devices are not
> called at write time for this, and no arrow is drawn for it. The full list, including these
> reference-only relationships, is in the table below.

### Context relationship types

| Type | Symbol | Description | Example |
|------|--------|-------------|---------|
| **Upstream → Downstream** | `U → D` | U provides, D consumes. D depends on U. | Devices → Measurements |
| **Open Host Service** | `OHS` | U publishes a stable, well-defined contract that many contexts can consume | Security → everyone else |
| **Published Language** | `PL` | The explicit shared contract itself (a claim format, an event schema) | JWT claims, domain events |
| **Anti-Corruption Layer** | `ACL` | Translates between different models to prevent leakage | Home Management uses `UserDirectoryPort` to query user names without depending on Security's domain model |

This project does not currently use Shared Kernel or Conformist relationships — there is no legacy system to protect against, and no two contexts share a model.

### Relationships table

Channels as implemented on 24-Oct-2026. "In-process" means both contexts still run inside
`energy-monitor-backend`; "internal REST" means `/internal/**` with `X-Internal-Token`; event
names in `code` are RabbitMQ routing keys.

| Context A | Relationship | Context B | Communication channel | Contract |
|-----------|-------------|-----------|------------------------|---------|
| Security | OHS / PL | Every other context | Access token validated locally with the public key (gateway and each service) | `sub` claim (user identity); no roles in the token |
| Security | OHS / PL | Home Management | Internal REST `POST /internal/users/profiles`; RabbitMQ `account.deleted` | User profiles; clean-up of deleted accounts |
| Home Management | U → D | Security | Internal REST `GET /internal/accounts/{id}/deletion-check` | May the account be deleted (204 / 409) |
| Security | OHS / PL | Notification | RabbitMQ `account.registered`, `account.administered`, `mail.email-verification`, `mail.password-reset`; internal REST `GET /internal/administrators`, `GET /internal/users/{id}/permissions` | Administrator notices and e-mail delivery |
| Home Management | U → D | Devices | Internal REST `GET /internal/homes/{homeId}/access` (device → backend); RabbitMQ `home.deleted` | Membership/ownership check; unlink the modules of a deleted home |
| Home Management | U → D | Monitoring and Measurements | Internal REST `GET /internal/homes/{homeId}/access`, `GET /internal/homes/{homeId}/thresholds` | Membership check; limits of the consumption summary |
| Devices | U → D | Monitoring and Measurements | MQTT (module → broker, authenticated by the device service); internal REST `GET /internal/devices/{id}`, `GET /internal/homes/{homeId}/devices`; RabbitMQ `device.linked`, `device.unlinked` | Readings are stored only for a known device; cache invalidation |
| Monitoring and Measurements | U → D | Alerts | RabbitMQ `measurement.recorded`, republished in memory as `MeasurementRecorded`; internal REST `GET /internal/consumption-levels`, `GET /internal/measurements/hourly-energy` | Threshold, limit and "device linked" alerts |
| Monitoring and Measurements | U → D | Recommendations | Internal REST `GET /internal/measurements/hourly-energy`, `GET /internal/measurements/reporting-devices` | Hourly energy per device |
| Devices | U → D | Alerts | RabbitMQ `device.connectivity-lost`, `device.connectivity-restored`, republished in memory as `DeviceConnectivityLost/Restored` | CONNECTIVITY alerts |
| Devices | U → D | Recommendations, Notification, Alerts | Internal REST `GET /internal/devices/{id}`, `GET /internal/homes/{homeId}/devices` (through the backend's `device` proxy, cached) | `deviceId` as a foreign reference |
| Home Management | U → D | Alerts, Recommendations, Notification | In-process (`home::api`) | `homeId` as a foreign reference |
| Alerts | U → D | Notification | In-process Spring event `AlertRaised` | See `domain-events.md` |
| Recommendations | U → D | Notification | In-process Spring event `RecommendationCreated` | See `domain-events.md` |
| Home Management | ACL | Security | `UserDirectoryPort` (outbound port), answered by the backend's `security` proxy | Translates Security's `User` to Home's `UserProfile` |

No context reads another context's database tables directly, even though the physical database
is shared. Every relationship above goes through an API call, a token claim, or a domain event.

---

## 4. Core Domain, Supporting, Generic

| Type | Description | Investment | Example |
|------|-------------|-----------|---------|
| **Core Domain** | Where the business competitive advantage lies. What differentiates us. | MAXIMUM — build, don't buy | Matching algorithm |
| **Supporting Subdomain** | Necessary for the core but not differentiating. Can be outsourced. | MEDIUM | Order management |
| **Generic Subdomain** | Commodity. Off-the-shelf solution exists. | MINIMUM — buy/use OSS | Authentication, emails |

### Classification of this project's bounded contexts

| Bounded Context | Type | Justification |
|-----------------|------|----------------|
| Monitoring and Measurements | Core | This is the reason the system exists: turning raw electrical readings into consumption attributed to a specific appliance, automatically. Nothing about this is off-the-shelf. |
| Alerts | Core | Timely, actionable feedback while there is still time to act is the value proposition described in `01-context/overview.md` — a generic utility bill cannot do this. |
| Recommendations | Core | Suggestions based on a home's own measured behavior, not generic advice, is part of the same value proposition as Alerts. |
| Notification | Core | Delivery of time-sensitive alerts and recommendations via email/push is integral to the value proposition; without it, alerts/recommendations are not actionable. |
| Devices | Supporting | Necessary so a measurement can be trusted and attributed correctly, but registering a device is not itself what the user is paying attention to. |
| Home Management | Supporting | Necessary organizational scaffolding — grouping devices and people — but not differentiating on its own. |
| Security | Generic | Identity, authentication, and RBAC are solved problems; the stack justification in `01-context/overview.md` already leans on Spring Security and OAuth2 instead of building this from scratch. |

---

## 5. Modeling decisions

No formal, facilitated Event Storming session was run with sticky notes. The boundaries above
were derived directly from the project's existing class diagram, the entity-relationship model,
and the domain description already produced before this documentation effort began — that
material had already gone through several iterations (see the security module redesign in the
project's own commit history) before it was translated into this format.

### Key decisions and discarded alternatives

| Decision | Discarded alternative | Reason |
|----------|------------------------|--------|
| Keep Security as a single bounded context covering identity, sessions, and authorization | Split into separate Identity, Session, and Authorization contexts | These responsibilities are tightly cohesive around one concern — access — and splitting them further would add inter-service calls for a two-person team without a clear business reason to deploy them independently |
| Keep Devices separate from Monitoring and Measurements | Merge them into one context, since a device's only purpose is to produce measurements | Device lifecycle (registration, connectivity, hardware association) changes independently from measurement ingestion, which is high-volume and append-only. They also fail independently: a device can go offline without any measurement being lost, and vice versa |
| Keep Alerts and Recommendations as two contexts instead of one shared "Notifications" context | Merge them, since both react to the same measurement stream | They have different lifecycles (`PENDING`/`RESOLVED` vs. `READ`/`UNREAD`) and are triggered by different logic — a threshold crossing versus a behavioral pattern. `Notification` remains a separate context for the delivery infrastructure (email/push), not the business logic of alerting or recommending |
| Separate `HomeThresholds` as its own aggregate (1:1 with `Home`) | Embed thresholds inside `Home` aggregate | Thresholds have their own lifecycle invariants (HOME-INV-005/006/007) and persistence row. The use case `UpdateHomeThresholdsService` orchestrates both aggregates. |
| Make `DeviceConnectivityRestored` an explicit domain event | Only have `DeviceConnectivityLost` and rely on manual resolution | The firmware and backend detect reconnection automatically; publishing a symmetric event enables auto-resolution of connectivity alerts and avoids manual intervention. |
| Use `messageKey` instead of storing message text in Alert/Recommendation/Notification | Store literal message text in each entity | Enables i18n without schema changes; the same alert/recommendation can be rendered in any supported language. The message catalog lives in the frontends. |

---

## 6. Discrepancies found vs. reference documentation

| Reference doc / prior version | Implementation reality | Resolution |
|------------------------------|------------------------|------------|
| `domain-map.md` listed 6 bounded contexts | Backend has 7 modules; `Notification` was missing | Added Notification as 7th bounded context |
| `domain-map.md` showed Kafka for domain events | Current implementation uses Spring Modulith in-process events; Kafka is planned for microservice extraction | Updated diagram and table to reflect Spring Modulith; noted Kafka as future transport |
| Kafka as the future transport (`ADR-002`) | Since the extractions of October 2026 events between services go over RabbitMQ with a transactional outbox (`ADR-008`); Spring events remain only inside the backend | Relationships table updated (24-Oct-2026) |
| Every context mapped to its own `*-service` | Security, Devices and Monitoring/Measurements are microservices; Home Management, Alerts, Recommendations and Notification still run in `energy-monitor-backend` | "Microservice(s)" rows and migration notes updated (24-Oct-2026) |
| `consumption_level` listed under Alerts | Owned by the measurement service (`measurement-002`) | Ownership corrected in `06-data/models.md` |
| `domain-events.md` listed only 2 events | Code has 7+ domain events across all contexts | Added all events: `MeasurementRecorded`, `DeviceConnectivityLost`, `DeviceConnectivityRestored`, `DeviceLinked`, `DeviceUnlinked`, `DeviceRegistered`, `AlertRaised`, `RecommendationCreated`, `AccountDeleted` |
| `domain-map.md` showed `Devices` → `Alerts` via Kafka | Implementation uses Spring Modulith events | Updated to Spring Modulith |
| `entities-and-rules.md` showed `HomeThresholds` as VO inside `Home` | Code has `HomeThresholds` as separate aggregate with own ID | Corrected to separate aggregate; updated invariants |
| `entities-and-rules.md` listed only 2 alert types | Code has 4: THRESHOLD, CONNECTIVITY, DEVICE, LIMIT | Added all 4 types with factory methods |
| `entities-and-rules.md` missing `Notification` aggregate | Code has full `Notification` aggregate with delivery lifecycle | Added Notification entity and VOs |
| Reference diagrams showed `RiskConsumption` enum | Code uses `RiskConsumption` in `measurement.api` and `ConsumptionLevel` entity | Aligned terminology |
| Reference showed `NotificationPreference` with email/push | Code has `NotificationPreference` but also `PushSubscription` | Added both; clarified `NotificationPreference` is per-user, `PushSubscription` per browser installation |

---

## 7. How to update this map

1. Before adding a new microservice, verify whether it belongs to an existing bounded context.
2. If a context's ubiquitous language is changing, review whether the context should be split.
3. Revisit this map whenever the domain changes significantly — a new context, a merged one, or a
   relationship that no longer holds.
4. The context map MUST stay synchronized with the C4 system-level diagram (`05-architecture/overview.md`).

> **Important correlation:** The bounded contexts in this document →
> Microservices in `09-microservices/service-catalog.md` →
> C4 diagrams in `08-uml/` →
> Service separation ADRs in `05-architecture/decisions/`

---

## 8. Migration notes: from modular monolith to microservices

Every bounded context started as a Spring Modulith module of `energy-monitor-backend`. Status on
24-Oct-2026:

| Context | Status | Data ownership | Async communication | Sync communication | Notes |
|---------|--------|----------------|---------------------|--------------------|-------|
| **Security** | ✅ Extracted — `en-monitor-security-service` (:8082), in production | `person`, `user`, `user_configuration`, `user_session`, `refresh_token`, `password_reset_token`, `password_policy`, `security_configuration`, `login_error_log`, RBAC tables, `audit_log`, `security_outbox` | Publishes `account.*`, `mail.*` (`energy-monitor.security`) | Serves `/internal/users/{id}/permissions`, `/internal/users/profiles`, `/internal/administrators`; calls `/internal/accounts/{id}/deletion-check` | Only holder of the JWT private key; the backend keeps a `security` proxy package |
| **Devices** | ✅ Extracted — `en-monitor-device-service` (:8084), in production | `appliance_type`, `device`, `device_home`, `device_status_log`, `device_outbox` | Publishes `device.*` (`energy-monitor.device`); consumes `home.deleted` | Serves `/internal/devices/**`, `/internal/homes/{homeId}/devices`, `/internal/mqtt/auth/**`; calls `/internal/homes/{homeId}/access` | Owns MQTT authentication, commands and connectivity; the backend keeps a cached `device` proxy |
| **Monitoring and Measurements** | ✅ Extracted — `en-monitor-measurement-service` (:8085), in production | `measurement`, `consumption_level`, `measurement_outbox` | Publishes `measurement.recorded` (`energy-monitor.measurement`); consumes `device.linked/unlinked` | Serves `/internal/consumption-levels`, `/internal/measurements/**`; calls the backend (access, thresholds) and the device service | Only subscriber that stores telemetry; the backend keeps a `measurement` proxy and is no longer an MQTT client |
| **Home Management** | 🟡 In the monolith — migration in progress (teammate) | `home`, `home_type`, `home_thresholds`, `user_home`, `home_outbox` | Publishes `home.deleted` (`energy-monitor.home`); consumes `account.deleted` | Serves `/internal/homes/{homeId}/access`, `/internal/homes/{homeId}/thresholds`, `/internal/accounts/{id}/deletion-check` | — |
| **Alerts** | 🟡 In the monolith — migration in progress (teammate) | `alert` | Consumes `measurement.recorded`, `device.connectivity-*` (through the proxies) | Reads levels and hourly energy from the measurement service | — |
| **Recommendations** | 🟡 In the monolith — migration in progress (teammate) | `recommendation` | — | Reads hourly energy and reporting devices from the measurement service, devices from the device service | — |
| **Notification** | 🟡 In the monolith — migration in progress (teammate) | `notification`, `notification_preference`, `push_subscription`, `push_device` | Consumes `account.registered`, `account.administered`, `mail.*` | Reads profiles and administrators from the security service | Only component that sends e-mail; Web Push (VAPID) and Expo push |

The four remaining contexts are being migrated by Miguel Ángel García Artunduaga; their service
names, ports and order are not defined yet.

**How the three extractions were done** (the pattern the remaining ones can follow — see the
`docs/Architecture.md` of each service repository):
1. Move the module to its own repository with its package, tests and Liquibase changelogs, keeping
   the changelog paths unchanged (`ADR-009`).
2. Add RabbitMQ with a transactional outbox for what other contexts must hear (`ADR-008`).
3. Expose `/internal/**` endpoints for what other contexts must ask, protected by network and
   `X-Internal-Token`.
4. Remove the module from `energy-monitor-backend`, leaving a proxy package that keeps its `api`
   (HTTP client with cache, events republished in memory).
5. Add the gateway routes (most specific first) so the clients keep calling the same paths.
6. Add the container to the infra Compose files, its CI/CD workflow and its `deploy.sh` target;
   check the whole stack end to end.
