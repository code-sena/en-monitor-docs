# PRD — Energy Monitor

| Field                 | Value                                                     |
| --------------------- | --------------------------------------------------------- |
| **Author(s)**         | Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga |
| **Status**            | Draft                                                     |
| **Date**              | 2026-09-09                                                |
| **Reviewers**         | SENA ADSO instructor(s), group 314556                     |
| **Related documents** | `problem-framing.md`, `discovery-brief.md`, `vision.md`   |

---

## 1. Summary

Energy Monitor is a web and mobile platform that lets members of a Colombian household register
their home, link electricity-measuring devices to it, and see their consumption in near
real time. It raises alerts when consumption exceeds a household-defined threshold and
generates personalized recommendations to reduce it — replacing the monthly electricity bill as
the household's only source of consumption information.

---

## 2. Problem statement

**Who has the problem?** Members of Colombian households who consume electricity and want to
understand, control, and optimize that consumption (see `problem-framing.md`).
**What is the problem?** They lack clear, timely, detailed information about how, when, and
where their electricity consumption happens, and today rely almost entirely on the monthly
bill — an after-the-fact, low-resolution signal.
**Evidence** Discovery survey (n=31): 80.6% consider monitoring very important, yet 54.8% do not
monitor consumption in any active way today. 83.9% want real-time visualization and 83.9% want
consumption alerts — the two most requested features by a wide margin (`discovery-brief.md`).

---

## 3. Goals and non-goals

### Goals

- Let a household see its electricity consumption broken down by device and time period.
- Notify a household automatically when consumption exceeds a threshold it has defined.
- Generate recommendations grounded in the household's own measured consumption behavior.
- Deliver the experience mobile-first (87.1% of surveyed users would access it from a phone).

### Non-goals

- Estimating the electricity **cost** in local currency — no utility tariff data source exists.
- Manual consumption entry from a paper/PDF bill — superseded by automatic MQTT ingestion.
- Remote control of connected devices — the weakest-ranked feature in discovery (16.1%).
- An iOS application — mobile is React Native, Android-only, by explicit decision.

---

## 4. Target users / personas

| Persona     | Description                                                                             | Primary need                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Home Owner  | Registers the home, links devices, manages members and their home role, sets thresholds | Full visibility and control over their household's energy use and who can see/manage it                         |
| Home Member | A person linked to a home who did not create it                                         | Visibility into the household's consumption, alerts, and recommendations, without administrative responsibility |

---

## 5. User stories covered

> Full acceptance criteria live in `04-requirements/user-stories.md`. Highest-value items shown;
> full ranked list in `product-backlog.md`.

| HU ID                   | Title                                                              | Priority |
| ----------------------- | ------------------------------------------------------------------ | -------- |
| HU-SEC-001/002/003/004  | Registration, login, RBAC, password recovery                       | Must     |
| HU-HOME-001/002/003/004 | Create/join a home, manage members, configure thresholds           | Must     |
| HU-DEV-001/002          | Register and authenticate a measuring device                       | Must     |
| HU-MON-001/002/003      | MQTT ingestion contract, idempotent ingestion, consumption history | Must     |
| HU-ALERT-001/003        | Threshold-exceeded alerts, alert lifecycle                         | Must     |
| HU-WEB-001 / HU-MOB-001 | Web dashboard and Android app for consumption/alerts               | Must     |
| HU-REC-001              | Personalized recommendations from consumption behavior             | Should   |

---

## 6. Functional requirements

| ID    | Requirement                                                                                                | Priority |
| ----- | ---------------------------------------------------------------------------------------------------------- | -------- |
| FR-1  | The system must let a user register, log in, and recover a forgotten password                              | Must     |
| FR-2  | The system must enforce role-based access control (Home Owner vs. Home Member)                             | Must     |
| FR-3  | A Home Owner must be able to create a home and link members and devices to it                              | Must     |
| FR-4  | A device must authenticate with a unique API key before its data is accepted                               | Must     |
| FR-5  | The system must ingest device measurements over MQTT idempotently (no double-counted energy on redelivery) | Must     |
| FR-6  | The system must let a household view consumption by hour/day/week/month and compare it to previous periods | Must     |
| FR-7  | The system must let a Home Owner configure a consumption threshold and raise an alert when it is exceeded  | Must     |
| FR-8  | The system must raise an alert when a device stops reporting                                               | Should   |
| FR-9  | The system must generate recommendations based on a home's own measured consumption                        | Should   |
| FR-10 | The system must deliver alerts (and recommendations) as push notifications                                 | Should   |

---

## 7. Non-functional requirements

| Category                  | Requirement                                                                                                                   |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Performance               | Response time for data/graph loading under 2 seconds                                                                          |
| Security                  | Authentication via access/refresh tokens; data encrypted in transit and at rest; account lockout after repeated failed logins |
| Availability              | Target uptime of 99.9%                                                                                                        |
| Usability / Accessibility | WCAG-compliant UI, mobile-first design (87.1% of users access via phone)                                                      |
| Scalability               | Services independently deployable and horizontally scalable as users/devices grow                                             |

---

## 8. Out of scope

| Item                              | Reason                                                      | Reconsider later?              |
| --------------------------------- | ----------------------------------------------------------- | ------------------------------ |
| iOS application                   | Android-only by explicit project decision                   | Yes, if the team/scope grows   |
| Manual consumption entry          | Superseded by automatic MQTT ingestion                      | No                             |
| Admin UI for Security             | Operated via backend endpoints this cycle                   | Yes, post-MVP                  |
| Per-service physical database     | Shared MySQL instance is the current architectural decision | Yes, if scale requires it      |
| Cost estimation in local currency | No tariff data source available                             | Yes, if a data source is found |

---

## 9. Success metrics

| Metric                                | Baseline                  | Target                                                 | Measured by                |
| ------------------------------------- | ------------------------- | ------------------------------------------------------ | -------------------------- |
| Data/graph load time                  | Not measured (pre-launch) | < 2 seconds                                            | Performance testing (RNF1) |
| Users actively monitoring consumption | 45.2% (survey baseline)   | ≥ 70% of onboarded users check the app at least weekly | In-app usage analytics     |
| System availability                   | Not measured              | ≥ 99.9% uptime                                         | Uptime monitoring          |
| Accessibility compliance              | Not measured              | WCAG-compliant across web and mobile                   | Accessibility audit        |

---

## 10. Timeline / milestones

> Full detail in `roadmap.md`.

| Milestone                                                               | Target date                             | Depends on                            |
| ----------------------------------------------------------------------- | --------------------------------------- | ------------------------------------- |
| Security context + core loop (auth → home → device → first measurement) | June 30, 2026                           | PZEM hardware arrival (June 15, 2026) |
| Consumption visibility + alerts live                                    | August 31, 2026                         | Core loop complete                    |
| Recommendations + i18n complete                                         | September 30, 2026                      | Alerts live                           |
| Hardening and final demo                                                | October 31, 2026 (last week of October) | All prior phases                      |

---

## 11. Risks and open questions

| Risk / Question                                                                     | Impact if unresolved                                                   | Owner | Status |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | ----- | ------ |
| PZEM-004T V3 hardware arrives late or behaves unexpectedly (expected June 15, 2026) | Delays MQTT ingestion validation, the most exposed item in the roadmap | Team  | Open   |
| Sprint length not yet formally agreed (assumed 1 week)                              | Roadmap dates may need adjustment                                      | Team  | Open   |
| "Entrevista a profesional" and "Entrevista a aprendiz" have no logged responses yet | Discovery brief's qualitative findings are incomplete                  | Team  | Open   |

---

## 12. Approval

| Role            | Name      | Decision | Date |
| --------------- | --------- | -------- | ---- |
| Product Owner   | [Pending] | Pending  |      |
| Tech Lead       | [Pending] | Pending  |      |
| SENA Instructor | [Pending] | Pending  |      |

---

## Correlations

- Problem this PRD addresses → `problem-framing.md`
- Research backing this PRD → `discovery-brief.md`
- Vision this PRD implements → `vision.md`
- Backlog derived from this PRD → `product-backlog.md`
- Detailed stories → `04-requirements/user-stories.md`
