# System Scope

> **Why this document exists:** Scope prevents scope creep and aligns expectations.
> It is equally important to define what the system does NOT do as what it does.
> Review this document at the start of each planning cycle.

---

## In Scope

What the system **DOES build and maintain**:

### MVP Features

| # | Feature | Description | Responsible context |
|---|---------|-------------|---------------------|
| 1 | Identity and authentication | User registration, login, logout, password recovery, session management with access and refresh tokens | Security |
| 2 | Authorization (RBAC) | System roles and permissions, assignable to users, enforced on every protected endpoint | Security |
| 3 | Security administration | Password policy, security configuration, login error tracking, account lockout | Security |
| 4 | Audit trail | Immutable log of security-relevant actions: who did what, when, from where | Security |
| 5 | Home management | Create a home, join it with an access code, manage members and their home role | Home Management |
| 6 | Device registration | Register a measuring device, associate it with a home, authenticate it with an API key | Devices |
| 7 | Automatic measurement ingestion | Devices publish voltage, current, active power, and energy readings over MQTT without manual entry | Monitoring and Measurements |
| 8 | Consumption history and statistics | Consumption viewed over time, per home and per device | Monitoring and Measurements |
| 9 | Threshold-based alerts | Notify when consumption approaches or exceeds a configured limit, or when a device stops reporting | Alerts |
| 10 | Recommendations | Suggestions generated from a home's own measured consumption behavior | Recommendations |
| 11 | MQTT broker and ingestion design | Selection and configuration of the MQTT broker that receives device publications, plus the topic and payload contract between devices and the backend. Not yet designed — only the measuring hardware components are currently in hand | Monitoring and Measurements |
| 12 | Web application | Full-featured client: management, detailed history, statistics | Frontend (React) |
| 13 | Android application | Consultation and alerts client for daily use | Mobile (React Native) |
| 14 | Measuring device (hardware) | ESP32 + PZEM-004T V3 unit that performs real electrical measurement, not estimation | Hardware |
| 15 | Push notifications | Alerts and recommendations are delivered through an external push channel, in addition to being surfaced inside the web and Android clients | Alerts / Recommendations |
| 16 | Internationalized interface | The interface is built with `react-i18next` and already supports more than one language | Frontend (React) |

### Included integrations

| External component | Integration type | Purpose |
|--------------------|-------------------|---------|
| ESP32 + PZEM-004T V3 | MQTT (publish) | The device publishes measurements to the backend; the backend never calls the device |

### Environments being built

| Environment | Purpose |
|-------------|---------|
| Local | Development on each developer's machine, including a locally connected or simulated measuring device |
| Development (dev) | Integration testing between services before merging to `main` |
| Production | The environment the finished project is evaluated and demonstrated in |

A dedicated staging environment is not planned given the size of the team; `dev` absorbs that
role for this project.

---

## Out of Scope

What the system **does NOT build** in this version and why:

| # | What is out of scope | Reason | Future version? |
|---|---------------------|--------|----------------|
| 1 | iOS application | The mobile app is built with React Native but targets Android only; no iOS build, signing, or App Store distribution is planned | N/A — not planned |
| 2 | Manual consumption entry from the utility bill | Superseded by automatic MQTT ingestion once the hardware was defined. Appears in the original SRS as the input method, but that description is historical and does not reflect the current architecture | N/A — superseded, not deferred |
| 3 | Dedicated administration UI for Security | The administrative use cases (role/permission assignment, security configuration, password policy) are exposed as backend endpoints and operated directly during this phase; no screens are built for them | Possible — depends on whether a non-technical administrator is ever needed |
| 4 | Physically separate database per service | All services currently share one physical MySQL instance, organized logically by domain, per the project's current architectural decision | Possible — see `05-architecture/decisions/` |
| 5 | Energy cost estimation in local currency | The project has no agreement with any utility provider, so there is no tariff data source to calculate a monetary cost from. The system reports energy (kWh), not billed cost | N/A — blocked on an external agreement that does not exist |

### What another system / team handles (and why not us)

This project has no functional dependency on another system or team. Other SENA groups are
building their own, unrelated projects; nothing they produce is consumed by, or required for,
Energy Monitor.

---

## Scope assumptions

> These assumptions are taken to be true. If they change, the scope must be renegotiated.

| # | Assumption | Consequence if false |
|---|-----------|---------------------|
| 1 | All services can reach one shared MySQL instance over the network during development and evaluation | The shared-database decision in the glossary and ADRs would need to be revisited |
| 2 | The team has continuous Wi-Fi connectivity available where devices are installed | MQTT ingestion could not rely on real-time delivery; offline buffering on the device would become mandatory rather than optional |
| 3 | End users access the system on modern browsers and Android versions actively supported by React Native | Compatibility work for older platforms would be added |

---

## Constraints

| Type | Description |
|------|-------------|
| **Time** | The project must be delivered by the last week of October 2026, per the SENA ADSO schedule for group 314556 |
| **Budget** | Approximately COP $180,000 already invested in hardware components (ESP32, PZEM-004T V3, HLK-PM03, and related parts). No budget is currently allocated for paid hosting, cloud infrastructure, or other paid third-party services |
| **Technology** | Fixed stack: Java 21 + Spring Boot (backend), React + Vite (web), React Native for Android (mobile), MySQL (database), ESP32 + PZEM-004T V3 (hardware) |
| **Regulatory** | None identified beyond standard handling of personal data (names, contact details, addresses) collected for `Person` and `Home` |
| **Team** | Two developers: Juan Esteban Cortés Parra and Miguel Ángel García Artunduaga |

---

## External dependencies

| Dependency | Team / Provider | Required date | Status |
|-----------|----------------|--------------|--------|
| Database migrations repository (Liquibase, with per-domain changelogs) | Internal, separate repo | Needed before any service connects to a real schema | 🟡 In progress |
| PZEM-004T V3 hardware units | Internal procurement | September 15, 2026 | 🟡 In progress |

---

## How to update the scope

The scope can change, but the change has a process:

1. Document the proposed change in this file.
2. Evaluate the impact on the current milestone and on the affected contexts.
3. Get agreement from both developers and, where relevant, the instructor.
4. Update the roadmap, if one exists, in `03-product/vision.md`.
5. Create or update user stories in `04-requirements/user-stories.md`.

---

## Correlations

- Vision and roadmap → `03-product/vision.md`
- Term glossary → `01-context/glossary.md`
- System overview → `01-context/overview.md`
- Scope-related risks → `15-project-control/risks.md`