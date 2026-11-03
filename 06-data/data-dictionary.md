# Data Dictionary

> This document does not repeat every column already listed in `06-data/models.md`. It exists
> only for fields whose name and SQL type are not enough to use them correctly — because they
> carry a business rule, a unit, a hidden state transition, or a risk of confusion with another
> field. If a field is self-explanatory from its name and type alone, it stays out of this file.

---

| Field | Service | Table | Type | Detailed description | Possible values |
|-------|---------|-------|------|------------------------|------------------|
| `password_hash` | en-monitor-security-service | `user` | VARCHAR(255) | Never the plain-text password. Must be produced by the project's `PasswordHasher` port (see `02-domain/entities-and-rules.md`), not stored or logged in any other form. | — |
| `email` | en-monitor-security-service | `user` | VARCHAR(255) | Used for login. Always normalized to lowercase before it is stored or compared — `user@X.com` and `USER@x.com` must resolve to the same account. | — |
| `status` | en-monitor-security-service | `user` | ENUM | Account lifecycle. `BLOCKED` is set automatically once `failed_login_attempts` reaches the threshold defined in `security_configuration`, not manually by an admin action. | `ACTIVE`, `INACTIVE`, `BLOCKED` |
| `failed_login_attempts` | en-monitor-security-service | `user` | INT | Counts consecutive failed logins. Resets to `0` on a successful login. Reaching the configured maximum flips `status` to `BLOCKED` (see `RegisterLoginErrorUseCase` in `entities-and-rules.md`). | — |
| `revoked` | en-monitor-security-service | `user_session` | BOOLEAN | A session can be invalid for two independent reasons: it was explicitly `revoked`, or it simply reached `expiration_at`. Checking only one of the two gives a false sense of validity — `isValid()` must check both. | — |
| `enabled` | en-monitor-security-service | `system_role` | BOOLEAN | A disabled role is not deleted. Existing `user_system_role` assignments to it remain in the table but stop granting access — this preserves the audit trail of who once had the role. | — |
| `code` | en-monitor-security-service | `permission` | VARCHAR(30) | The stable identifier checked in authorization logic (e.g. `DEVICE_CREATE`). `name` is only for display; code must never be used for anything user-facing, and name must never be used in an authorization check. | e.g. `USER_READ`, `DEVICE_CREATE`, `HOME_UPDATE` |
| `access_code` | en-monitor-home-service | `home` | VARCHAR(8) | The code a user types to **join** an existing home. Not the same as `id_home`, which is the technical primary key and is never shown to the user. | — |
| `use_system_defaults` | en-monitor-home-service | `home_thresholds` | BOOLEAN | When `true`, `daily_limit` and `monthly_limit` on this row are ignored at evaluation time — the system falls back to the platform-wide defaults (`SystemConfig`, per the Singleton pattern in the class diagram) instead of the home's own values. | — |
| `daily_limit`, `monthly_limit` | en-monitor-home-service | `home_thresholds` | DOUBLE | Both expressed in kWh. `daily_limit` should not exceed `monthly_limit` — this is not enforced by a database constraint, only by the `Threshold` value object at the application layer (see `entities-and-rules.md`). | — |
| `installation_date` | en-monitor-device-service | `device` | TIMESTAMP | The business-meaningful moment the physical device was installed. Do not confuse with `created_at`, which is the technical moment the database row was inserted — the two can differ, for example if a device is registered in the system days after it was physically installed. | — |
| `active_power` | en-monitor-measurement-service | `measurement` | DOUBLE | Instantaneous power, in **watts (W)**. A rate, not an accumulated amount. Do not confuse with `stored_energy` — see `02-domain/domain-map.md`, "Terms that must not be confused". | — |
| `stored_energy` | en-monitor-measurement-service | `measurement` | DOUBLE | Accumulated energy, in **kilowatt-hours (kWh)**. This is the figure comparable to `daily_limit` / `monthly_limit`; `active_power` is not. | — |
| `min_limit`, `max_limit` | en-monitor-measurement-service | `consumption_level` | DOUBLE | Define the range a measurement's consumption must fall into for this level to apply. Ranges across the four levels (`LOW`…`CRITICAL`) must not overlap or leave gaps — not enforced by a database constraint. **Open question, not yet resolved:** whether these are absolute kWh values or a percentage of the home's threshold (see `01-context/glossary.md` history). | — |
| `consumption_level_id`, `measurement_id` | en-monitor-alert-service | `alert` | VARCHAR(10) | Both nullable, but not independently optional: a `THRESHOLD` alert always has both populated; a `CONNECTIVITY` alert always has both `NULL`, since it was not triggered by a measurement. | — |
| `alert_status` | en-monitor-alert-service | `alert` | ENUM | Lifecycle of the alert row itself. This is different from severity: there is no `severity` column in this table at all — see below. | `PENDING`, `RESOLVED` |
| *(no column — by design)* | en-monitor-alert-service | `alert` | — | Severity (`WARNING` / `CRITICAL`) is **never persisted**. It is derived at request time from `consumption_level_id` for `THRESHOLD` alerts, or from the connectivity rule for `CONNECTIVITY` alerts. Looking for a `severity` column in this table is a common mistake — it does not exist on purpose. | `WARNING`, `CRITICAL` (computed, not stored) |
| `message_key`, | en-monitor-alert-service, en-monitor-recommendation-service | `alert`, `recommendation` | VARCHAR(100) | A lookup key into the application's message catalog, resolved to the user's language at presentation time. Never store literal, untranslated text in this column. | — |
| `status` | en-monitor-recommendation-service | `recommendation` | ENUM | Whether the user has read the recommendation. Unrelated to `alert_status` in the `alert` table, despite the similar-sounding name — the two tables use different enums for different lifecycles. | `READ`, `UNREAD` |

---

## Correlations

- Full per-table column listing → `06-data/models.md`
- Business rules and invariants behind these fields → `02-domain/entities-and-rules.md`
- Terms that must not be confused (source for several rows above) → `02-domain/domain-map.md`