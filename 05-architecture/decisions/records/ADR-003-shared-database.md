# ADR-003 — Shared Physical Database

- **ID:** ADR-003
- **Date:** 2026-09-10
- **Status:** Proposed
- **Authors:** Miguel Ange Garcia and Juan Esteban Cortes Parra

---

## Context

The system is built as six microservices (`security-service`, `home-service`,
`device-service`, `measurement-service`, `alert-service`, `recommendation-service`, per
`02-domain/domain-map.md` and `05-architecture/overview.md`). Every one of them needs a place to
persist its own bounded context's data, and this decision has already been assumed and cited as
pending in seven places (`06-data/models.md` ×6, `01-context/glossary.md` ×1) without ever being
formalized as an ADR — this document closes that gap, it does not introduce a new option.

**Known constraints:**

- Two-person team, no dedicated database administration role
- Six services need to exchange references to each other's entities (e.g. an alert needs a
  `homeId`, a device needs to be looked up by a `deviceId` from more than one service)
- The team's chosen stack is Java 21 + Spring Boot with MySQL 8 (`01-context/overview.md`)

---

## Decision

**We decided:** one physical MySQL 8 instance, shared by all six microservices, organized
logically by bounded context. Each service has read/write access only to its own context's
tables; no service queries or joins another service's tables directly, and cross-context
references (e.g. `alerts.home_id`) are stored as plain columns without a database-level
`FOREIGN KEY` constraint.

**Justification:** A true database-per-service split, with six separate physical database
instances, adds real operational cost (six connections/credentials to provision and back up,
cross-service data access that always has to go over the network even for simple lookups) that a
two-person team does not have the capacity to carry well under a fixed deadline. A single shared
instance gets almost all of that isolation — enforced by an access-boundary convention rather
than by six separate server processes — at a fraction of the operational cost. What matters is
enforcing the discipline that _conceptually_ keeps each service independent (no direct queries
or joins across contexts), not whether the bytes physically live in one server process or six.

---

## Evaluated alternatives

| Alternative                                                                    | Pros                                                                                                                                                                                                                | Cons                                                                                                                                                                                     | Reason for discarding                                                                                                         |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **One shared MySQL instance, access-boundary enforced by convention — chosen** | One instance to provision, back up, and monitor; cross-context lookups can still use a simple SQL query during development even though the access boundary is respected; keeps the option to split physically later | Nothing at the database engine level stops a service from breaking the access-boundary convention — it has to be enforced by code review and discipline, not by the database itself      | — (chosen)                                                                                                                    |
| One physical database per service                                              | Real, enforced isolation — a service literally cannot query another's tables even by mistake; matches the canonical microservices pattern                                                                           | Six databases to provision, secure, and back up; every cross-context reference becomes a network call even for a trivial lookup; disproportionate operational cost for a two-person team | Discarded — the isolation benefit doesn't outweigh its cost at this team size                                                 |
| A single database with cross-context foreign keys enforced at the SQL level    | Referential integrity guaranteed by the database itself                                                                                                                                                             | Forces cross-context migrations to run in a fixed order (e.g. `alert-service` always after `home-service`), and hard-blocks ever physically separating the databases later               | Discarded — see `06-data/models.md`'s own reasoning; this project deliberately wants to keep a future physical split possible |

---

## Consequences

**Positive:**

- One database instance to operate, consistent with the team's actual capacity
- Cross-context lookups remain simple SQL during development, without giving up the conceptual
  boundary between services
- The path to a future physical split is not blocked — no cross-context FK constraint locks
  services' migration ordering together

**Negative / Trade-offs:**

- The access-boundary rule (a service only touches its own tables) is enforced by convention and
  code review, not by the database engine — a mistake is possible in a way it wouldn't be with
  physically separate databases
- Referential integrity across contexts (e.g. an `alert` pointing at a `home` that no longer
  exists) is not caught automatically by the database; it has to be handled at the application
  level

**Impact on the system:**

- Affected services: all six (`security-service`, `home-service`, `device-service`,
  `measurement-service`, `alert-service`, `recommendation-service`)
- Documents that must be updated once this is Accepted: the seven existing "pending" citations —
  `06-data/models.md` (six occurrences) and `01-context/glossary.md` (one occurrence) — should
  be updated from "to be formalized in ADR-003-shared-database.md, pending" to a direct
  reference to this (now-written) ADR

---

## Risks

| Risk                                                                                                                                    | Probability                                                        | Impact                                                              | Mitigation                                                                                                                                           |
| --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| A service's code queries or joins another context's tables directly, silently coupling two services that are supposed to be independent | Medium — nothing at the database level prevents this               | Medium–High (undermines the whole point of the microservices split) | Enforce in code review; consider a lightweight automated check (e.g. a linter rule or a repository-layer convention) that flags cross-schema queries |
| Migrations for two contexts collide or need coordinated ordering despite the "no cross-context FK" rule                                 | Low, if `06-data/migration-strategy.md`'s conventions are followed | Medium                                                              | Each service's changelog stays self-contained, per the migrations-repository decision (see `ADR-006-migrations-repository-structure.md`)             |
| The shared instance becomes a single point of failure or a performance bottleneck as all six services load it simultaneously            | Low, at this project's expected scale                              | Medium                                                              | Revisit only if load testing shows an actual problem — do not pre-optimize for a scale this project hasn't reached                                   |

---

## References

- `06-data/models.md` — the six per-context sections already citing this decision as pending, and
  the "Distributed Monolith" anti-pattern this decision avoids
- `01-context/glossary.md` — `Shared Database` and `Distributed Monolith` definitions
- Related to: `ADR-006-migrations-repository-structure.md` (the migration-ordering consequence of
  not enforcing cross-context foreign keys, referenced above)
- `09-microservices/services/01-api-gateway/README.md` also references an "ADR-003" for a
  different topic ("Authentication strategy at the gateway vs. in each service") — that citation
  predates this document and should be corrected to avoid a second, conflicting ADR-003; see the
  note accompanying this batch of ADRs.
