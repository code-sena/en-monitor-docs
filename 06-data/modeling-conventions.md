# Modeling Conventions

> This file is the canonical source for naming, ID format, timestamps, soft delete, auditing,
> units, and enum handling across the project's schema. `06-data/models.md` keeps only the
> reasoning specific to its own content — the shared-database decision and why cross-context
> foreign keys are not enforced at the database level — and points here for everything else.

---

## Primary keys

Primary keys are **not** auto-generated UUIDs. Every table uses a business identifier:

```
id_[entity]   VARCHAR(10)   e.g. id_user, id_home, id_device, id_measurement
```

**Open question, not yet resolved:** the MER fixes the type (`VARCHAR(10)`) but never defines how
the value itself is generated — a sequential counter, a random alphanumeric code, a prefix plus a
number, or something else. This needs to be decided before the first `CreateUserUseCase` or
`CreateHomeUseCase` is implemented, since whichever service creates the row is what has to
generate this value.

## Foreign keys

```
[referenced_entity]_id   VARCHAR(10)   e.g. user_id, home_id, device_id
```

The referenced entity's name, not the referencing table's own name — a foreign key is always
named after what it points to.

**Cross-context foreign keys are not enforced as database constraints.** A column like
`alert.home_id` exists and is populated, but no `FOREIGN KEY ... REFERENCES` is declared for it,
even though `alert` and `home` sit in the same physical MySQL instance. Foreign keys **within**
the same bounded context (e.g. `user.person_id` inside Security) are real constraints. The full
reasoning is in `06-data/models.md`, section "Data modeling principles" — this file states the
rule, that file explains why.

This boundary is enforced one level up as well: each domain has its own Liquibase repository, and
that repository may only define or alter tables belonging to its own domain (see
`06-data/migration-strategy.md`). A domain never gains write access to another domain's schema,
even temporarily during a migration.

## Table and column naming

```
Tables:   snake_case, singular   → home, user_session, device_status_log
Columns:  snake_case             → active_power, installation_date, access_code
Enums:    native MySQL ENUM(...) → status ENUM('ACTIVE','INACTIVE','BLOCKED')
```

Table names are singular, matching the entity names in the MER (`02-domain/entities-and-rules.md`
and the MER diagram) one-to-one, rather than the plural convention some frameworks default to.
This was a deliberate choice to keep the table name and the domain entity name identical, so
there is never a translation step between "which entity is this" and "which table is this."

One consequence to know about: `user` is a reserved name in MySQL (it is the system's own account
table). Every query against this project's `user` table needs backticks (`` `user` ``) to avoid
ambiguity with MySQL's internal table of the same name.

## Timestamps

```
TIMESTAMP     -- not TIMESTAMPTZ, which MySQL does not support
```

`TIMESTAMPTZ` belongs to PostgreSQL. MySQL's `TIMESTAMP` stores and converts values based on the
server's configured time zone rather than an explicit per-row offset — this is a real behavioral
difference, not just a naming one, and matters if the deployment environment's time zone ever
changes.

## Auditing

Every table includes:

```sql
created_at  TIMESTAMP  NOT NULL  DEFAULT CURRENT_TIMESTAMP,
updated_at  TIMESTAMP  NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
deleted_at  TIMESTAMP            -- NULL = active (soft delete)
```

`created_by` / `updated_by` columns are **not** currently part of the schema. The project has no
stated requirement to track which user made a given change at the row level — `audit_log` in
Security already covers who performed a security-relevant action. If a future requirement needs
to know who created or modified an arbitrary row (not just security actions), that is a schema
change to make explicitly, not something to assume is already covered.

## Soft delete

No table uses a physical `DELETE`. `deleted_at IS NOT NULL` marks a row as deleted; `IS NULL`
means active. Every query that lists records must filter on `deleted_at IS NULL` unless it has an
explicit reason to include deleted rows (e.g. an admin recovery screen).

## Business dates vs audit dates

Several tables carry both a business-meaningful date and a technical audit timestamp that are
easy to confuse:

| Business date | Audit timestamp | Why they can differ |
|-----------------|-------------------|------------------------|
| `home.creation_date` | `home.created_at` | The home could have existed informally before it was registered in the system |
| `device.installation_date` | `device.created_at` | A device can be installed physically days before someone registers it |

Business dates describe when something happened in the real world. Audit timestamps describe when
the database row was written. Do not use one to answer a question that belongs to the other.

## Units

Two columns are easy to confuse because they are both plain `DOUBLE` values describing
electricity:

| Column | Unit | Meaning |
|--------|------|---------|
| `measurement.active_power` | watts (W) | Instantaneous rate |
| `measurement.stored_energy` | kilowatt-hours (kWh) | Accumulated amount |

`home_thresholds.daily_limit` and `.monthly_limit` are also kWh, comparable to `stored_energy` —
never to `active_power`. See `02-domain/domain-map.md`, "Terms that must not be confused".

## Enumerations

Enumerated values are stored as native MySQL `ENUM(...)` columns, not as a separate lookup table
and not as a free-text `VARCHAR` with an application-level check. The allowed values for each one
are listed once, in `06-data/models.md`, next to the table that defines them — they are not
repeated here.

---

## Correlations

- Full schema and the reasoning behind the cross-context FK rule → `06-data/models.md`
- Fields with hidden business rules → `06-data/data-dictionary.md`
- Domain-level definitions behind these conventions → `02-domain/entities-and-rules.md`