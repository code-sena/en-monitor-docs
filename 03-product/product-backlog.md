# Product Backlog — Monitor de Consumo Eléctrico

Functional HUs are ordered by dependency, exactly as stated in each HU's own "Dependencias"
field. Non-functional HUs are cross-cutting quality requirements — they don't sit at one point
in the sequence; they apply across the functional HUs listed next to them and should be
validated continuously, not just at the end.

---

## Functional backlog (HU1–HU6)

| Rank | HU  | Covers                                         | Title                                          | Role                                      | Priority    | Dependencies                              |
| ---- | --- | ---------------------------------------------- | ---------------------------------------------- | ----------------------------------------- | ----------- | ----------------------------------------- |
| 1    | HU1 | RF1 — User Management                          | Registration and Account Management            | User (web or mobile)                      | High        | None — base for RF2, RF3, RF4, RF5        |
| 2    | HU2 | RF2 — Project Management                       | Project and Home Management                    | Responsible user                          | High        | HU1. Uses RF3 and RF4                     |
| 3    | HU3 | RF3 — System Configuration                     | System Configuration and Personalization       | User                                      | Medium-High | HU1. Uses RF4                             |
| 4    | HU6 | RF5 — Device Management                        | Measuring Device Management                    | System / Responsible user                 | Very High   | RF1, RF2, RF3, RF5                        |
| 5    | HU4 | RF4 (partial) — Consumption Query and Analysis | Electricity Consumption Query and Analysis     | Registered user associated with a project | Very High   | RF1, RF2, RF3, RF5                        |
| 6    | HU5 | RF4 (partial) — Alerts and Recommendations     | Alert Management and Automatic Recommendations | User                                      | High        | ERF4.3 and ERF4.4 (both part of RF4), RF3 |

---

## Non-functional backlog (HU7–HU10) — cross-cutting

| HU   | Covers             | Title                       | Role                              | Priority | Applies to         |
| ---- | ------------------ | --------------------------- | --------------------------------- | -------- | ------------------ |
| HU7  | RNF1 — Performance | System Performance          | Frequent user                     | High     | RF2, RF5           |
| HU9  | RNF3 — Security    | Information Security        | Registered user                   | High     | RF1, RF3           |
| HU8  | RNF2 — Scalability | System Scalability          | Administrator / technical team    | Medium   | Whole system       |
| HU10 | RNF4 — Usability   | Usability and Accessibility | User without technical experience | Medium   | RF1, RF2, RF4, RF5 |

> RNF5 (Availability), RNF6 (Maintainability), and RNF7 (Portability) exist in the SRS as a
> one-line overview but have no dedicated HU yet — see the open item already logged in
> `04-requirements/non-functional.md`. Not repeated as a backlog row here since there is no
> story to point to.

---

## Rules for keeping this list healthy

1. Order functional HUs by dependency — a story cannot rank above one it depends on.
2. Non-functional HUs are not "done once" — re-validate them as each functional HU that touches
   their scope is completed (e.g. re-check HU7's response-time target after HU4 ships, since HU4
   is the heaviest read path).
3. If HU4 or HU5 grow additional acceptance criteria during grooming, check whether they should
   split into two HUs before entering a sprint — see `04-requirements/user-stories.md` for their
   current, already-detailed acceptance criteria.
4. Do not add new backlog items sourced from the discovery survey alone. The survey
   (`03-product/discovery-brief.md`) informs which of these already-scoped HUs to prioritize
   first; it is not itself a source of new requirements.

---

## Correlations

- Backlog item format → [`template-backlog.md`](./template-backlog.md)
- Full acceptance criteria for every HU → `04-requirements/user-stories.md`
- Functional/non-functional requirement detail (ERF/RNF codes) → `04-requirements/functional.md`, `04-requirements/non-functional.md`
- Delivery sequencing → `03-product/roadmap.md`
- Source documents → `SRS.docx`, `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`
