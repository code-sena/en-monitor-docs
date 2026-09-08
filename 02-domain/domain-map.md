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

---

### Bounded Context: Home Management

| Field | Value |
|-------|-------|
| **Name** | Home Management |
| **Responsibility** | Groups devices, members, and thresholds under a home, and manages who belongs to which home and with what role |
| **Owning team** | Miguel Garcia |
| **Microservice(s)** | `home-service` |
| **Database** | Shared MySQL instance; owns the tables under the home domain |
| **Ubiquitous Language** | `Home`, `Home Type`, `Access Code`, `Home Role`, `Threshold`. Full definitions in `01-context/glossary.md`. |

**Context-specific terms:**

| Term | Meaning in THIS context | Different in another context? |
|------|--------------------------|-------------------------------|
| Role | `Home Role` — scoped to one specific home (`OWNER`/`MEMBER`) | Yes — in Security, "role" means `System Role`, global to the platform |

---

### Bounded Context: Devices

| Field | Value |
|-------|-------|
| **Name** | Devices |
| **Responsibility** | Registers measuring devices, associates each one with a home, authenticates them, and tracks their connectivity |
| **Owning team** | Juan Cortes |
| **Microservice(s)** | `device-service` |
| **Database** | Shared MySQL instance; owns the tables under the device domain |
| **Ubiquitous Language** | `Device`, `Appliance`, `Appliance Type`, `Device Status`, `API Key`. Full definitions in `01-context/glossary.md`. |

**Context-specific terms:**

| Term | Meaning in THIS context | Different in another context? |
|------|--------------------------|-------------------------------|
| Device | The physical measuring unit — the thing that measures | Distinct from `Appliance`, which is the load being measured. Both terms are defined here to be used consistently everywhere else. |

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
| **Ubiquitous Language** | `Alert`, `Alert Severity`, `Consumption Level` (consumed, not owned). Full definitions in `01-context/glossary.md`. |

---

### Bounded Context: Recommendations

| Field | Value |
|-------|-------|
| **Name** | Recommendations |
| **Responsibility** | Generates suggestions from a home's own measured consumption behavior, and tracks whether the user has read them |
| **Owning team** | Juan Cortes |
| **Microservice(s)** | `recommendation-service` |
| **Database** | Shared MySQL instance; owns the tables under the recommendation domain |
| **Ubiquitous Language** | `Recommendation`, `Message Key`. Full definitions in `01-context/glossary.md`. |

---

## 3. Context Map

```
┌────────────┐   OHS / PL    ┌──────────────────┐
│  Security  │──────────────▶│  Home Management │
│ (identity) │               └──────────────────┘
│            │                         │
│            │   OHS / PL              │  U → D
│            │──────────────▶  ┌───────▼────────┐
│            │                 │    Devices     │
└─────┬──────┘                 └───────┬────────┘
      │ OHS / PL                       │ U → D
      │                                 ▼
      │                    ┌─────────────────────────┐
      └───────────────────▶│ Monitoring and          │
                            │ Measurements            │
                            └──────────┬──────────────┘
                                       │
                                       │    
                                       │                
                          ┌────────────┴────────────┐
                          ▼                         ▼
                    ┌──────────┐            ┌────────────────┐
                    │  Alerts  │            │ Recommendations│
                    └──────────┘            └────────────────┘
```

### Context relationship types

| Type | Symbol | Description | Example |
|------|--------|-------------|---------|
| **Upstream → Downstream** | `U → D` | U provides, D consumes. D depends on U. | Devices → Measurements |
| **Open Host Service** | `OHS` | U publishes a stable, well-defined contract that many contexts can consume | Security → everyone else |
| **Published Language** | `PL` | The explicit shared contract itself (a claim format, an event schema) | JWT claims, domain events |

This project does not currently use Shared Kernel, Conformist, or Anti-Corruption Layer
relationships — there is no legacy system to protect against, and no two contexts share a model.

### Relationships table

| Context A | Relationship | Context B | Communication channel | Contract |
|-----------|-------------|-----------|------------------------|---------|
| Security | OHS / PL | Home Management | REST | Access token claims (user identity) |
| Security | OHS / PL | Devices | REST | Access token claims (user identity) |
| Security | OHS / PL | Monitoring and Measurements | REST | Access token claims (user identity) |
| Home Management | U → D | Devices | REST | `homeId` as a foreign reference, not a shared entity |
| Devices | U → D | Monitoring and Measurements | MQTT + REST | `deviceId` validated through its `apiKey` before a measurement is accepted |
| Monitoring and Measurements | U → D | Alerts | Domain event (name and payload to be defined in `domain-events.md`) | Not yet defined |
| Monitoring and Measurements | U → D | Recommendations | Domain event (name and payload to be defined in `domain-events.md`) | Not yet defined |

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
| Keep Alerts and Recommendations as two contexts instead of one shared "Notifications" context | Merge them, since both react to the same measurement stream | They have different lifecycles (`PENDING`/`RESOLVED` vs. `READ`/`UNREAD`) and are triggered by different logic — a threshold crossing versus a behavioral pattern. `Notification` remains a conceptual umbrella in the glossary, not a bounded context |

---

## 6. How to update this map

1. Before adding a new microservice, verify whether it belongs to an existing bounded context.
2. If a context's ubiquitous language is changing, review whether the context should be split.
3. Revisit this map whenever the domain changes significantly — a new context, a merged one, or a
   relationship that no longer holds.
4. The context map MUST stay synchronized with the C4 system-level diagram (`05-architecture/overview.md`).

> **Important correlation:** The bounded contexts in this document →
> Microservices in `09-microservices/service-catalog.md` →
> C4 diagrams in `08-uml/` →
> Service separation ADRs in `05-architecture/decisions/`