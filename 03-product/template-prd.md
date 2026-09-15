# PRD — Monitor de Consumo Eléctrico

| Field                 | Value                                                               |
| --------------------- | ------------------------------------------------------------------- |
| **Author(s)**         | Miguel Ángel García Artunduaga, Juan Esteban Cortés                 |
| **Status**            | Draft                                                               |
| **Instructor**        | Motta Vargas José de Jesús                                          |
| **Program**           | SENA — Programación de Software, group 3145556                      |
| **Related documents** | `problem-framing.md`, `discovery-brief.md`, `vision.md`, `SRS.docx` |

---

## 1. Summary

Monitor de Consumo Eléctrico is a web and mobile platform that lets a responsible user register
a project or home, link electricity-measuring devices to it, and let anyone with access see
consumption in near real time. It raises alerts when consumption exceeds a configured
threshold and generates personalized recommendations to reduce it — replacing the monthly
electricity bill as the household's only source of consumption information.

---

## 2. Problem statement

**Who has the problem?** People who consume electricity at home and want to understand,
control, and optimize that consumption (see `problem-framing.md`).
**What is the problem?** They lack clear, timely, detailed information about how, when, and
where their electricity consumption happens, and today rely almost entirely on the monthly
bill — an after-the-fact, low-resolution signal.
**Evidence** Discovery survey (n=31): 80.6% consider monitoring very important, yet 54.8% do not
monitor consumption in any active way today. 83.9% want real-time visualization and 83.9% want
consumption alerts — the two most requested features by a wide margin (`discovery-brief.md`).
This survey informed prioritization; the requirements below come from the SRS, not from the
survey itself.

---

## 3. Goals and non-goals

### Goals

- Let anyone with access to a project/home see its electricity consumption broken down by
  time period (RF4.2).
- Notify users automatically when consumption exceeds a configured threshold (RF4.3).
- Generate recommendations grounded in the project's own measured consumption behavior (RF4.4).
- Support both web and mobile access with responsive, accessible interfaces (RNF4).

### Non-goals

- Estimating electricity cost in local currency — not part of any RF in the SRS; the system
  reports measured consumption, not a billed amount.
- Anything requiring a specific hardware brand, communication protocol, or message broker not
  named in the SRS — the SRS specifies HTTP and a 60-second reporting interval (ERF5.1) and
  nothing else about the device's implementation.
- A dedicated iOS/Android/desktop-specific feature set beyond what RF1–RF5 already describe as
  "web and mobile" — the SRS does not distinguish platform-specific scope beyond that.

---

## 4. Target users / personas

| Persona                                | Description                                                                                                                            | Primary need                                                                                                      |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Usuario responsable (Responsible user) | Registers a project or home, links devices and regular users to it, holds the platform's advanced permission for that project (ERF2.4) | Full visibility and administrative control over a project/home's devices, users, and configuration                |
| Usuario regular (Regular user)         | A person linked to a project/home they did not register                                                                                | Visibility into that project's consumption, alerts, and recommendations, with permissions scoped by role (ERF2.2) |

---

## 5. User stories covered

> Full acceptance criteria live in `04-requirements/user-stories.md`. All 10 HUs are listed —
> there is no larger backlog behind this PRD.

| HU   | Title                                                         | Priority    |
| ---- | ------------------------------------------------------------- | ----------- |
| HU1  | Registration and Account Management (RF1)                     | High        |
| HU2  | Project and Home Management (RF2)                             | High        |
| HU3  | System Configuration and Personalization (RF3)                | Medium-High |
| HU4  | Electricity Consumption Query and Analysis (RF4, partial)     | Very High   |
| HU5  | Alert Management and Automatic Recommendations (RF4, partial) | High        |
| HU6  | Measuring Device Management (RF5)                             | Very High   |
| HU7  | System Performance (RNF1)                                     | High        |
| HU8  | System Scalability (RNF2)                                     | Medium      |
| HU9  | Information Security (RNF3)                                   | High        |
| HU10 | Usability and Accessibility (RNF4)                            | Medium      |

---

## 6. Functional requirements

> Full ERF-level detail in `04-requirements/functional.md`. Summarized by RF here.

| ID        | Requirement                                                                                                                                                                                                                                           | Priority    |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| RF1       | The system must support user registration, authentication (including social login), profile editing, password recovery, and account deletion                                                                                                          | High        |
| RF2       | The system must let a responsible user register a project or home, link regular users and devices to it, and let any user with access mark it as a favorite                                                                                           | High        |
| RF3       | The system must let a user configure consumption thresholds, alert channel, color palette, and language, with system defaults as fallback                                                                                                             | Medium-High |
| RF4.1–4.2 | The system must automatically record consumption from connected devices and let users view it filtered by date range, compared against historical averages, and as trend charts                                                                       | Very High   |
| RF4.3–4.4 | The system must automatically generate alerts when thresholds are exceeded and recommendations based on detected consumption behavior                                                                                                                 | High        |
| RF5       | The measuring device must transmit voltage, current, active power, and accumulated energy every 60 seconds over HTTP, authenticate with a unique identifier and access key, report its connection status, and reconnect automatically after a failure | Very High   |

---

## 7. Non-functional requirements

> Full detail in `04-requirements/non-functional.md`, including the SRS's abnormal-situation
> handling for each.

| Category           | Requirement                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| Performance (RNF1) | Response time under 2 seconds for data/chart loading; stable throughput under multiple concurrent users                |
| Scalability (RNF2) | Horizontal scaling to handle growth in users, connected devices, and data volume without affecting the user experience |
| Security (RNF3)    | Secure authentication, encryption in transit and at rest, compliance with regulations such as GDPR or equivalent       |
| Usability (RNF4)   | Intuitive, responsive interface, WCAG accessibility compliance, contextual help and clear error messages               |

RNF5 (Availability), RNF6 (Maintainability), and RNF7 (Portability) exist in the SRS only as a
one-line overview, with no detailed metric and no HU yet — see the open item in
`04-requirements/non-functional.md`.

---

## 8. Out of scope

| Item                                          | Reason                                                                                |
| --------------------------------------------- | ------------------------------------------------------------------------------------- |
| Cost estimation in local currency             | Not part of any RF in the SRS                                                         |
| Any specific hardware brand or message broker | Not specified anywhere in the SRS; only HTTP and a 60s interval are (ERF5.1)          |
| RNF5–RNF7 as buildable, tested requirements   | The SRS documents them only as a one-line overview, without the detail RNF1–RNF4 have |

---

## 9. Success metrics

| Metric                                                   | Baseline                  | Target                                                                                          | Measured by                 |
| -------------------------------------------------------- | ------------------------- | ----------------------------------------------------------------------------------------------- | --------------------------- |
| Data/graph load time                                     | Not measured (pre-launch) | < 2 seconds                                                                                     | RNF1 verification           |
| Users who report monitoring consumption in an active way | 45.2% (survey baseline)   | Increase post-launch, re-measured with a follow-up survey                                       | Follow-up survey            |
| Alert accuracy                                           | Not measured              | Alerts raised only when the configured threshold is genuinely exceeded, resolved when it is not | RNF1/RF4.3 verification     |
| Accessibility compliance                                 | Not measured              | WCAG-compliant across web and mobile                                                            | RNF4 / accessibility review |

---

## 10. Timeline / milestones

> The SRS does not specify a delivery date, project start date, or sprint length anywhere in its
> scope, constraints, or planning sections. `03-product/roadmap.md` sequences the 10 HUs by
> dependency instead of by calendar date — fill in real dates there once the team and instructor
> agree on them, rather than here.

| Milestone                                                       | Depends on |
| --------------------------------------------------------------- | ---------- |
| Phase 1 — Foundation (HU1, HU2) complete                        | None       |
| Phase 2 — Configuration & Device Onboarding (HU3, HU6) complete | Phase 1    |
| Phase 3 — Consumption Visibility (HU4) complete                 | Phase 2    |
| Phase 4 — Alerts & Recommendations (HU5) complete               | Phase 3    |

---

## 11. Risks and open questions

| Risk / Question                                                                     | Impact if unresolved                                                 | Owner             | Status |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------- | ----------------- | ------ |
| No delivery date or sprint length defined                                           | The roadmap can sequence work but cannot commit to when it ships     | Team + instructor | Open   |
| "Entrevista a profesional" and "Entrevista a aprendiz" have no logged responses yet | Discovery brief's qualitative findings are incomplete                | Team              | Open   |
| RNF5–RNF7 have no detailed metric or HU                                             | Availability, maintainability, and portability targets are undefined | Team              | Open   |

---

## 12. Approval

| Role              | Name                                                | Decision | Date |
| ----------------- | --------------------------------------------------- | -------- | ---- |
| Responsible users | Miguel Ángel García Artunduaga, Juan Esteban Cortés | Pending  |      |
| Instructor        | Motta Vargas José de Jesús                          | Pending  |      |

---

## Correlations

- Problem this PRD addresses → `problem-framing.md`
- Research backing this PRD → `discovery-brief.md`
- Vision this PRD implements → `vision.md`
- Backlog derived from this PRD → `product-backlog.md`
- Detailed stories and requirement codes → `04-requirements/user-stories.md`, `04-requirements/functional.md`, `04-requirements/non-functional.md`
- Source documents → `SRS.docx`, `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`
