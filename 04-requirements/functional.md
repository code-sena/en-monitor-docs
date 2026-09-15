# Functional Requirements (FR)

> Transcribed directly from the SRS (`RF`/`ERF` numbering preserved exactly as defined there —
> this document does not introduce new requirements, it only restates the SRS's own RF1–RF5 in
> the table format this section of the repo expects, and links each one to the Historia de
> Usuario (HU) that carries it. Source of truth: `SRS.docx`, section "Requerimientos
> Funcionales." Source of the HU column: `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`
> (10 HUs total — 6 functional, 4 non-functional).

---

## RF1 — User Management

| ID       | Module          | Description                                                                                                                                            | Source (HU) | Priority |
| -------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- | -------- |
| ERF1.1   | User Management | The system must allow user registration and authentication, on both the web and mobile versions                                                        | HU1         | High     |
| ERF1.1.1 | User Management | The user must be able to log in or register using external authentication providers (social login); this is stored as part of their user configuration | HU1         | High     |
| ERF1.2   | User Management | The system must allow the user to log in                                                                                                               | HU1         | High     |
| ERF1.3.1 | User Management | The system must allow users to edit and update their personal information                                                                              | HU1         | High     |
| ERF1.3.2 | User Management | The system must allow users to reset a forgotten password via email or another secure verification method                                              | HU1         | High     |
| ERF1.3.3 | User Management | Users must be able to delete their account if they no longer wish to use the service                                                                   | HU1         | High     |

## RF2 — Project Management

> **Terminology note:** the SRS calls this entity "Project," and this table preserves that
> wording exactly since it is a direct transcription (see the file-level note above). Every other
> section of this repository — `02-domain/domain-map.md`, `06-data/models.md`,
> `05-architecture/` — calls the same entity **"Home"** (`Home Management` bounded context,
> `home` table, `HomeOwner`/`HomeMember` roles). "Project" below and "Home" everywhere else refer
> to the same thing; the SRS predates the Project→Home rename made in the technical model, and
> that correction was never carried back into `SRS.docx` or the Historia de Usuario document.
> This is tracked as a gap in `04-requirements/traceability-matrix.md`.

| ID     | Module             | Description                                                                                                                                                               | Source (HU) | Priority |
| ------ | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | -------- | --- | --- | ---- |
| ERF2.1 | Project Management | The system must let the Home Owner register a project (e.g. a house, a building, a company)                                                                               | HU2         | High     |
| ERF2.2 | Project Management | The system must assign permissions based on the user's role (Home Member or Home Owner), determining what actions they can perform, such as managing devices and projects | HU2         | High     |
| ERF2.3 | Project Management | The Homer Owner registers and links devices to the project, and links Home Member to the project                                                                          | HU2         | High     |
| ERF2.4 | Project Management | Every project or home must have one Home Owner in charge of managing all its devices; this role holds the platform's advanced permission for that project                 | HU2         | High     |
| ERF2.5 | Project Management | The system must let any user with access mark a project or home as a favorite, to speed up access to it                                                                   | HU2         | High     |     | HU2 | High |

## RF3 — System Configuration

| ID     | Module               | Description                                                                                                                                                                                      | Source (HU) | Priority    |
| ------ | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- | ----------- |
| ERF3.1 | System Configuration | The user can configure consumption thresholds to receive alerts and personalized recommendations; the system also provides preset default thresholds based on the user's needs                   | HU3         | Medium-High |
| ERF3.2 | System Configuration | The user can personalize the system's colors, choosing from predefined palettes that automatically adapt to dark mode                                                                            | HU3         | Medium-High |
| ERF3.3 | System Configuration | The user can configure the system language, with up to 4 languages available                                                                                                                     | HU3         | Medium-High |
| ERF3.4 | System Configuration | The user can configure how they want to receive alerts — through notifications or another digital channel such as email                                                                          | HU3         | Medium-High |
| ERF3.5 | System Configuration | The system must have a predefined global configuration establishing default values (consumption limits, language, notifications) that serve as the base for each user's individual configuration | HU3         | Medium-High |

## RF4 — Electricity Consumption Recording and Processing

| ID       | Module                    | Description                                                                                                                                                               | Source (HU) | Priority  |
| -------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | --------- |
| ERF4.1   | Consumption Recording     | The system must automatically record electricity consumption coming from connected devices, in a way the user can parameterize                                            | HU4         | Very High |
| ERF4.2.1 | Consumption Visualization | The user must be able to select date ranges to analyze consumption patterns                                                                                               | HU4         | Very High |
| ERF4.2.2 | Consumption Visualization | The system must allow comparing current consumption against historical averages                                                                                           | HU4         | Very High |
| ERF4.2.3 | Consumption Visualization | The system must offer a panel with consumption trend charts (peaks, critical hours, and monthly comparisons)                                                              | HU4         | Very High |
| ERF4.3   | Alerts                    | The system must automatically generate alerts by analyzing incoming measurements, using internal mechanisms that evaluate consumption levels in real time                 | HU5         | High      |
| ERF4.3.1 | Alerts                    | The user must be able to view alerts generated by the system and resolve them according to what the notification indicates                                                | HU5         | High      |
| ERF4.4   | Recommendations           | The system must generate automatic recommendations based on the electricity consumption behavior detected in the measurements, even when no critical situation is present | HU5         | High      |
| ERF4.4.1 | Recommendations           | The user must be able to view the recommendations generated by the system and mark them as read                                                                           | HU5         | High      |

## RF5 — Measuring Device Management

| ID     | Module            | Description                                                                                                                                                                                                                                                                                 | Source (HU) | Priority  |
| ------ | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | --------- |
| ERF5.1 | Device Management | The measuring device must automatically send electricity consumption data (voltage, current, active power, and accumulated energy) to the broker at regular 60-second intervals using the MQTT protocol, so that the broker can transmit the consumption information to subscribed clients. | HU6         | Very High |
| ERF5.2 | Device Management | The system must recognize and authenticate devices using a unique identifier and an access key, enabling their secure linking to a project                                                                                                                                                  | HU6         | Very High |
| ERF5.3 | Device Management | The system must monitor the device's connection status (online / offline) based on the last measurement received                                                                                                                                                                            | HU6         | Very High |
| ERF5.4 | Device Management | The device must be able to automatically reconnect to the system after a WiFi connection loss or a temporary communication failure, with no user intervention                                                                                                                               | HU6         | Very High |

---

## Note on methodology

These functional requirements were **not** derived from the discovery survey. The survey
(31 valid responses, see `03-product/discovery-brief.md`) was a data-collection instrument used
during discovery to validate which features mattered to prospective users — it informed which
of these already-scoped requirements to prioritize, not the requirements themselves. The
requirements above come from the SRS, which was authored from the project's own analysis, not
generated from the survey sample size.

---

## Correlations

- Full SRS source → `SRS.docx`, section "Requerimientos Funcionales"
- Formalized HUs these FRs are grouped under → `04-requirements/user-stories.md`
- Non-functional counterparts → `04-requirements/non-functional.md`
- Coverage tracking → `04-requirements/traceability-matrix.md`
