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

The system exposes two clients over the same backend API. They are not equivalent: each one serves a different moment of use.

| Client | Platform | Purpose |
|--------|----------|---------|
| Web application | Browser | Full experience: home and device management, detailed history, statistics, and administration |
| Mobile application | Android | Day-to-day consultation and alerts, on the device the user already carries |

## Bounded contexts (modules)

The backend is a **modular monolith** built with Spring Modulith. Each module is a bounded context with its own domain model, application layer, and hexagonal architecture (ports & adapters). Modules communicate through well-defined APIs and domain events. Each module is designed to become an independently deployable microservice in the future.

| Context | Module name | Responsibility | Key events published | Depends on |
|---------|-------------|----------------|---------------------|------------|
| **Security** | `security` | Identity, authentication, sessions, RBAC, audit trail | — | (none) |
| **Home Management** | `home` | Homes, membership, home-scoped roles, consumption thresholds | — | `security::api` |
| **Devices** | `device` | Device registry, device authentication (API key), connectivity tracking | `DeviceConnectivityLost` | `home::api` |
| **Monitoring and Measurements** | `measurement` | Measurement ingestion, consumption history, statistics, consumption levels | `MeasurementRecorded` | `device::api`, `home::api` |
| **Alerts** | `alert` | Threshold and connectivity alerts | `AlertRaised` | `home::api`, `device::api`, `measurement::api` |
| **Recommendations** | `recommendation` | Consumption-based suggestions | `RecommendationCreated` | `device::api`, `measurement::api`, `home::api` |
| **Notification** | `notification` | Delivery log (email, push) for alerts, recommendations, password reset | — | `security::api`, `home::api`, `device::api`, `alert::api`, `recommendation::api` |

## Technology stack

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Measuring hardware | ESP32 + PZEM-004T V3 + HLK-PM03 | The PZEM measures real electrical magnitudes rather than estimating them; the ESP32 provides Wi-Fi connectivity and enough capacity to report measurements autonomously; the HLK-PM03 allows the device to be powered from the same mains circuit it monitors |
| Device communication | MQTT over Wi-Fi, via Eclipse Mosquitto (TLS) | Designed for constrained devices and unreliable networks; the publish/subscribe model decouples devices from the backend. The broker runs in the `devices` repository, not in the backend. |
| Web frontend | React + Vite | The component-based model fits a dashboard with charts and reusable views; the ecosystem also provides mature support for internationalization and data visualization |
| Android mobile application | React Native (Android only) | Shares language, component model, and mental model with the web client, so the team maintains one set of skills instead of two; produces a native Android application. iOS is not targeted and is not planned |
| Backend | Java 21 + Spring Boot 3.x + Spring Modulith | Spring Security and OAuth2 cover the identity requirements; Spring Modulith enforces module boundaries; the ecosystem supports hexagonal architecture |
| Database | MySQL | The relational model fits the domain, which contains many relationships; a single shared instance keeps operational complexity low at this stage |
| Infrastructure | Docker, Docker Compose | Provides reproducible environments for local development and production |

> **Note:** The backend is a **modular monolith** (single deployable artifact). Modules are separated by Spring Modulith boundaries, not by deployment. Migration to independently deployed microservices is planned but not yet executed.

## Current status

- **Phase:** In development
- **Current version:** Not yet versioned. No release tags exist.
- **Last release:** None
- **Implemented bounded contexts:** All 7 contexts are implemented (Security, Home Management, Devices, Monitoring and Measurements, Alerts, Recommendations, Notification)
- **Next milestone:** Harden cross-module integration tests, document module contracts (OpenAPI), prepare for microservice extraction

## Project contacts

| Role | Name | Contact |
|------|------|---------|
| Developer | Juan Esteban Cortés Parra | juanestebanp171@gmail.com |
| Developer | Miguel Ángel García Artunduaga | garmiguelangel89@gmail.com |

**Training program:** Software Analysis and Development — SENA, group 314556.

## Correlations

- System scope → `01-context/scope.md`
- Domain glossary → `01-context/glossary.md`
- Architecture decisions → `05-architecture/decisions/records/`
- Module contracts → `07-api/contracts/`