# Data Model — Security Service

> `en-monitor-security-service` is the **authoritative owner** of the tables below. No other
> service reads them directly: the backend asks for permissions, profiles and administrators
> through the internal API, and hears account changes as events.

---

## Database engine

**Engine:** MySQL 8.4, the one shared schema `energy_monitor` (`ADR-003-shared-database.md`).

**Justification:** a platform decision — one instance for the whole system, each service owning
its own tables, no foreign key across services.

---

## Tables

Column-level detail of every table except the outbox is in `06-data/models.md` (Bounded Context:
Security).

```mermaid
erDiagram
    person ||--o| user : "fk_user_person_id"
    user ||--o{ user_session : "fk_user_session_user_id"
    user_session ||--o{ refresh_token : "fk_refresh_token_user_session"
    refresh_token ||--o{ refresh_token : "parent_id: rotation family"
    user ||--o{ password_reset_token : ""
    user ||--o| user_configuration : ""
    user ||--o{ login_error_log : ""
    user ||--o{ audit_log : ""
    user ||--o{ user_system_role : ""
    system_role ||--o{ user_system_role : ""
    system_role ||--o{ system_role_permission : ""
    permission ||--o{ system_role_permission : ""
    security_outbox {
        varchar id_message PK "also the message id on the broker"
        varchar message_type "routing key, e.g. account.deleted"
        text payload
        datetime occurred_at
        datetime expires_at "mail requests expire with their code"
        datetime published_at
        datetime discarded_at
        int attempts
        varchar last_error
    }
```

| Table | Purpose |
|---|---|
| `person` | Personal data (name, last name, profile image) |
| `user` | Account: e-mail, password hash, status, Google subject, soft delete |
| `user_configuration` | Per-user notification and interface preferences |
| `user_session` | Persisted, revocable session |
| `refresh_token` | Hashed refresh tokens; `parent_id` links the rotation family for reuse detection |
| `password_reset_token` | Recovery codes (peppered HMAC), attempts |
| `password_policy`, `security_configuration` | Password rules and system-wide security settings |
| `login_error_log` | Failed sign-ins and their error type (including `EMAIL_NOT_VERIFIED`) |
| `system_role`, `permission`, `system_role_permission`, `user_system_role` | Global RBAC (`ADMIN` seeded by `security-025`) |
| `audit_log` | Who did what, when, from where |
| `security_outbox` | Transactional outbox of account events and mail requests (`security-026`) |

---

## Modeling decisions

- **Outbox in the service's own tables:** the message row is written in the same transaction as
  the change (`ADR-008`).
- **Mail requests carry a code in clear text:** the payload is erased as soon as the broker
  confirms it, and `expires_at` lets the relay discard it unsent once the code is no longer valid.
- **Deleted accounts release their address:** the deleted row keeps
  `deleted+<id_user>@deleted.invalid`, so the same address can register again.

---

## Schema migration

**Tool:** Liquibase, applied by the service on startup (`ADR-009`).
**Script location:** `src/main/resources/db/changelog/security/` (same relative path as in the
monolith, so already-applied changesets are recognised).
**Naming convention:** `security-NNN-description` (`security-001` … `security-027` on 24-Oct-2026).

**Rollback policy:** every changeset has a `rollback` block; changes are forward-only in
production (`06-data/migration-strategy.md`).
