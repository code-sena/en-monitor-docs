# User Stories — Backlog

> Formalized version of the project's actual Historia de Usuario document
> (`Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`). **There are 10 HUs — 6
> functional, 4 non-functional — not 31.** The 31 responses referenced elsewhere in this repo
> (`03-product/discovery-brief.md`) come from the discovery survey, a data-collection instrument
> that informed which of these requirements to prioritize; it is not a count of requirements or
> stories. Each HU below keeps the exact título/como/quiero/para/criterios/prioridad/dependencias
> content from the source document, reformatted with Given/When/Then acceptance criteria and
> linked to its RF/RNF in the SRS.

---

## Backlog status

| Group          | HUs      | Total | Source           |
| -------------- | -------- | ----- | ---------------- |
| Functional     | HU1–HU6  | 6     | Covers RF1–RF5   |
| Non-functional | HU7–HU10 | 4     | Covers RNF1–RNF4 |

RNF5–RNF7 (Availability, Maintainability, Portability) exist in the SRS but have no HU yet —
see the open item in `04-requirements/non-functional.md`.

---

## Functional User Stories

### HU1 — Registration and Account Management {#HU1}

**Covers:** RF1 (User Management) — ERF1.1, ERF1.1.1, ERF1.2, ERF1.3.1, ERF1.3.2, ERF1.3.3

> **As** a user of the system (web or mobile)
> **I want** to register, authenticate (directly or via social login), edit my profile, recover
> my password, and accept the terms and license
> **so that** I can securely access the platform and manage my projects and energy consumption

**Acceptance Criteria:**

```gherkin
Scenario 1: Manual registration
  Given a person who is not yet registered, with a name, email, and password
  When they submit the registration form and accept the terms, conditions, and license permissions
  Then a new account is created

Scenario 2: Email already registered
  Given an email that already belongs to an existing account
  When someone tries to register with that same email
  Then registration is rejected

Scenario 3: Account self-service
  Given a logged-in user
  When they update their profile data, request a password reset via secure email, or request account deletion
  Then the requested change is applied
```

| Field        | Value                           |
| ------------ | ------------------------------- |
| Priority     | High                            |
| Dependencies | Base for RF2, RF3, RF4, and RF5 |
| Status       | Backlog                         |

---

### HU2 — Project and Home Management {#HU2}

**Covers:** RF2 (Project Management) — ERF2.1, ERF2.2, ERF2.3, ERF2.4, ERF2.5

> **As** a responsible user
> **I want** to register projects or homes, link regular users, and assign measuring devices
> **so that** I can structure the administration of a specific environment's energy consumption
> and grant the corresponding permissions

**Acceptance Criteria:**

```gherkin
Scenario 1: Register a project or home
  Given a responsible user
  When they register a project or home with a name, location, and description
  Then the project or home is created and that user holds the advanced platform permission for it

Scenario 2: Link secondary users and devices
  Given an existing project or home and its responsible user
  When the responsible user links a regular user or a measuring device to it
  Then the linked user or device gains the access/permissions corresponding to their role

Scenario 3: Mark as favorite
  Given a user with access to a project or home
  When they mark it as a favorite
  Then it appears as a favorite for that user, regardless of their role
```

| Field        | Value                                   |
| ------------ | --------------------------------------- |
| Priority     | High                                    |
| Dependencies | HU1 (User Management). Uses RF3 and RF4 |
| Status       | Backlog                                 |

---

### HU3 — System Configuration and Personalization {#HU3}

**Covers:** RF3 (System Configuration) — ERF3.1, ERF3.2, ERF3.3, ERF3.4, ERF3.5

> **As** a user of the system
> **I want** to adjust consumption thresholds, notification channels, color palette, and
> language
> **so that** I can adapt the application to my visual, language, and energy-control needs

**Acceptance Criteria:**

```gherkin
Scenario 1: Configure thresholds
  Given a logged-in user
  When they define custom consumption thresholds (hourly, daily, or monthly) or leave them unset
  Then the system either applies the custom values or falls back to the preset system defaults

Scenario 2: Configure alerts, colors, and language
  Given a logged-in user
  When they choose how to receive alerts (push notification, email, or both), select a color palette, or change the language (up to 4 available)
  Then the selected preferences are applied

Scenario 3: Invalid threshold value
  Given a threshold value entered by the user
  When it falls outside a valid range
  Then the system rejects it and does not save the invalid value
```

| Field        | Value                           |
| ------------ | ------------------------------- |
| Priority     | Medium-High                     |
| Dependencies | HU1 (User Management). Uses RF4 |
| Status       | Backlog                         |

---

### HU4 — Electricity Consumption Query and Analysis {#HU4}

**Covers:** RF4 (partial: ERF4.1, ERF4.2.1, ERF4.2.2, ERF4.2.3)

> **As** a registered user associated with a project
> **I want** to query my historical measurements, filter by date range, and see trend charts
> **so that** I can identify consumption patterns, peaks or critical hours, and evaluate energy
> saving opportunities

**Acceptance Criteria:**

```gherkin
Scenario 1: View consumption charts and tables
  Given a project with recorded consumption
  When the user opens the consumption view
  Then recorded consumption is shown in dynamic charts (trends, peaks, critical hours) and tables

Scenario 2: Filter by date range
  Given a project with historical measurements
  When the user selects a custom date range
  Then only consumption within that range is shown

Scenario 3: Compare and export
  Given consumption data for the current period
  When the user requests a comparison against historical averages, or requests a report
  Then the system shows the comparison, and/or generates the report as a PDF export
```

| Field        | Value              |
| ------------ | ------------------ |
| Priority     | Very High          |
| Dependencies | RF1, RF2, RF3, RF5 |
| Status       | Backlog            |

---

### HU5 — Alert Management and Automatic Recommendations {#HU5}

**Covers:** RF4 (partial: ERF4.3, ERF4.3.1, ERF4.4, ERF4.4.1)

> **As** a user of the system
> **I want** to receive automatic alerts when thresholds are exceeded, and personalized
> indications for efficient use
> **so that** I can react promptly to unusual consumption and optimize my usage habits

**Acceptance Criteria:**

```gherkin
Scenario 1: Automatic alert generation
  Given a project whose consumption exceeds its configured threshold
  When the system evaluates incoming measurements in real time
  Then an alert is generated and reported automatically

Scenario 2: Automatic recommendations
  Given a project's detected consumption behavior
  When the system analyzes that behavior, even without a critical situation present
  Then a personalized recommendation is generated automatically

Scenario 3: View and act on notifications
  Given a user with generated alerts and recommendations
  When they open the notification history
  Then they can view alerts and resolve them, and mark recommendations as read
```

| Field        | Value                                                     |
| ------------ | --------------------------------------------------------- |
| Priority     | High                                                      |
| Dependencies | ERF4.3 and ERF4.4 (both part of RF4), RF3 (Configuration) |
| Status       | Backlog                                                   |

---

### HU6 — Measuring Device Management {#HU6}

**Covers:** RF5 (Device Management) — ERF5.1, ERF5.2, ERF5.3, ERF5.4

> **As** the system / a responsible user
> **I want** the measuring device to send periodic readings, authenticate securely, and report
> its connection status
> **so that** the automated, uninterrupted recording of electrical variables is guaranteed

**Acceptance Criteria:**

```gherkin
Scenario 1: Periodic data transmission
  Given a linked measuring device
  When it operates normally
  Then it transmits voltage, current, active power, and accumulated energy every 60 seconds via MQTT to the broker.

Scenario 2: Secure linking
  Given a device with a unique identifier and access key
  When it attempts to link to a project
  Then the system authenticates it and links it only if the credentials are valid

Scenario 3: Connection status and recovery
  Given a linked device
  When its last received measurement is recent, or when a WiFi/communication failure occurs
  Then the system shows it as online/offline accordingly, and the device attempts automatic reconnection without user intervention
```

| Field        | Value              |
| ------------ | ------------------ |
| Priority     | Very High          |
| Dependencies | RF1, RF2, RF3, RF5 |
| Status       | Backlog            |

---

## Non-Functional User Stories

### HU7 — System Performance {#HU7}

**Covers:** RNF1 (Performance) — ERNF1.1

> **As** a frequent user of the application
> **I want** queries, charts, and alert generation to respond quickly
> **so that** I can interact with the platform fluidly and without delays

**Acceptance Criteria:**

```gherkin
Scenario 1: Response time
  Given a request for data or a chart
  When it is processed under normal load
  Then the response time is under 2 seconds

Scenario 2: Concurrent users
  Given multiple users accessing the system simultaneously
  When load increases
  Then the system maintains stable throughput without service degradation
```

| Field        | Value                         |
| ------------ | ----------------------------- |
| Priority     | High                          |
| Dependencies | RNF1 (applies to RF2 and RF5) |
| Status       | Backlog                       |

---

### HU8 — System Scalability {#HU8}

**Covers:** RNF2 (Scalability) — ERNF2.1

> **As** an administrator / technical team
> **I want** the system architecture to scale horizontally
> **so that** it can support the constant increase of users, connected devices, and data volume
> without degrading the service

**Acceptance Criteria:**

```gherkin
Scenario 1: Handle load peaks
  Given a sudden or sustained increase in load
  When the system detects it
  Then it responds via autoscaling or task redistribution

Scenario 2: Continuous operation under growth
  Given a constant increase in users, devices, or data volume over time
  When the system continues operating
  Then it does so without loss of performance
```

| Field        | Value   |
| ------------ | ------- |
| Priority     | Medium  |
| Dependencies | RNF2    |
| Status       | Backlog |

---

### HU9 — Information Security {#HU9}

**Covers:** RNF3 (Security) — ERNF3.1

> **As** a registered user
> **I want** my credentials and consumption data to be transmitted and stored with encryption
> mechanisms
> **so that** the privacy of my information is guaranteed and unauthorized access is prevented

**Acceptance Criteria:**

```gherkin
Scenario 1: Encryption in transit and at rest
  Given any credential or consumption data handled by the system
  When it is transmitted or stored
  Then it is encrypted, both in transit and at rest

Scenario 2: Unauthorized access attempts
  Given repeated failed authentication attempts against an account
  When the configured attempt threshold is reached
  Then the account is temporarily locked

Scenario 3: Regulatory compliance
  Given the system's handling of personal data
  When it processes that data
  Then it complies with applicable personal-data-protection policies and regulations
```

| Field        | Value                         |
| ------------ | ----------------------------- |
| Priority     | High                          |
| Dependencies | RNF3 (applies to RF1 and RF3) |
| Status       | Backlog                       |

---

### HU10 — Usability and Accessibility {#HU10}

**Covers:** RNF4 (Usability) — RNF4.1

> **As** a user without specialized technical experience
> **I want** an interface that is responsive, clear, and has contextual help
> **so that** I can navigate and interpret information without difficulty, on mobile or web

**Acceptance Criteria:**

```gherkin
Scenario 1: Intuitive, responsive interface
  Given the web or mobile interface
  When a user navigates it
  Then the design is intuitive, coherent, and responsive across both environments

Scenario 2: Accessibility compliance
  Given the interface's visual and interactive elements
  When evaluated against WCAG guidelines
  Then they comply with those accessibility standards

Scenario 3: Error handling and help
  Given a user error or point of confusion
  When it occurs
  Then the system shows a clear error message, or offers contextual assistance or a tutorial
```

| Field        | Value                                    |
| ------------ | ---------------------------------------- |
| Priority     | Medium                                   |
| Dependencies | RNF4 (applies to RF1, RF2, RF4, and RF5) |
| Status       | Backlog                                  |

---

## Correlations

- Functional requirements these HUs implement → `04-requirements/functional.md`
- Non-functional requirements these HUs implement → `04-requirements/non-functional.md`
- Traceability matrix → `04-requirements/traceability-matrix.md`
- Source document → `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`
- Full SRS → `SRS.docx`
