# Non-Functional Requirements (NFR)

> Transcribed directly from the SRS (`RNF`/`ERNF` numbering preserved exactly as defined there).
> RNF1–RNF4 have full detail tables in the SRS (including abnormal-situation handling); RNF5–RNF7
> only appear as a one-line overview in the SRS, with no detailed table — that gap is called out
> explicitly below rather than invented. Source of truth: `SRS.docx`, section "Requerimientos No
> Funcionales." Source of the HU column: `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`.

---

## RNF1: Performance

| ID      | Requirement                              | Metric                                                                                                               | How to verify                                                                              |
| ------- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| ERNF1.1 | Response time, load capacity, throughput | Data/graph loading response time < 2 seconds; supports multiple concurrent users while maintaining stable throughput | Load tests, concurrent-user simulations (per SRS, "Documentos de visualización asociados") |

**Applies to:** RF2 (Project Management), RF5 (Device Management), per the SRS's own
"Especialización" field for this requirement.

**Abnormal-situation handling (from the SRS):**

- Performance degradation → activate load balancing or scale services
- Congestion → notify the user and retry automatically

**Source HU:** HU7 (System Performance) — **Priority:** High

---

## RNF2: Scalability

| ID      | Requirement                                | Metric                                                                                                                                           | How to verify                                              |
| ------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| ERNF2.1 | Capacity to handle growth in users or data | The system must scale horizontally to handle constant growth in users, connected devices, and data volume, without affecting the user experience | Autoscaling / task redistribution tests under growing load |

**Abnormal-situation handling (from the SRS):**

- Service saturation → autoscaling or task redistribution

**Source HU:** HU8 (System Scalability) — **Priority:** Medium

---

## RNF3: Security

| ID      | Requirement                                                               | Description                                                                                                                                                                           |
| ------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ERNF3.1 | Protection against unauthorized access, encryption, regulatory compliance | The system must protect user information through secure authentication mechanisms, data encryption in transit and at rest, and compliance with regulations such as GDPR or equivalent |

**Abnormal-situation handling (from the SRS):**

- Unauthorized access attempts → temporary lockout and event logging
- Detected vulnerabilities → apply a patch immediately (per SRS)

**Source HU:** HU9 (Information Security) — **Priority:** High

---

## RNF4: Usability

| ID     | Requirement                                 | Description                                                                                                                                         |
| ------ | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| RNF4.1 | Ease of use, accessibility, user experience | The application must be easy to use, accessible to users without technical experience, and comply with usability and accessibility standards (WCAG) |

**Abnormal-situation handling (from the SRS):**

- Navigation confusion → contextual assistance or tutorial
- User errors → clear messages and correction options

**Source HU:** HU10 (Usability and Accessibility) — **Priority:** Medium

---

## RNF5–RNF7: Defined in the SRS overview, not yet detailed

The SRS lists these three additional non-functional requirements only as a one-line overview,
with no detailed table (unlike RNF1–RNF4 above) and **no corresponding HU** in the current
10-story document:

| ID             | Name            | SRS one-line description                                       |
| -------------- | --------------- | -------------------------------------------------------------- |
| RNF5 / RNF5.1  | Availability    | Percentage of time the system must be operational (e.g. 99.9%) |
| RNF6 / ERNF6.1 | Maintainability | Ease of updating or fixing the system                          |
| RNF7 / ERNF7.1 | Portability     | Ability to operate across different platforms or environments  |

**Portability, as further specified in the SRS's interface requirements:** Windows 7 or later
for desktop, the latest available version of Android and iOS for mobile, and compatibility with
Mozilla, Chrome, Edge, Brave, Vivaldi, and similar browsers.

> **Open item:** RNF5–RNF7 are part of the SRS but currently have no dedicated Historia de
> Usuario. If the team wants explicit backlog coverage for them (rather than treating them as
> implicit quality bars), the natural next step is to add HU11 (Availability), HU12
> (Maintainability), and HU13 (Portability) to `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`,
> mirroring the format already used for HU7–HU10. This is flagged here, not done unilaterally,
> since it changes the HU count the team is tracking.

---

## Correlations

- Full SRS source → `SRS.docx`, section "Requerimientos No Funcionales"
- Formalized HUs these NFRs are grouped under → `04-requirements/user-stories.md`
- Functional counterparts → `04-requirements/functional.md`
- Coverage tracking → `04-requirements/traceability-matrix.md`
