# Product Backlog — Energy Monitor

> Built from [`_template-backlog.md`](./_template-backlog.md). Prioritized list of everything
> that must be built for the MVP defined in `01-context/scope.md`, ordered by user value.

**Method used:** MoSCoW (Must / Should / Could / Won't have this cycle), per the template.
Within the same MoSCoW band, rows are ordered by dependency — an item cannot rank above
something it depends on, even if it looks equally valuable on its own.

`HU ID`s are left as targets (`HU-[SERVICE]-NNN`) — the ID is assigned for real once the item
is promoted into a full story in `04-requirements/user-stories.md` using
[`04-requirements/_template-hu.md`](../04-requirements/_template-hu.md).

---

## Must have — the MVP does not work without these

| Rank | HU ID        | Context                        | Title                                                                | User role                | One-line value                                                                         | Size (SP) | Status  | Phase   |
| ---- | ------------ | ------------------------------ | -------------------------------------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------- | --------- | ------- | ------- |
| 1    | HU-SEC-001   | Security                       | Register with email and password                                     | Prospective user         | Can create an account to start using the system                                        | 3         | Backlog | Phase 1 |
| 2    | HU-SEC-002   | Security                       | Log in and receive access/refresh tokens                             | Registered user          | Can authenticate and stay authenticated safely                                         | 5         | Backlog | Phase 1 |
| 3    | HU-SEC-003   | Security                       | Enforce system roles (RBAC) on protected endpoints                   | Registered user / Admin  | Only authorized users can perform sensitive actions                                    | 5         | Backlog | Phase 1 |
| 4    | HU-SEC-004   | Security                       | Recover a forgotten password                                         | Registered user          | Can regain access without creating a new account                                       | 2         | Backlog | Phase 1 |
| 5    | HU-HOME-001  | Home Management                | Create a home                                                        | Home Owner               | Has a place to group devices, members, and thresholds                                  | 2         | Backlog | Phase 1 |
| 6    | HU-HOME-002  | Home Management                | Join an existing home with an access code                            | Home Member              | Can access a household they didn't create                                              | 3         | Backlog | Phase 1 |
| 7    | HU-HOME-003  | Home Management                | Manage home members and their home role                              | Home Owner               | Controls who has access to the home and what they can do in it                         | 3         | Backlog | Phase 1 |
| 8    | HU-DEV-001   | Devices                        | Register a measuring device and associate it with a home             | Home Owner               | Consumption data can be attributed to their home                                       | 3         | Backlog | Phase 1 |
| 9    | HU-DEV-002   | Devices                        | Authenticate a device with its own API key                           | Device                   | Only authorized devices can publish measurements                                       | 3         | Backlog | Phase 1 |
| 10   | HU-MON-001   | Monitoring and Measurements    | Define the MQTT topic/payload contract between device and backend    | Device / Backend         | Devices and backend can exchange measurements reliably                                 | 3         | Backlog | Phase 1 |
| 11   | HU-MON-002   | Monitoring and Measurements    | Ingest voltage/current/power/energy readings over MQTT, idempotently | Home Owner / Home Member | Consumption data is captured automatically, without duplicates even if MQTT redelivers | 5         | Backlog | Phase 1 |
| 12   | HU-MON-003   | Monitoring and Measurements    | View consumption history and statistics per home and per device      | Home Owner / Home Member | Understands how much energy is being used, where, and when                             | 5         | Backlog | Phase 2 |
| 13   | HU-HOME-004  | Home Management                | Configure a consumption threshold for a home                         | Home Owner               | Can define what "too much" means for their own household                               | 3         | Backlog | Phase 2 |
| 14   | HU-ALERT-001 | Alerts                         | Raise an alert when consumption exceeds the configured threshold     | Home Owner / Home Member | Finds out about a problem while there is still time to act                             | 5         | Backlog | Phase 2 |
| 15   | HU-ALERT-003 | Alerts                         | Alert lifecycle: an alert can be raised and later resolved           | Home Owner / Home Member | Can tell current problems apart from ones already handled                              | 3         | Backlog | Phase 2 |
| 16   | HU-WEB-001   | Frontend (React)               | Web dashboard with consumption charts and history views              | Home Owner / Home Member | Can visually understand consumption without reading raw numbers                        | 5         | Backlog | Phase 2 |
| 17   | HU-MOB-001   | Mobile (React Native, Android) | Mobile screens for consumption and alerts                            | Home Owner / Home Member | Can check consumption and alerts from the device they carry daily                      | 5         | Backlog | Phase 2 |

---

## Should have — needed for a complete, credible product, not for the first working loop

| Rank | HU ID        | Context          | Title                                                                    | User role                | One-line value                                                                               | Size (SP) | Status  | Phase   |
| ---- | ------------ | ---------------- | ------------------------------------------------------------------------ | ------------------------ | -------------------------------------------------------------------------------------------- | --------- | ------- | ------- |
| 18   | HU-SEC-005   | Security         | Lock an account after repeated failed login attempts                     | Registered user          | Reduces the impact of a brute-force attempt on their account                                 | 2         | Backlog | Phase 2 |
| 19   | HU-DEV-003   | Devices          | Derive device connectivity status from reporting activity                | Home Owner / Home Member | Knows whether the data they're seeing is current or stale                                    | 2         | Backlog | Phase 2 |
| 20   | HU-ALERT-002 | Alerts           | Raise an alert when a device stops reporting                             | Home Owner               | Finds out about a broken/disconnected device instead of silently missing data                | 3         | Backlog | Phase 2 |
| 21   | HU-ALERT-004 | Alerts           | Deliver alerts as push notifications, in addition to in-app              | Home Owner / Home Member | Learns about a problem even without the app open, which is how most users say they'll engage | 5         | Backlog | Phase 2 |
| 22   | HU-SEC-006   | Security         | Record an audit log of security-relevant actions                         | Admin                    | Can trace who did what, when, and from where if something goes wrong                         | 3         | Backlog | Phase 2 |
| 23   | HU-REC-001   | Recommendations  | Generate recommendations from a home's own measured consumption behavior | Home Owner / Home Member | Gets concrete, personalized guidance instead of generic energy-saving tips                   | 5         | Backlog | Phase 3 |
| 24   | HU-WEB-002   | Frontend (React) | Fully internationalized interface (react-i18next)                        | Any user                 | Can use the product in their preferred language                                              | 3         | Backlog | Phase 3 |

---

## Could have — valuable, but the product still works without them this cycle

| Rank | HU ID      | Context                     | Title                                                                       | User role                | One-line value                                                   | Size (SP) | Status  | Phase   |
| ---- | ---------- | --------------------------- | --------------------------------------------------------------------------- | ------------------------ | ---------------------------------------------------------------- | --------- | ------- | ------- |
| 25   | HU-REC-002 | Recommendations             | Deliver recommendations through the same push channel as alerts             | Home Owner / Home Member | Sees recommendations without hunting for them inside the app     | 2         | Backlog | Phase 3 |
| 26   | HU-MON-004 | Monitoring and Measurements | Compare consumption across periods (week-over-week, month-over-month)       | Home Owner / Home Member | Can tell whether a change in habits actually reduced consumption | 3         | Backlog | Phase 3 |
| 27   | HU-SEC-007 | Security                    | Configure personal preferences: notification channel, color theme, language | Registered user          | Can tailor the app to their own preferences                      | 3         | Backlog | Phase 3 |
| 28   | HU-SEC-008 | Security                    | Security administration: password policy and lockout configuration          | Admin                    | Can tune security behavior without a code change                 | 3         | Backlog | Phase 4 |

---

## Won't have this cycle

> Kept here, not deleted, so nobody re-proposes them without checking why they were cut.
> Source of truth for these exclusions: `01-context/scope.md`.

| Item                                           | Why it's out                                                                                         |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| iOS application                                | Mobile is React Native, Android-only by explicit decision; no iOS build/signing/distribution planned |
| Manual consumption entry from the utility bill | Superseded by automatic MQTT ingestion once hardware was defined                                     |
| Dedicated administration UI for Security       | Administrative use cases are operated via backend endpoints directly this cycle                      |
| Per-service physical database                  | Current architecture uses one shared MySQL instance, organized logically by domain                   |
| Energy cost estimation in local currency       | No utility tariff data source exists; the system reports energy (kWh), not billed cost               |

---

## Correlations

- Backlog item format → [`_template-backlog.md`](./_template-backlog.md)
- Phases these items are scheduled into → `03-product/roadmap.md`
- Full requirement detail once an item is promoted → `04-requirements/user-stories.md`
- Source feature list and exclusions → `01-context/scope.md`
