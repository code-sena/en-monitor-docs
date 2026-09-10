# Product Backlog Template — Energy Monitor (initial pass)

> First, prioritized pass over what must be built, before each item is developed into a full
> user story in `04-requirements/user-stories.md` using `04-requirements/_template-hu.md`.

**Method used:** MoSCoW (Must / Should / Could / Won't have this cycle).

## Backlog table (initial pass)

| Rank | HU ID        | Context / Epic              | Title                                                    | User role               | One-line value                                            | Priority | Size (SP) | Status  | Target phase |
| ---- | ------------ | --------------------------- | -------------------------------------------------------- | ----------------------- | --------------------------------------------------------- | -------- | --------- | ------- | ------------ |
| 1    | HU-SEC-001   | Security                    | Register with email and password                         | Prospective user        | Can create an account to start using the system           | Must     | 3         | Backlog | Phase 1      |
| 2    | HU-SEC-002   | Security                    | Log in and receive access/refresh tokens                 | Registered user         | Can authenticate and stay authenticated safely            | Must     | 5         | Backlog | Phase 1      |
| 3    | HU-SEC-003   | Security                    | Enforce system roles (RBAC) on protected endpoints       | Registered user / Admin | Only authorized users can perform sensitive actions       | Must     | 5         | Backlog | Phase 1      |
| 4    | HU-SEC-004   | Security                    | Recover a forgotten password                             | Registered user         | Can regain access without creating a new account          | Must     | 2         | Backlog | Phase 1      |
| 5    | HU-HOME-001  | Home Management             | Create a home                                            | Home Owner              | Has a place to group devices, members, and thresholds     | Must     | 2         | Backlog | Phase 1      |
| 6    | HU-HOME-002  | Home Management             | Join an existing home with an access code                | Home Member             | Can access a household they didn't create                 | Must     | 3         | Backlog | Phase 1      |
| 7    | HU-HOME-003  | Home Management             | Manage home members and their home role                  | Home Owner              | Controls who has access and what they can do              | Must     | 3         | Backlog | Phase 1      |
| 8    | HU-DEV-001   | Devices                     | Register a measuring device and associate it with a home | Home Owner              | Consumption data can be attributed to their home          | Must     | 3         | Backlog | Phase 1      |
| 9    | HU-DEV-002   | Devices                     | Authenticate a device with its own API key               | Device                  | Only authorized devices can publish measurements          | Must     | 3         | Backlog | Phase 1      |
| 10   | HU-MON-001   | Monitoring and Measurements | Define the MQTT topic/payload contract                   | Device / Backend        | Devices and backend exchange measurements reliably        | Must     | 3         | Backlog | Phase 1      |
| 11   | HU-MON-002   | Monitoring and Measurements | Ingest readings over MQTT, idempotently                  | Home Owner / Member     | Consumption is captured automatically, without duplicates | Must     | 5         | Backlog | Phase 1      |
| 12   | HU-MON-003   | Monitoring and Measurements | View consumption history and statistics                  | Home Owner / Member     | Understands how much energy is used, where and when       | Must     | 5         | Backlog | Phase 2      |
| 13   | HU-HOME-004  | Home Management             | Configure a consumption threshold for a home             | Home Owner              | Can define what "too much" means for their household      | Must     | 3         | Backlog | Phase 2      |
| 14   | HU-ALERT-001 | Alerts                      | Raise an alert when consumption exceeds threshold        | Home Owner / Member     | Finds out about a problem while there's time to act       | Must     | 5         | Backlog | Phase 2      |
| 15   | HU-ALERT-003 | Alerts                      | Alert lifecycle: raised → resolved                       | Home Owner / Member     | Can tell current problems apart from handled ones         | Must     | 3         | Backlog | Phase 2      |
| 16   | HU-WEB-001   | Frontend (React)            | Web dashboard with consumption charts                    | Home Owner / Member     | Visually understands consumption                          | Must     | 5         | Backlog | Phase 2      |
| 17   | HU-MOB-001   | Mobile (Android)            | Mobile screens for consumption and alerts                | Home Owner / Member     | Checks consumption/alerts from their phone                | Must     | 5         | Backlog | Phase 2      |
| 18   | HU-SEC-005   | Security                    | Lock account after repeated failed logins                | Registered user         | Reduces impact of brute-force attempts                    | Should   | 2         | Backlog | Phase 2      |
| 19   | HU-DEV-003   | Devices                     | Derive device connectivity status                        | Home Owner / Member     | Knows whether data is current or stale                    | Should   | 2         | Backlog | Phase 2      |
| 20   | HU-ALERT-002 | Alerts                      | Raise an alert when a device stops reporting             | Home Owner              | Finds out about a broken/disconnected device              | Should   | 3         | Backlog | Phase 2      |
| 21   | HU-ALERT-004 | Alerts                      | Deliver alerts as push notifications                     | Home Owner / Member     | Learns about a problem without opening the app            | Should   | 5         | Backlog | Phase 2      |
| 22   | HU-SEC-006   | Security                    | Record an audit log of security-relevant actions         | Admin                   | Can trace who did what, when                              | Should   | 3         | Backlog | Phase 2      |
| 23   | HU-REC-001   | Recommendations             | Generate recommendations from consumption behavior       | Home Owner / Member     | Gets concrete, personalized guidance                      | Should   | 5         | Backlog | Phase 3      |
| 24   | HU-WEB-002   | Frontend (React)            | Fully internationalized interface                        | Any user                | Can use the product in their language                     | Should   | 3         | Backlog | Phase 3      |
| 25   | HU-REC-002   | Recommendations             | Deliver recommendations via push                         | Home Owner / Member     | Sees recommendations without hunting for them             | Could    | 2         | Backlog | Phase 3      |
| 26   | HU-MON-004   | Monitoring and Measurements | Compare consumption across periods                       | Home Owner / Member     | Can tell whether habits reduced consumption               | Could    | 3         | Backlog | Phase 3      |
| 27   | HU-SEC-007   | Security                    | Configure notification channel, theme, language          | Registered user         | Can tailor the app to preferences                         | Could    | 3         | Backlog | Phase 3      |
| 28   | HU-SEC-008   | Security                    | Security administration: password policy                 | Admin                   | Can tune security behavior without a code change          | Could    | 3         | Backlog | Phase 4      |

**Status legend:** Backlog (identified, not yet detailed) · Ready (meets Definition of Ready) ·
In Sprint (being built) · Done (meets Definition of Done).

## Rules for keeping this list healthy

1. Order by value, not by convenience.
2. Split anything sized 8 or 13 before marking it Ready.
3. Re-rank at every backlog refinement, not just when new items are added.
4. An item without a clear user role and value statement is not ready to be ranked.
5. Do not delete completed items — mark them Done and move them down.

---

## Correlations

- Full user story format → `04-requirements/_template-hu.md`
- Refined, ranked version of this list → `03-product/product-backlog.md`
- Prioritization feeds → `03-product/roadmap.md`
- Vision this backlog implements → `03-product/vision.md`
