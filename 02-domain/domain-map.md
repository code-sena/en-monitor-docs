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
| **Microservice(s)** | `security-service` |
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
| **Microservice(s)** | `home-service` |
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
| **Microservice(s)** | `device-service` |
| **Database** | Shared MySQL instance; owns the tables under the device domain |
| **Ubiquitous Language** | `Device`, `Appliance`, `Appliance Type`, `Device Status`, `API Key`, `Device Code`. Full definitions in `01-context/glossary.md`. |

**Context-specific terms:**

| Term | Meaning in THIS context | Different in another context? |
|------|--------------------------|-------------------------------|
| Device | The physical measuring unit — the thing that measures | Distinct from `Appliance`, which is the load being measured |
| API Key | Secret assigned to a device to authenticate it against the backend | Not a user credential; validated by the broker via HTTP callback to the backend |

---

### Bounded Context: Monitoring and Measurements

| Field | Value |
|-------|-------|
| **Name** | Monitoring and Measurements |
| **Responsibility** | Receives readings published by devices over MQTT, stores them, and makes consumption history and statistics available |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | `measurement-service` |
| **Database** | Shared MySQL instance; owns the tables under the measurement domain |
| **Ubiquitous Language** | `Measurement`, `Voltage`, `Current`, `Active Power`, `Energy`, `Power Factor`, `Consumption`, `Threshold`, `Consumption Level`. Full definitions in `01-context/glossary.md`. |

---

### Bounded Context: Alerts

| Field | Value |
|-------|-------|
| **Name** | Alerts |
| **Responsibility** | Reacts to a measurement or a device going silent by raising and tracking an alert until it is resolved |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | `alert-service` |
| **Database** | Shared MySQL instance; owns the tables under the alerting domain |
| **Ubiquitous Language** | `Alert`, `Alert Severity`, `Alert Type` (THRESHOLD, CONNECTIVITY, DEVICE, LIMIT), `Consumption Level` (consumed, not owned). Full definitions in `01-context/glossary.md`. |

---

### Bounded Context: Recommendations

| Field | Value |
|-------|-------|
| **Name** | Recommendations |
| **Responsibility** | Generates suggestions from a home's own measured consumption behavior, and tracks whether the user has read them |
| **Owning team** | Juan Cortes |
| **Microservice(s)** | `recommendation-service` |
| **Database** | Shared MySQL instance; owns the tables under the recommendation domain |
| **Ubiquitous Language** | `Recommendation`, `Recommendation Type`, `Message Key`. Full definitions in `01-context/glossary.md`. |

---

### Bounded Context: Notification

| Field | Value |
|-------|-------|
| **Name** | Notification |
| **Responsibility** | Owns the delivery log: a record that an outbound message was attempted, over which channel, and whether it succeeded. Delivers alerts, recommendations, and password reset codes via email and Web Push. |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | `notification-service` |
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
                                      │    U → D (Spring Modulith event)
                          ┌───────────┴───────────┐
                          ▼                       ▼
                    ┌──────────┐            ┌────────────────┐
                    │  Alerts  │            │Recommendations │
                    └────┬─────┘            └───────┬────────┘
                         │                         │
                         │  U → D (Spring Modulith event)
                         ▼                         ▼
                    ┌────────────────────────────────────┐
                    │         Notification               │
                    │  (delivers alerts, recommendations,│
                    │   password reset via email/push)   │
                    └────────────────────────────────────┘
```

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

| Context A | Relationship | Context B | Communication channel | Contract |
|-----------|-------------|-----------|------------------------|---------|
| Security | OHS / PL | Home Management | REST (UserDirectoryPort) | Access token claims (user identity); user profile query |
| Security | OHS / PL | Devices | REST (via broker auth) | Access token claims; device credentials validated by backend |
| Security | OHS / PL | Monitoring and Measurements | REST | Access token claims (user identity) |
| Security | OHS / PL | Alerts | REST | Access token claims (user identity) |
| Security | OHS / PL | Recommendations | REST | Access token claims (user identity) |
| Security | OHS / PL | Notification | REST (PasswordResetDeliveryPort) | Access token claims; password reset delivery port |
| Home Management | U → D | Devices | REST (HomeLookupPort) | `homeId` validated before device linking |
| Devices | U → D | Monitoring and Measurements | MQTT (device→backend) + REST | `deviceId` validated through its `apiKey` before a measurement is accepted |
| Monitoring and Measurements | U → D | Alerts | Spring Modulith event (`MeasurementRecorded`) | See `domain-events.md` |
| Monitoring and Measurements | U → D | Recommendations | Spring Modulith event (`MeasurementRecorded`) | See `domain-events.md` |
| Devices | U → D | Alerts | Spring Modulith event (`DeviceConnectivityLost`, `DeviceConnectivityRestored`) | See `domain-events.md` |
| Home Management | U → D | Alerts | REST (HomeLookupPort, ConsumptionLimitPort) | `homeId` as a foreign reference |
| Home Management | U → D | Recommendations | REST (HomeReaderPort) | `homeId` as a foreign reference |
| Devices | U → D | Recommendations | REST (DeviceLookupPort) | `deviceId` as a foreign reference |
| Alerts | U → D | Notification | Spring Modulith event (`AlertRaised`) | See `domain-events.md` |
| Recommendations | U → D | Notification | Spring Modulith event (`RecommendationCreated`) | See `domain-events.md` |
| Security | U → D | Notification | Spring Modulith event (`AccountDeleted`) | See `domain-events.md` |
| Home Management | ACL | Security | UserDirectoryPort (outbound port) | Translates Security's `User` to Home's `UserProfile` |

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

Each bounded context above is implemented as a Spring Modulith module with explicit allowed
dependencies. To extract a module as an independently deployable microservice:

| Context | Data ownership | Async communication to extract | Sync dependencies to break | Shared infrastructure to separate |
|---------|----------------|-------------------------------|---------------------------|-----------------------------------|
| **Security** | `user`, `person`, `session`, `audit_log`, `system_role`, `permission` tables | `AccountDeleted` event | `UserDirectoryPort` (called by Home, Notification) → expose REST/gRPC | JWT key pair; password reset pepper |
| **Home Management** | `home`, `home_thresholds`, `user_home`, `home_type` tables | — | `HomeLookupPort` (called by Devices, Alerts, Recommendations) → expose REST | Access code generator |
| **Devices** | `device`, `device_home`, `device_status_log`, `appliance_type` tables | `DeviceConnectivityLost`, `DeviceConnectivityRestored`, `DeviceLinked`, `DeviceUnlinked`, `DeviceRegistered` | `HomeLookupPort` → expose REST; `DeviceCommandPort` (MQTT downlink) | MQTT broker auth callback (`/internal/mqtt/auth/*`) |
| **Monitoring and Measurements** | `measurement`, `consumption_level` tables | `MeasurementRecorded` | `ConsumptionLevelLookupPort` (called by Alerts) → expose REST | — |
| **Alerts** | `alert` table | `AlertRaised` | `HomeLookupPort`, `ConsumptionLevelLookupPort`, `ConsumptionLimitPort`, `DeviceLookupPort` → expose REST | — |
| **Recommendations** | `recommendation` table | `RecommendationCreated` | `HomeReaderPort`, `ConsumptionReaderPort`, `DeviceLookupPort` → expose REST | — |
| **Notification** | `notification`, `notification_preference`, `push_subscription` tables | — | `HomeDirectoryPort`, `PushSubscriptionPort`, `NotificationPreferencePort` → expose REST; `PasswordResetDeliveryPort` (called by Security) → expose REST | SMTP config; VAPID keys for Web Push |

**Extraction order recommendation:**
1. **Security** (upstream for all; stable contract via JWT claims)
2. **Devices** (produces `MeasurementRecorded` trigger; has MQTT auth dependency)
3. **Home Management** (referenced by many; stable `homeId` contract)
4. **Monitoring and Measurements** (high-volume append-only; core domain)
5. **Alerts** + **Recommendations** (consume `MeasurementRecorded`; can be extracted together or separately)
6. **Notification** (consumes `AlertRaised`, `RecommendationCreated`, `AccountDeleted`; has external SMTP/push deps)

Each extraction requires:
- Defining the OpenAPI contract for the service's REST endpoints
- Setting up the event schema (Avro/Protobuf) for async communication
- Configuring the service's own database (or schema) and migration strategy
- Implementing the Outbox pattern for reliable event publishing
- Updating the API Gateway routing