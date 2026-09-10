# Migration Strategy

---

## Tool

**Liquibase**, kept separate from each service's own application code. Whether that means one
shared migrations repository with changelogs organized per domain, or one separate repository per
domain, is a decision still pending — not assumed here.

## Migration rules

```
✓ Migrations are ALWAYS forward-only
✓ One changeset per logical change
✓ Every changeset has a corresponding rollback block
✗ Never modify a changeset already executed in any environment
✗ Never DROP COLUMN / DROP TABLE if code still reads or writes it
    (process: 1-deprecate in code → 2-cleanup migration in the next release)
```

---

## Rollback policy

Every changeset must declare how to undo it. Liquibase supports this per changeset:

```xml
<changeSet id="005-add-color-theme-to-user-configuration" author="team">
    <addColumn tableName="user_configuration">
        <column name="color_theme" type="VARCHAR(50)"/>
    </addColumn>
    <rollback>
        <dropColumn tableName="user_configuration" columnName="color_theme"/>
    </rollback>
</changeSet>
```

Two cases where a mechanical rollback is not enough:

- **A changeset that deletes or transforms data** (for example, a script that backfills
  `stored_energy` for old rows) cannot be undone by reversing the SQL alone, because the original
  values are gone. The rollback for this kind of changeset must restore from a backup taken
  immediately before running it, not attempt an automatic reverse migration.
- **A changeset already run in production** is never edited to fix a mistake. A new changeset is
  written to correct it going forward, even if that means the schema briefly carried the mistake.
  Editing an already-applied changeset breaks Liquibase's checksum tracking and makes environments
  diverge silently.

---

## Handling migrations with data already in production

The risk is not the schema change itself — it is applying it to a table that already has rows
that do not fit the new shape. Three situations this project is likely to hit, given the schema
in `06-data/models.md`:

**Adding a `NOT NULL` column to a table that already has rows.**
Adding `NOT NULL` without a `DEFAULT` fails immediately on any populated table. Always provide a
default, or split it into two changesets: add the column as nullable, backfill it with a script,
then add the `NOT NULL` constraint in a later changeset once every row has a value.

**Adding a value to an existing `ENUM` column.**
MySQL requires an `ALTER TABLE ... MODIFY COLUMN` that rewrites the entire enum definition — this
locks the table for the duration on large tables. For a table expected to grow large, such as
`measurement`, this should run during a maintenance window, not as a normal deploy-time migration.

**Removing an `ENUM` value that existing rows still use.**
Never remove an enum value while any row still holds it. First migrate every existing row away
from that value in a data-migration changeset, confirm none remain, and only then remove the
value from the enum definition in a later changeset.

---

## Compatible schema changes (non-breaking)

```sql
-- Add a nullable column → always safe
ALTER TABLE home ADD COLUMN notes TEXT;

-- Add a NOT NULL column with a valid DEFAULT → safe
ALTER TABLE device ADD COLUMN firmware_version VARCHAR(20) NOT NULL DEFAULT 'unknown';

-- Create a new index → safe
CREATE INDEX idx_measurement_device_id ON measurement (device_id);
```

## Incompatible changes (require a 2-phase migration)

```sql
-- Rename a column → 2 phases:
-- Phase 1 (release N): add the new column, copy data, update code to read/write both
ALTER TABLE alert ADD COLUMN severity_label VARCHAR(20);
UPDATE alert SET severity_label = old_column;

-- Phase 2 (release N+1): remove the old column, once no code reads it anymore
ALTER TABLE alert DROP COLUMN old_column;
```

---

## Correlations

- Full schema this strategy applies to → `06-data/models.md`
- Naming and ID conventions referenced above → `06-data/modeling-conventions.md`
- Fields most likely to need a careful, staged migration → `06-data/data-dictionary.md`