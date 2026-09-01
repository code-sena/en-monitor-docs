# Project Glossary

> **Instructions:** Define here all technical and business terms used in the project.
> This is the official dictionary — if there is ambiguity, this document wins.
> Add terms throughout the project, not only at the start.

---

## How to use this glossary

1. Before using a technical or business term in code, docs, or conversations: look it up here.
2. If it's not there: add it with its definition.
3. If there is disagreement about the definition: discuss it as a team and update this document.
4. This document defines **what a term means**, not how it is stored or structured.
   Attributes, cardinalities, and allowed enumeration values belong to the data model and the
   domain rules, not here.
---

## Domain terms — Identity and access

| Term | Definition | Notes |
|------|-----------|-------|
| Person | Personal information of a human being: name, contact details, profile image. Holds no credentials. | A person never authenticates by itself. |
| User | System account that owns credentials, account status, and login history. | Do not merge with `Person` in code. |
| User Configuration | A user's personal preferences: notification channels, color theme, language, social provider. | Not security configuration. See `Security Configuration`. |
| Credentials | Email plus password hash used to prove identity. | Passwords are never stored in plain text, only as a hash. |
| Session | Persisted record of an authenticated login, which can be inspected and revoked. | A user may hold several simultaneous sessions. |
| Access Token | Short-lived token sent on every request to prove identity and carry claims. | Stateless. Not persisted. |
| Refresh Token | Long-lived token bound to a persisted `Session`, used to obtain a new access token. | Revocable. Not merely a long-lived access token. |
| Password Reset Token | Single-use, time-limited token that authorizes changing a forgotten password. | Invalidated after use. |
| Password Policy | Configurable rules a password must satisfy before it is accepted. | Applied at registration and at password reset. |
| Security Configuration | System-wide settings that govern security behavior, such as lockout rules and session duration. | Global scope, not per user. |
| Account Lockout | Automatic blocking of an account after repeated failed authentication attempts. | The threshold is a `Security Configuration` value. |
| System Role | Global role that groups permissions across the whole platform. | Do **not** confuse with `Home Role`. |
| Permission | A single authorizable action, identified by a stable code. | Granted to roles, never directly to users. |
| RBAC | Authorization model used in this project: permissions are granted to system roles, and system roles are assigned to users. | A user's effective permissions are the union of those of their roles. |
| Audit Log | Immutable record of a relevant action: what happened, who did it, when, from where, and in which application. | Written, never updated. |

---

## Domain terms — Homes, devices, and consumption

| Term | Definition | Notes |
|------|-----------|-------|
| Home | Physical dwelling that groups devices, members, thresholds, alerts, and recommendations. The main aggregation unit of consumption. | Belongs to `Home Management`, not to `Security`. |
| Home Type | Category of dwelling used to classify a home. | Reference data. |
| Access Code | Short code that lets a user join an existing home. | Belongs to the home, not to the user. |
| Home Role | Role a user holds **inside a specific home**. | Scoped to one home. Do **not** confuse with `System Role`. |
| Device | Physical measuring unit installed on an electrical load. It is the thing that measures. | Authenticates against the backend with its own key, not with user credentials. |
| Appliance | The electrical load being measured: refrigerator, air conditioner, lighting circuit. It is the thing being measured. | Not modeled as an entity; classified through `Appliance Type`. |
| Appliance Type | Category of appliance used to describe what a device monitors. | Reference data. |
| Device Status | Connectivity state of a device, derived from whether it is still reporting. | Not reported by the user; inferred by the system. |
| Measurement | A single electrical reading taken by a device at a point in time. | Immutable once stored. The atomic unit of the whole system. |
| Voltage | Electrical potential difference at the load, in volts (V). | |
| Current | Electrical current drawn by the load, in amperes (A). | |
| Active Power | Instantaneous real power consumed by the load, in watts (W). | A rate, not an amount. Do **not** confuse with `Energy`. |
| Energy | Accumulated electrical consumption over time, in kilowatt-hours (kWh). | This is what appears on a utility bill. |
| Power Factor | Ratio between active power and apparent power. | Reported by the sensor; whether it is persisted is a data model decision. |
| Consumption | Business reading of accumulated energy over a period, attributed to a home or a device. | Derived from measurements; not stored as an entity. |
| Threshold | Consumption limit configured for a home, expressed in energy. | A home may define its own limits or fall back to system defaults. |
| Consumption Level | Named band that classifies how high a consumption value is, defined by a numeric range. | A classification, not an alert. |
| Alert | Notification that something requires the user's attention: an exceeded threshold, or a device that stopped reporting. | Has a lifecycle: it is raised and later resolved. |
| Alert Severity | How serious an alert is. **Derived at runtime, never persisted.** | Computed from the consumption level, or from the connectivity rule. |
| Recommendation | Suggested action generated from consumption behavior to help the user reduce usage. | Informative, never blocking. |
| Notification | Conceptual umbrella covering both `Alert` and `Recommendation`. | Not a persisted entity. |
| Message Key | Identifier stored in place of literal message text, resolved to the user's language at presentation time. | Translated text is never stored in the database. |

---

## Terms that must not be confused

| These two | Are not the same because... |
|-----------|-----------------------------|
| `Device` vs `Appliance` | The device measures; the appliance is measured. |
| `System Role` vs `Home Role` | `System Role` is global and belongs to `Security`. `Home Role` is scoped to one home and belongs to `Home Management`. |
| `Person` vs `User` | `Person` is who someone is; `User` is the account they log in with. |
| `Active Power` (W) vs `Energy` (kWh) | Power is a rate at an instant; energy is an accumulated amount over time. Thresholds are expressed in energy. |
| `Access Token` vs `Refresh Token` vs `Session` | The access token proves identity per request, the refresh token renews it, the session is the persisted and revocable record backing both. |
| `Alert` vs `Recommendation` | An alert reports a problem that already happened. A recommendation proposes an improvement. |
| Authentication vs Authorization | Authentication answers *who is the user*. Authorization answers *what may the user do*. Both belong to `Security`. |
| Domain model vs Persistence entity | The domain model holds business rules and knows nothing about the database. The persistence entity maps a table. They are deliberately separate. |
| `Consumption Level` vs `Alert Severity` | The consumption level classifies a measurement. The severity qualifies the resulting alert and is derived from it. |
| Entity vs Microservice | A service boundary is defined by cohesion and business responsibility, not by tables. One service owns many entities. |

---

## Hardware and communication terms

| Term | Definition |
|------|-----------|
| ESP32 | Microcontroller with Wi-Fi that reads the sensor and publishes measurements to the backend. |
| PZEM-004T V3 | Electrical sensor module that measures voltage, current, active power, energy, frequency, and power factor. |
| HLK-PM03 | AC/DC power supply module that powers the measuring device from mains voltage. |
| UART | Serial protocol used between the sensor and the microcontroller. |
| MQTT | Lightweight publish/subscribe protocol used to transport measurements from devices to the backend. |
| MQTT Broker | Intermediary that receives published messages and routes them to subscribers, decoupling devices from the backend. |
| Topic | Hierarchical channel name a device publishes to and a service subscribes to. |
| API Key | Secret assigned to a device so it can authenticate when reporting measurements. Not a user credential. |

---

## Technical terms of the project

| Term | Definition |
|------|-----------|
| Microservice | Independently deployed service that owns one bounded context, with its own repository, lifecycle, and CI/CD. **In this project it does not imply a dedicated database** — see `Shared Database`. |
| Bounded Context | Boundary within which a particular domain model has consistent meaning. `Security`, `Home Management`, `Devices`, `Monitoring and Measurements`, `Alerts`, and `Recommendations` are separate contexts. |
| Shared Database | Current decision: all services connect to a single physical database, organized logically by domain. Schema migrations live in a separate repository. |
| Distributed Monolith | Anti-pattern this project explicitly avoids: services that share a database also reaching into each other's tables or entities instead of talking through contracts. |
| Hexagonal Architecture | Architectural style, also called Ports and Adapters, where the domain is isolated from frameworks and all external interaction happens through explicit ports implemented by adapters. |
| Port | Interface the application layer depends on. Inbound ports expose use cases; outbound ports declare capabilities the application needs from the outside. |
| Adapter | Concrete implementation of a port. Inbound adapters receive external requests; outbound adapters reach persistence, security, or external clients. |
| Use Case | Application-layer coordinator of a single meaningful business operation. Not a wrapper around one entity method. |
| Domain Model | Framework-free class holding business state and rules. Does not depend on the web, the database, or any framework. |
| Persistence Entity | Class that maps a database table. Lives in the persistence adapter, never in the domain. |
| Mapper | Component that translates between domain models and persistence entities, keeping storage details out of the domain. |
| Value Object | Immutable type defined by its value rather than by an identity. |
| Domain Event | A fact that occurred in the business that other services can observe. Name always in past tense. |
| API Gateway | Single entry point to the system that routes requests to the corresponding services. |
| Observer | Pattern used so that a new measurement notifies subscribers without coupling ingestion to the reaction it triggers. |
| Factory Method | Pattern used to build notifications from measurements or device status without the caller knowing the concrete type. |
| Singleton | Pattern used to expose system-wide default values from a single instance. |
| Idempotence | Property of an operation to produce the same result if executed multiple times. Required for measurement ingestion, since MQTT may redeliver. |
| DTO | Data Transfer Object used at the web boundary. Domain models and persistence entities are never exposed directly. |

---

## Acronyms

| Acronym | Meaning |
|---------|---------|
| ADR | Architecture Decision Record |
| API | Application Programming Interface |
| CI/CD | Continuous Integration / Continuous Delivery |
| CRUD | Create, Read, Update, Delete |
| DoD | Definition of Done |
| DoR | Definition of Ready |
| DTO | Data Transfer Object |
| ERD | Entity Relationship Diagram |
| FR | Functional Requirement |
| IAM | Identity and Access Management |
| JWT | JSON Web Token |
| MQTT | Message Queuing Telemetry Transport |
| NFR | Non-Functional Requirement |
| PR | Pull Request |
| RBAC | Role-Based Access Control |
| SLA | Service Level Agreement |
| SLO | Service Level Objective |
| SRS | Software Requirements Specification |
| UART | Universal Asynchronous Receiver-Transmitter |
| kWh | Kilowatt-hour — unit of accumulated energy |
| W | Watt — unit of active power |

---

