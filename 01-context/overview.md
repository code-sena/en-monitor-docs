# System Overview

## What is Energy Monitor?

**Energy Monitor** is a system that measures how much electricity each appliance or electrical load in a home actually consumes and turns those readings into information that the household can use to make informed decisions.

Measuring devices installed on individual electrical loads automatically report their readings. The system then makes this information available through a web application and a mobile application for Android. Users can view consumption history, identify unusual consumption, monitor consumption levels, and receive recommendations for reducing energy usage.

Energy Monitor is not only software: the measuring hardware is also part of the product.

## Problem it solves

**Before the system:** A household receives a single monthly electricity bill for the entire home. It shows a total amount of energy consumed, but does not indicate which appliance caused that consumption or when it occurred. Any attempt to reduce consumption is therefore based on guesswork. People may turn devices off without knowing whether they have a significant impact and only find out whether their actions worked thirty days later, when the next bill arrives.

**With the system:** Consumption is attributed to specific appliances or electrical loads and to specific moments in time. The household can identify which loads account for most of its consumption, receive alerts when consumption approaches a configured threshold, and receive recommendations based on its own measured consumption behavior rather than generic advice.

Feedback is provided while there is still time to act, rather than after the billing period has ended.

## Main users

| Role | Description | What they do in the system |
|------|-------------|----------------------------|
| Home Owner | The person who registers a home and is responsible for it | Creates the home, registers measuring devices, sets consumption thresholds, invites members, and manages the home configuration |
| Home Member | A person who belongs to a home they did not create | Consults consumption, history, and statistics; receives alerts and recommendations for that home |

A user may belong to several homes and hold a different role in each one.

The measuring device is not a user, but it does act on the system. It authenticates with its own key and publishes measurements without human intervention.

## Client applications

The system exposes two clients over the same public API, served through the API gateway. They are not equivalent: each one serves a different moment of use.

| Client | Platform | Purpose |
|--------|----------|---------|
| Web application | Browser | Full experience: home and device management, detailed history, statistics, and administration |
| Mobile application | Android | Day-to-day consultation and alerts, on the device the user already carries |

## Bounded contexts and where they run

The backend started as a **modular monolith** built with Spring Modulith, one module per bounded
context. Since 28-Oct-2026 every context runs as its own **microservice** behind an **API
gateway**; the monolith `energy-monitor-backend` was retired (the migration of the last four
contexts was carried out by Miguel Ángel García Artunduaga). Every context keeps its own domain
model and hexagonal architecture (ports & adapters).

| Context | Runs in | Responsibility | Key events published | Status (28-Oct-2026) |
|---------|---------|----------------|---------------------|---------------------|
| **Security** | `en-monitor-security-service` (:8082) | Accounts, e-mail verification, login, Google sign-in, sessions (JWT + refresh), password recovery, RBAC, audit trail, support contacts | `account.registered`, `account.administered`, `account.deleted`, mail requests (RabbitMQ) | Microservice, in production |
| **Devices** | `en-monitor-device-service` (:8084) | Device linking and credentials (API key), appliance catalogue, MQTT authentication and ACL, commands, connectivity tracking | `device.linked`, `device.unlinked`, `device.connectivity-lost`, `device.connectivity-restored` (RabbitMQ) | Microservice, in production |
| **Monitoring and Measurements** | `en-monitor-measurement-service` (:8085) | Measurement ingestion (MQTT), consumption history, statistics, home consumption, consumption levels | `measurement.recorded` (RabbitMQ) | Microservice, in production |
| **Home Management** | `en-monitor-home-service` (:8086) | Homes, membership, home-scoped roles, consumption thresholds, ownership transfer | `home.created`, `home.member-joined`, `home.member-left`, `home.member-removed`, `home.thresholds-updated`, `home.ownership-transferred`, `home.deleted` (RabbitMQ) | Microservice, in production |
| **Alerts** | `en-monitor-alert-service` (:8088) | Threshold, unusual-consumption, limit, connectivity and linked-device alerts | `alert.raised` (RabbitMQ) | Microservice, in production |
| **Recommendations** | `en-monitor-recommendation-service` (:8089) | Consumption-based suggestions (five rules, hourly) | `recommendation.created` (RabbitMQ) | Microservice, in production |
| **Notification** | `en-monitor-notification-service` (:8087) | E-mail, Web Push (VAPID), mobile push (Expo), in-app inbox and administrator notices; delivery log | — (it only consumes) | Microservice, in production |

All client requests enter through `en-monitor-api-gateway` (:8080). Details:
`05-architecture/overview.md`.

## Technology stack

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Measuring hardware | ESP32 + PZEM-004T V3 + HLK-PM03, firmware 1.3.8 | The PZEM measures real electrical magnitudes rather than estimating them; the ESP32 provides Wi-Fi and Bluetooth (used to link a module from the web or the app) and enough capacity to report measurements autonomously; the HLK-PM03 allows the device to be powered from the same mains circuit it monitors |
| Device communication | MQTT over Wi-Fi, via Eclipse Mosquitto with `mosquitto-go-auth` (TLS on 8883) | Designed for constrained devices and unreliable networks; the publish/subscribe model decouples devices from the services. The broker runs in the `energy-monitor-infra` Docker stack and authenticates every client against the device service (`ADR-007`) |
| Web frontend | React 19 + Vite 8 | The component-based model fits a dashboard with charts and reusable views; the ecosystem also provides mature support for internationalization and data visualization |
| Android mobile application | Expo 57 / React Native (Android only) | Shares language, component model, and mental model with the web client, so the team maintains one set of skills instead of two; produces a native Android application. iOS is not targeted and is not planned |
| API gateway | Spring Cloud Gateway Server Web MVC (Spring Boot 4.0.8, Spring Cloud 2025.1.3) | Same Java/Spring stack as the services (`ADR-004`) |
| Backend services | Java 21 + Spring Boot 4.1.1 (seven services) | Spring Security and the OAuth2 resource server cover the identity requirements; the ecosystem supports hexagonal architecture. Spring Modulith was used while the backend was a monolith (retired 28-Oct-2026) |
| Service-to-service events | RabbitMQ 4.1, transactional outbox per service | A change and its message commit together; at-least-once delivery with dead-letter queues (`ADR-008`) |
| Database | MySQL 8.4, one shared schema; Liquibase per service | The relational model fits the domain; one shared instance keeps operational complexity low, with each service owning its tables (`ADR-003`, `ADR-009`) |
| Infrastructure | Docker, Docker Compose (local, development and production modes); Nginx + Certbot on an Ubuntu VPS | Reproducible environments for local development and production (`05-architecture/deployment.md`) |
| CI/CD | GitHub Actions, one workflow per repository | Tests on every pull request and push to `dev`/`main`; approved deployment from `main` |

## Current status

- **Phase:** In production at `https://energy-monitor.tech` since 24-Oct-2026.
- **Current version:** Not versioned with release tags; production runs the `main` branch of each
  repository (`09-microservices/service-catalog.md` lists the commit of each one).
- **Implemented bounded contexts:** all seven.
- **Architecture:** API gateway + seven microservices (Security, Devices, Monitoring and
  Measurements, Home Management, Notification, Alerts, Recommendations). The
  `energy-monitor-backend` modular monolith was retired on 28-Oct-2026.
- **Known limitations:** the thresholds of the unusual-consumption alert are constants in the code
  (DT-27); the news shown in the web are generated from the git history and their titles are in
  English (DT-26); the mobile app does not have the administration screens of the web.
- **Project delivery:** 28-Oct-2026.

## Project contacts

| Role | Name | Contact |
|------|------|---------|
| Developer | Juan Esteban Cortés Parra | juanestebanp171@gmail.com |
| Developer | Miguel Ángel García Artunduaga | garmiguelangel89@gmail.com |

**Training program:** Software Analysis and Development (ADSO) — SENA, group (ficha) 3145556.

## Correlations

- System scope → `01-context/scope.md`
- Domain glossary → `01-context/glossary.md`
- Architecture decisions → `05-architecture/decisions/records/`
- Module contracts → `07-api/contracts/`