# Normalization Assessment

> This document evaluates the normal form of the current schema and justifies any intentional
> denormalization. As of this writing, the schema has **no intentional denormalization** — every
> table is in Third Normal Form (3NF). This is stated plainly rather than manufactured content to
> fill the section.

---

## What each normal form actually checks

| Form | Requirement | Violated by |
|------|-------------|-------------|
| 1NF | Every column holds a single, atomic value — no repeating groups, no comma-separated lists in one field | A column like `permissions: "USER_READ,DEVICE_CREATE"` instead of a join table |
| 2NF | Every non-key attribute depends on the **whole** primary key, not just part of it | In a composite-key table, a column that only depends on one of the two key columns |
| 3NF | Every non-key attribute depends **only** on the key, not on another non-key attribute | Storing `home_type_name` directly on `home` instead of `home_type_id` referencing a separate table |

---

## 1NF — checked against this schema

No column stores more than one value. Where a real-world concept is naturally multi-valued (a
user can hold several system roles, a home can have several members), the schema uses a join
table instead of a delimited list:

- `user_system_role` (composite key: `user_id`, `system_role_id`) instead of a `roles` text
  column on `user`.
- `user_home` (composite key: `user_id`, `home_id`) instead of a list of home IDs on `user`.
- `device_home` (composite key: `device_id`, `home_id`) for the same reason.

## 2NF — checked against this schema

Every table with a composite primary key was reviewed for attributes that depend on only half of
that key:

- `user_system_role.assigned_at` describes the *pairing* of that user with that role, not the
  user alone or the role alone — correct.
- `system_role_permission.assigned_at` — same reasoning, describes the pairing.
- `user_home.role`, `user_home.favorite` — both describe this specific user's relationship to
  this specific home, not the user in general (a user can be `OWNER` of one home and `MEMBER` of
  another) — correct.
- `device_home` has no non-key attributes beyond the audit fields, so there is nothing to check.

No partial dependency was found.

## 3NF — checked against this schema

Every reference to a fixed category is a foreign key to its own table, never an inlined value
that depends on that category rather than on the row's own key:

| Table | Category | Stored as |
|-------|----------|-----------|
| `home` | Home type | `home_type_id` → `home_type` |
| `device` | Appliance type | `appliance_type_id` → `appliance_type` |
| `alert` | Consumption level | `consumption_level_id` → `consumption_level` |

None of these tables store the category's name, description, or range values directly — only the
reference. Renaming a home type, for example, requires changing exactly one row in `home_type`,
never a bulk update across every `home` that uses it.

---

## Why severity is not a case of "should have been denormalized"

`alert` has no `severity` column, even though severity is exactly the kind of value a team is
often tempted to cache for read performance. It was deliberately left out and is computed at
request time from `consumption_level_id` (see `02-domain/entities-and-rules.md`). This is not an
oversight to fix — caching it would be the denormalization, not the current design.

---

## This is not the same question as the cross-context foreign key decision

`06-data/modeling-conventions.md` documents that foreign keys crossing a bounded context (for
example `alert.home_id`) are not enforced as database constraints. **This is not
denormalization.** Normalization is about whether data is duplicated or transitively dependent;
the cross-context FK decision is about which service's migration owns a constraint. `alert.home_id`
still stores a single, non-redundant reference to exactly one home — the schema itself is just as
normalized whether or not MySQL enforces that reference for you.

---

## A denormalization this project has not made, and the trade-off if it ever does

If `alert` or `recommendation` start needing to display a home's name or a device's name in a
list view without calling another service on every request, a common shortcut is to cache that
name directly on the row (`alert.home_name`) and accept that it can go stale if the home is
renamed. This has not been done, and should not be done silently — if performance ever makes it
necessary, it belongs in this document as a new, explicitly justified entry, with a stated policy
for how the cached copy gets refreshed.

---

## Correlations

- Full schema → `06-data/models.md`
- Naming, ID format, and the cross-context FK rule → `06-data/modeling-conventions.md`
- Domain entities and their invariants → `02-domain/entities-and-rules.md`