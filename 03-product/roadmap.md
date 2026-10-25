# Roadmap

> Delivery plan over time. Evaluated and adjusted at every retrospective — this is a living
> document, not a one-time plan.

---

## Planning assumptions

| Assumption              | Value                                                                    | Source                                                   |
| ----------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------- |
| Hard deadline           | Last week of October 2026 (October 31, 2026)                             | `01-context/scope.md` — SENA ADSO schedule, group 314556 |
| Project start           | May 1, 2026                                                              | Project initiation                                       |
| Timeline                | 6 months (~26 weeks)                                                     | Adjusted planning schedule                               |
| Team                    | 2 developers (Juan Esteban Cortés Parra, Miguel Ángel García Artunduaga) | `01-context/overview.md`                                 |
| Sprint length           | 1 week (assumed — `agile-conventions.md` has this as an open field)      | To confirm with the team                                 |
| Current status          | In development. Next milestone: build the `Security` bounded context     | `01-context/overview.md`                                 |
| Key external dependency | PZEM-004T V3 hardware units expected **September 15, 2026**              | `01-context/scope.md`                                    |

---

## Phase 1 — Foundation & Core Loop (Sprints 1–8 · May 1 – June 30)

Goal: a user can register, create or join a home, register a device, and see that device's
measurements arrive in the system. No alerts or recommendations yet — this phase proves the
data actually flows end to end.

- User registration, login/logout, password recovery (`Security`)
- Access and refresh tokens, session management (`Security`)
- System roles and RBAC enforced on protected endpoints (`Security`)
- Security configuration (lockout rules, session duration) and account lockout (`Security`)
- Create a home, join a home with an access code, manage home members and their home role
  (`Home Management`)
- Register a measuring device, associate it with a home, authenticate it with an API key
  (`Devices`)
- MQTT broker selection and the topic/payload contract between device and backend
  (`Monitoring and Measurements`) — **blocked on hardware arrival (September 15)**; the contract can
  be designed and stubbed before the physical units arrive
- Idempotent ingestion of voltage, current, active power, and energy readings over MQTT
  (`Monitoring and Measurements`)
- Web app skeleton wired to authentication (`Frontend — React`)
- Android app skeleton wired to authentication (`Mobile — React Native`)

**Exit criterion:** a registered user can log in, create a home, link a device, and see at
least one real measurement recorded against that home in the database.

---

## Phase 2 — Consumption Visibility & Alerts (Sprints 9–16 · July 1 – August 31)

Goal: the core value proposition becomes visible — households can see their consumption and
get warned when something is off.

- Consumption history and statistics, viewed over time, per home and per device
  (`Monitoring and Measurements`)
- Threshold configuration per home, with system defaults as fallback (`Home Management` /
  `Alerts`)
- Device status derived from reporting activity (online / not reporting) (`Devices`)
- Threshold-exceeded alerts and device-stopped-reporting alerts, with an alert lifecycle
  (raised → resolved) (`Alerts`)
- Push notifications for alerts, in addition to in-app surfacing (`Alerts`)
- Access and refresh tokens, session management (`Security`)
- System roles and RBAC enforced on protected endpoints (`Security`)
- Audit log for security-relevant actions (`Security`)
- Web dashboard: charts and history views (`Frontend — React`)
- Mobile: consumption and alerts screens (`Mobile — React Native`)

**Exit criterion:** a home exceeding its configured threshold produces a visible alert in both
web and mobile, and a push notification is delivered.

---

## Phase 3 — Recommendations & Internationalization (Sprints 17–20 · September 1 – September 30)

Goal: close the product loop with proactive guidance, and finish the cross-cutting UI work that
was deferred while the core loop was being built.

- Security configuration (lockout rules, session duration) and account lockout (`Security`)
- Recommendations generated from a home's own measured consumption behavior (`Recommendations`)
- Recommendation delivery through the same push channel as alerts (`Recommendations`)
- Interface fully internationalized with `react-i18next` (`Frontend — React`)
- UI polish: comparisons across periods, empty/error states, loading states (`Frontend` /
  `Mobile`)

**Exit criterion:** a home with a week of consumption history receives at least one
recommendation that references its own measured behavior, and the UI is verified in more than
one language.

---

## Phase 4 — Hardening & Delivery (Sprints 21–24 · October 1 – October 31)

Goal: the system is demo-ready and defensible in front of an evaluator, not just functionally
complete.

- End-to-end test pass across Security → Home Management → Devices → Monitoring → Alerts →
  Recommendations
- Security review against `00-governance/security-rules.md` and `security-policy.md`
- Load/idempotence check on MQTT ingestion (duplicate delivery must not double-count energy)
- CI/CD pipelines green for every service (Docker + GitHub Actions)
- Documentation pass: service READMEs, ADRs, OpenAPI contracts up to date with the final
  implementation
- Buffer for any slippage caused by the hardware dependency (Phase 1's MQTT contract work is the
  most exposed item if PZEM units arrive late or behave unexpectedly)
- Final demo script and rehearsal

**Exit criterion:** the system can be demonstrated live, from a physical device measurement to
an alert or recommendation appearing on a phone, without manual intervention.

---

## Explicitly out of this roadmap

Per `01-context/scope.md`, the following are **not** planned in any phase above and should not
be pulled into a sprint without first updating the scope document:

- iOS application
- Manual consumption entry from a utility bill (superseded by automatic MQTT ingestion)
- A dedicated administration UI for Security (endpoints only, operated directly)
- Per-service physical databases (shared MySQL instance is the current architectural decision)
- Energy cost estimation in local currency (no utility tariff data source exists)

---

## Correlations

- Features come from → `03-product/product-backlog.md`
- Feature list source of truth → `01-context/scope.md`
- Definition of Done every story must meet → `00-governance/definition-of-done.md`
- Risks that could shift this roadmap → `15-project-control/risks.md`
