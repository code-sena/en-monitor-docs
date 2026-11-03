# ADR-009 — Liquibase Changelogs per Service, One Shared Changelog Table

- **ID:** ADR-009
- **Date:** 2026-10-20
- **Status:** Accepted
- **Authors:** Juan Esteban Cortés Parra

---

## Context

`ADR-006` decided on one separate migrations repository with a folder per bounded context. That
repository was never created: from the first schema (September 2026) the Liquibase changelogs
lived inside `energy-monitor-backend`, one folder per module (`db/changelog/<context>/`), and the
backend applied them on startup.

When the contexts were extracted into their own services (Security, Devices, Monitoring/Measurements, then Home, Notification, Alerts and Recommendations), their
tables stayed in the same shared schema (`ADR-003-shared-database.md`) with data already in
production. The changesets that created those tables had already been applied by the monolith
and were recorded in `DATABASECHANGELOG`. Moving them must not make Liquibase run them a second
time.

**Known constraints:**
- One shared MySQL 8.4 schema with production data; tables are not recreated or copied
- Liquibase identifies a changeset by id, author and file path
- Each service is deployed on its own (`infra/deploy/deploy.sh <component>`), so a service has to
  be able to apply its own schema changes without the others

---

## Decision

**We decided:** each component keeps the Liquibase changelogs of the tables it owns **inside its
own application repository**, and applies only those on startup. All of them share the one
`DATABASECHANGELOG` table of the schema. A changelog that moves to a new service keeps **the
same relative path** it had in the monolith, so the changesets already applied are recognised
and never run again.

| Component | Changelog folder | Changesets (28-Oct-2026) |
|---|---|---|
| `en-monitor-security-service` | `db/changelog/security/` | `security-001` … `security-030` (`security-026`: `security_outbox`) |
| `en-monitor-device-service` | `db/changelog/device/` | `device-001` … `device-009` (`device-009`: `device_outbox`) |
| `en-monitor-measurement-service` | `db/changelog/measurement/` | `measurement-001` … `measurement-004` (`measurement-004`: `measurement_outbox`) |
| `en-monitor-home-service` | `db/changelog/home/` | `home-001` … `home-007` (`home-007`: `home_outbox`) |
| `en-monitor-notification-service` | `db/changelog/notification/` | `notification-001` … `notification-007` |
| `en-monitor-alert-service` | `db/changelog/alert/` | `alert-001` … `alert-004` (`alert-004`: `alert_outbox`) |
| `en-monitor-recommendation-service` | `db/changelog/recommendation/` | `recommendation-001`, `recommendation-002` (`recommendation_outbox`) |

Rules kept from `06-data/migration-strategy.md`: forward-only changes, one changeset per logical
change, a `rollback` block on every changeset, and ids of the form
`{context}-{sequence}-{description}`.

**Justification:** a migration then ships, is reviewed and is deployed together with the code
that needs it, and each service can be deployed alone. Keeping the paths unchanged was the
condition for moving the changelogs without touching production data.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Changelogs in each service repository, shared `DATABASECHANGELOG`, same paths (chosen)** | Schema change and code ship together; each service deploys alone; applied changesets are recognised after a move | The full schema history is spread over four repositories; nothing in the database stops a service from touching another's tables | — (chosen) |
| One separate migrations repository (`ADR-006`) | One place for the whole history | One more repository and pipeline; schema changes deploy apart from the code that needs them | Never created; superseded by this ADR |
| One changelog table per service | Each service's history fully isolated | Every moved changeset would look new to its service and run again against existing tables | Would break the move of tables that already hold data |

---

## Consequences

**Positive:**
- No changeset ran twice during the three extractions; the tables and their data stayed in place
- Each `deploy.sh <component>` applies that component's pending changesets on startup

**Negative / Trade-offs:**
- A moved changelog must never be renamed or moved to another path, or Liquibase sees it as new
- The schema history is read across four repositories instead of one
- The first start against an empty database takes about two minutes while all changesets run
  (infra README, troubleshooting)

**Impact on the system:**
- Affected: `en-monitor-security-service`, `en-monitor-device-service`,
  `en-monitor-measurement-service`, `energy-monitor-backend`
- Documents updated with this ADR: `06-data/migration-strategy.md`, `06-data/models.md`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A changelog path changes during a later extraction (Home, Alerts, Recommendations, Notification) and its changesets run again | Medium during a move | High | Move changelogs with the same relative path, as the three extractions did; check `DATABASECHANGELOG` after the first start |
| Two services create changesets with the same id | Low | Medium | Ids carry the context prefix |

---

## References

- Supersedes → `ADR-006-migrations-repository-structure.md`
- Shared schema → `ADR-003-shared-database.md`
- Migration rules → `06-data/migration-strategy.md`
- Code: `docs/Architecture.md` ("Database") of `en-monitor-security-service`,
  `en-monitor-device-service` and `energy-monitor-backend`; infra README (first deployments of
  each service)
