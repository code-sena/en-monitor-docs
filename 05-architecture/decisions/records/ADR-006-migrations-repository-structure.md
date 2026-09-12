# ADR-006 — Database Migrations Repository Structure

> **Numbering note:** originally proposed as ADR-004. Renumbered to 006 because ADR-004 is
> already claimed by the API Gateway decision, cited four times across three files
> (`05-architecture/README.md`, `overview.md` ×2, `deployment.md`) before this document was
> written. See the note accompanying this batch of ADRs for the full explanation.

- **ID:** ADR-006
- **Date:** 2026-09-10
- **Status:** Accepted
- **Authors:** Miguel Angel Garcia Artunduaga and Juan Esteban Cortes Parra

---

## Context

`06-data/migration-strategy.md` already commits to Liquibase, kept separate from each service's
own application code, but explicitly leaves one question open: _"whether that means one shared
migrations repository with changelogs organized per domain, or one separate repository per
domain, is a decision still pending."_ `01-context/scope.md` lists "Database migrations
repository (Liquibase, with per-domain changelogs)" as an external dependency, status
🟡 In progress — so the tool is decided, but the repository structure is not, and it's a
blocker: `01-context/scope.md` itself notes this is "needed before any service connects to a
real schema."

**Known constraints:**

- Six services (`security-service`, `home-service`, `device-service`, `measurement-service`,
  `alert-service`, `recommendation-service`), each owning its own tables in the one shared MySQL
  instance (`ADR-003-shared-database.md`)
- Two-person team — minimizing the number of repositories to clone, configure CI for, and keep
  in sync is a real, not academic, concern
- `06-data/migration-strategy.md`'s rules (forward-only migrations, one changeset per logical
  change, mandatory rollback blocks) apply regardless of which repository structure is chosen —
  this ADR does not revisit those rules

---

## Decision

**We decided:** one shared migrations repository, with Liquibase changelogs organized into one
subfolder per bounded context (matching the six services), rather than six separate
repositories.

**Justification:** With a single shared MySQL instance (`ADR-003-shared-database.md`), the
migrations that create and evolve that instance's schema are naturally a single, ordered stream
of changes — even though each changeset only touches one context's tables. A single repository
gives one place to see the full schema history in order, one CI pipeline to validate that
changesets apply cleanly against a fresh database, and one clone for a two-person team to keep
track of. Splitting into six repositories would multiply that setup work six times over for a
project where no two services are ever migrated by different, independent teams working out of
sync with each other.

---

## Evaluated alternatives

| Alternative                                                                                           | Pros                                                                                                                                                                    | Cons                                                                                                                                                                                                                                       | Reason for discarding                                                                                                                             |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **One shared repository, per-context subfolders — chosen**                                            | One clone, one CI pipeline, one place to see the full migration history in execution order; still keeps each context's changelog visually and organizationally separate | Requires a folder-naming/ordering convention so changesets from different contexts don't collide or apply out of the intended order                                                                                                        | — (chosen)                                                                                                                                        |
| One repository per service/context                                                                    | Matches "one service, one everything" microservices purity; a context's migration history is fully isolated                                                             | Six repositories to clone, configure, and run CI for for a team of two; no natural place to see the instance's full schema history in one view, despite it being one physical database                                                     | Discarded — the isolation this buys doesn't matter when all six contexts land in the same physical instance (`ADR-003-shared-database.md`) anyway |
| Migrations live inside each service's own application repository (no separate migrations repo at all) | Simplest possible setup — no extra repository                                                                                                                           | `migration-strategy.md` already establishes migrations are kept **separate** from application code; mixing them back in would contradict that existing decision and make it harder to run migrations independently of a service deployment | Discarded — reopens a question `migration-strategy.md` already settled                                                                            |

---

## Consequences

**Positive:**

- One repository and one CI pipeline for all schema changes, appropriately sized for a
  two-person team
- The full, ordered migration history of the single shared database instance is visible in one
  place, matching the fact that it really is one physical database (`ADR-003-shared-database.md`)
- Per-context subfolders keep the organizational separation `migration-strategy.md`'s conventions
  already assume, without needing six repositories to get it

**Negative / Trade-offs:**

- A naming/ordering convention across subfolders has to be established and followed (e.g. a
  global changeset ID sequence, or a per-context prefix) so two contexts' changesets don't
  collide when applied against the one shared instance
- Any future physical database split (kept open by `ADR-003-shared-database.md`) would require
  splitting this repository too, at that point — an accepted, deferred cost, not a reason to
  choose differently now

**Impact on the system:**

- Affected: all six services' schema (indirectly — the services themselves are unaffected by
  where their migrations live, only by what schema results)
- Documents that must be updated: `01-context/scope.md`'s dependency row ("🟡 In progress") should
  be updated to reference this ADR and marked resolved once the repository actually exists;
  `06-data/migration-strategy.md`'s open question should link here instead of leaving the
  question unresolved

---

## Risks

| Risk                                                                                                                                    | Probability                                     | Impact | Mitigation                                                                                                                                       |
| --------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Two changesets from different context subfolders are assigned conflicting IDs or apply out of intended order                            | Medium, without an explicit convention          | Medium | Establish and document a changeset ID convention (e.g. `{context}-{sequence}-{description}`) as part of implementing this ADR, not left implicit |
| The single migrations repository becomes a bottleneck if both developers need to add migrations for different contexts at the same time | Low, at a two-person team's actual working pace | Low    | Standard git branching/PR review is enough at this scale; revisit only if it's genuinely causing friction                                        |

---

## References

- `06-data/migration-strategy.md` — the open question this ADR resolves
- `01-context/scope.md` — the dependency entry that flagged this as blocking, in progress
- Related to: `ADR-003-shared-database.md` (this decision follows directly from the shared
  physical instance that ADR establishes)
