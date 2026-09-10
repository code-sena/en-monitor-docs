# Product Backlog Template — Monitor de Consumo Eléctrico (initial pass)

> First, prioritized pass over what must be built. **10 HUs total — HU1–HU6 functional
> (RF1–RF5), HU7–HU10 non-functional (RNF1–RNF4)** — not a 28-item list, and not derived from
> the discovery survey. Transcribed from `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`.

## Backlog table (initial pass)

| Rank | HU   | Covers        | Title                                          | Role                              | Priority    | Dependencies                      |
| ---- | ---- | ------------- | ---------------------------------------------- | --------------------------------- | ----------- | --------------------------------- |
| 1    | HU1  | RF1           | Registration and Account Management            | User (web or mobile)              | High        | None — base for RF2–RF5           |
| 2    | HU2  | RF2           | Project and Home Management                    | Responsible user                  | High        | HU1. Uses RF3 and RF4             |
| 3    | HU3  | RF3           | System Configuration and Personalization       | User                              | Medium-High | HU1. Uses RF4                     |
| 4    | HU6  | RF5           | Measuring Device Management                    | System / Responsible user         | Very High   | RF1, RF2, RF3, RF5                |
| 5    | HU4  | RF4 (partial) | Electricity Consumption Query and Analysis     | Registered user in a project      | Very High   | RF1, RF2, RF3, RF5                |
| 6    | HU5  | RF4 (partial) | Alert Management and Automatic Recommendations | User                              | High        | ERF4.3, ERF4.4 (part of RF4), RF3 |
| 7    | HU7  | RNF1          | System Performance                             | Frequent user                     | High        | Applies to RF2, RF5               |
| 8    | HU9  | RNF3          | Information Security                           | Registered user                   | High        | Applies to RF1, RF3               |
| 9    | HU8  | RNF2          | System Scalability                             | Administrator / technical team    | Medium      | Whole system                      |
| 10   | HU10 | RNF4          | Usability and Accessibility                    | User without technical experience | Medium      | Applies to RF1, RF2, RF4, RF5     |

**Status legend:** Backlog (identified, not yet detailed) · Ready (meets Definition of Ready) ·
In Sprint (being built) · Done (meets Definition of Done). All 10 rows above are currently
Backlog — see `04-requirements/user-stories.md` for the fully detailed, Ready version of each.

## Rules for keeping this list healthy

1. Order functional HUs by dependency, exactly as their own "Dependencias" field states — not
   by convenience.
2. Non-functional HUs are cross-cutting; re-validate them as related functional HUs ship, don't
   treat them as "done once."
3. Re-rank at every backlog refinement, not just when new items are added.
4. Do not invent new backlog items from the discovery survey — it informs prioritization of
   these 10 HUs, it is not itself a requirements source.
5. Do not delete completed items — mark them Done and move them down.

---

## Correlations

- Full user story format → `04-requirements/_template-hu.md`
- Refined, ranked version of this list → `03-product/product-backlog.md`
- Prioritization feeds → `03-product/roadmap.md`
- Requirement detail (ERF/RNF codes) → `04-requirements/functional.md`, `04-requirements/non-functional.md`
