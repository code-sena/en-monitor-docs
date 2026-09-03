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

The system exposes two clients over the same backend API. They are not equivalent: each one
serves a different moment of use.

| Client | Platform | Purpose |
|--------|----------|---------|
| Web application | Browser | Full experience: home and device management, detailed history, statistics, and administration |
| Mobile application | Android | Day-to-day consultation and alerts, on the device the user already carries |

## Technology stack

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Measuring hardware | ESP32 + PZEM-004T V3 + HLK-PM03 | The PZEM measures real electrical magnitudes rather than estimating them; the ESP32 provides Wi-Fi connectivity and enough capacity to report measurements autonomously; the HLK-PM03 allows the device to be powered from the same mains circuit it monitors |
| Device communication | MQTT over Wi-Fi | Designed for constrained devices and unreliable networks; the publish/subscribe model decouples devices from the backend, so neither needs to know the other's internal implementation |
| Web frontend | React + Vite | The component-based model fits a dashboard with charts and reusable views; the ecosystem also provides mature support for internationalization and data visualization |
| Android mobile application | React Native | Provides a cross-platform development framework while the project explicitly targets Android for the mobile application; it also allows the team to use React-based development concepts |
| Backend | Java 21 + Spring Boot | Spring Security and OAuth2 cover the identity requirements without building them from scratch; the ecosystem supports the hexagonal architecture required by the project |
| Database | MySQL | The relational model fits the domain, which contains many relationships; a single shared instance keeps operational complexity low at this stage |
| Infrastructure | Docker, Jenkins | Provides reproducible environments and independent CI/CD pipelines for each service |

Services are split by business domain and deployed independently, each with its own repository.

They currently share one physical database organized logically by domain. This is a deliberate architectural decision, not a pending migration.

## Current status

- **Phase:** In development

- **Current version:** Not yet versioned. No release tags exist.

- **Last release:** None

- **Next milestone:** Build the `Security` context, which owns identity, authentication, sessions, authorization, and auditing. It is a prerequisite for every other context because all of them depend on knowing who the user is.

## Project contacts

| Role | Name | Contact |
|------|------|---------|
| Developer | Juan Esteban Cortés Parra | juanestebanp171@gmail.com |
| Developer | Miguel Ángel García Artunduaga | garmiguelangel89@gmail.com |

**Training program:** Software Analysis and Development — SENA, group 314556.