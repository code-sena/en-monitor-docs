# Data Models

---

## Data modeling principles

This project does **not** follow a database-per-service model. It uses **one physical MySQL
instance, organized logically by bounded context** — a deliberate architectural decision (see
`Shared Database` in `01-context/glossary.md`), to be formalized in `ADR-003-shared-database.md`
(`05-architecture/decisions/records/`, pending). Each microservice has read/write access only to
the tables of its own bounded context; no service directly queries or joins another service's
tables. What this avoids is the `Distributed Monolith` anti-pattern (also defined in the
glossary): a service reaching into another service's tables instead of requesting that data
through an API or a domain event.

```
✓ security-service → Security tables (same physical MySQL instance)
✓ alert-service → Alerts tables (same physical instance)
✓ alert-service needs home data → requests it from home-service's API, or already has homeId as
  a stored reference (see below)
✗ alert-service JOINs directly against home-service's homes table
```

**Cross-context foreign keys are intentionally not enforced at the database level.** A column
like `alerts.home_id` exists and is populated, but no `FOREIGN KEY ... REFERENCES` constraint is
declared for it in SQL, even though the referenced table lives in the same physical database.
Two reasons:

1. A real FK constraint would force `alert-service`'s migrations to always run after
   `home-service`'s, and would hard-block ever separating the databases physically later — an
   option this project's architecture deliberately keeps open (see `01-context/glossary.md`).
2. It keeps each service's own migration history self-contained, consistent with the
   `Database migrations repository` decision in `01-context/scope.md` (one repository, with
   per-domain changelogs).

Foreign keys **within** the same bounded context (for example `user.person_id` inside Security)
are declared as real constraints, since both tables are always migrated together by the same
service.

Naming, primary key format, timestamps, soft delete, auditing, units, and enum handling are not
repeated here — they are defined once in `06-data/modeling-conventions.md` and apply to every
table below without exception.


---

## Bounded Context: Security

**Owning microservice:** `security-service` (`02-domain/domain-map.md`)

**DB Engine:** MySQL 8 — physical instance **shared** with every other service (decision to be formalized in `ADR-003-shared-database.md`, pending in `05-architecture/decisions/records/`). `security-service` has read/write access only to the tables listed in this section; no other service accesses them directly.

### Table: `person`

**Purpose:** see the `Person` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE person (
  id_person VARCHAR(10) PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  cellphone VARCHAR(15),
  address VARCHAR(200),
  profile_image VARCHAR(255),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_person` | VARCHAR(10) | PK | — | Person identifier |
| `name` | VARCHAR(100) | NOT NULL | — | First name |
| `last_name` | VARCHAR(100) | NOT NULL | — | Last name |
| `cellphone` | VARCHAR(15) | — | — | Phone number |
| `address` | VARCHAR(200) | — | — | Address |
| `profile_image` | VARCHAR(255) | — | — | Profile image path/URL |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `user`

**Purpose:** see the `User` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE user (
  id_user VARCHAR(10) PRIMARY KEY,
  person_id VARCHAR(10) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  email_verified BOOLEAN NOT NULL,
  registration_date TIMESTAMP NOT NULL,
  status ENUM('ACTIVE', 'INACTIVE', 'BLOCKED') NOT NULL,
  failed_login_attempts INT NOT NULL,
  last_login_at TIMESTAMP,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (person_id) REFERENCES person(id_person)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_user` | VARCHAR(10) | PK | — | User identifier |
| `person_id` | VARCHAR(10) | FK, UNIQUE, NOT NULL | person | Reference to `person` |
| `password_hash` | VARCHAR(255) | NOT NULL | — | Password hash |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | — | Email (used for login) |
| `email_verified` | BOOLEAN | NOT NULL | — | Whether the email was verified |
| `registration_date` | TIMESTAMP | NOT NULL | — | Registration date |
| `status` | ENUM('ACTIVE', 'INACTIVE', 'BLOCKED') | NOT NULL | — | Account status |
| `failed_login_attempts` | INT | NOT NULL | — | Consecutive failed login attempts |
| `last_login_at` | TIMESTAMP | — | — | Last successful login |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `user_configuration`

**Purpose:** see the `UserConfiguration` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE user_configuration (
  id_configuration VARCHAR(10) PRIMARY KEY,
  user_id VARCHAR(10) UNIQUE,
  notify_by_email BOOLEAN NOT NULL,
  notify_by_push BOOLEAN NOT NULL,
  color_theme VARCHAR(50),
  language VARCHAR(50),
  social_provider VARCHAR(50),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES user(id_user)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_configuration` | VARCHAR(10) | PK | — | Configuration identifier |
| `user_id` | VARCHAR(10) | FK, UNIQUE | user | Reference to `user` |
| `notify_by_email` | BOOLEAN | NOT NULL | — | Notify by email |
| `notify_by_push` | BOOLEAN | NOT NULL | — | Notify by push |
| `color_theme` | VARCHAR(50) | — | — | Color theme |
| `language` | VARCHAR(50) | — | — | Preferred language |
| `social_provider` | VARCHAR(50) | — | — | Social provider |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `password_reset_token`

**Purpose:** see the `PasswordResetToken` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE password_reset_token (
  id_reset_token VARCHAR(10) PRIMARY KEY,
  user_id VARCHAR(10) NOT NULL,
  reset_token VARCHAR(100) NOT NULL UNIQUE,
  used BOOLEAN NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  expiration_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES user(id_user)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_reset_token` | VARCHAR(10) | PK | — | Token identifier |
| `user_id` | VARCHAR(10) | FK, NOT NULL | user | Reference to `user` |
| `reset_token` | VARCHAR(100) | UNIQUE, NOT NULL | — | Token identifier |
| `used` | BOOLEAN | NOT NULL | — | Whether the token was already consumed |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `expiration_at` | TIMESTAMP | NOT NULL | — | Expiration date |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `user_session`

**Purpose:** see the `UserSession` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE user_session (
  id_user_session VARCHAR(10) PRIMARY KEY,
  user_id VARCHAR(10) NOT NULL,
  refresh_token VARCHAR(255) NOT NULL UNIQUE,
  revoked BOOLEAN NOT NULL,
  ip_address VARCHAR(45),
  user_agent VARCHAR(150),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  expiration_at TIMESTAMP NOT NULL,
  closed_at TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES user(id_user)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_user_session` | VARCHAR(10) | PK | — | Session identifier |
| `user_id` | VARCHAR(10) | FK, NOT NULL | user | Reference to `user` |
| `refresh_token` | VARCHAR(255) | UNIQUE, NOT NULL | — | Current refresh token |
| `revoked` | BOOLEAN | NOT NULL | — | Whether the session was manually revoked |
| `ip_address` | VARCHAR(45) | — | — | Origin IP (IPv4/IPv6) |
| `user_agent` | VARCHAR(150) | — | — | Client user agent |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `expiration_at` | TIMESTAMP | NOT NULL | — | Expiration date |
| `closed_at` | TIMESTAMP | — | — | Actual close time (logout or revocation) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `security_configuration`

**Purpose:** see the `SecurityConfiguration` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE security_configuration (
  id_security_configuration VARCHAR(10) PRIMARY KEY,
  config_name VARCHAR(100) NOT NULL UNIQUE,
  config_value VARCHAR(100) NOT NULL,
  description VARCHAR(255),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_security_configuration` | VARCHAR(10) | PK | — | Identifier |
| `config_name` | VARCHAR(100) | UNIQUE, NOT NULL | — | Configuration name (key) |
| `config_value` | VARCHAR(100) | NOT NULL | — | Value |
| `description` | VARCHAR(255) | — | — | Explanation of what this configuration value controls |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `password_policy`

**Purpose:** see the `PasswordPolicy` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE password_policy (
  id_password_policy VARCHAR(10) PRIMARY KEY,
  min_length INT NOT NULL,
  max_length INT NOT NULL,
  require_uppercase BOOLEAN NOT NULL,
  require_numbers BOOLEAN NOT NULL,
  require_symbols BOOLEAN NOT NULL,
  expiration_days INT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_password_policy` | VARCHAR(10) | PK | — | Identifier |
| `min_length` | INT | NOT NULL | — | Minimum password length |
| `max_length` | INT | NOT NULL | — | Maximum password length |
| `require_uppercase` | BOOLEAN | NOT NULL | — | Require uppercase letters |
| `require_numbers` | BOOLEAN | NOT NULL | — | Require numbers |
| `require_symbols` | BOOLEAN | NOT NULL | — | Require symbols |
| `expiration_days` | INT | NOT NULL | — | Days before a password change is required |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `login_error_log`

**Purpose:** see the `LoginErrorLog` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE login_error_log (
  id_login_error VARCHAR(10) PRIMARY KEY,
  user_id VARCHAR(10),
  error_type ENUM('INVALID_PASSWORD', 'USER_NOT_FOUND', 'ACCOUNT_BLOCKED', 'ACCOUNT_INACTIVE') NOT NULL,
  description VARCHAR(500),
  ip_address VARCHAR(45),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES user(id_user)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_login_error` | VARCHAR(10) | PK | — | Identifier |
| `user_id` | VARCHAR(10) | FK | user | Reference to `user` |
| `error_type` | ENUM('INVALID_PASSWORD', 'USER_NOT_FOUND', 'ACCOUNT_BLOCKED', 'ACCOUNT_INACTIVE') | NOT NULL | — | Error type |
| `description` | VARCHAR(500) | — | — | Additional detail about the failed login attempt |
| `ip_address` | VARCHAR(45) | — | — | Origin IP (IPv4/IPv6) |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `system_role`

**Purpose:** see the `SystemRole` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE system_role (
  id_system_role VARCHAR(10) PRIMARY KEY,
  name VARCHAR(50) NOT NULL UNIQUE,
  description VARCHAR(200),
  enabled BOOLEAN NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_system_role` | VARCHAR(10) | PK | — | Identifier |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL | — | Role name |
| `description` | VARCHAR(200) | — | — | What this role represents |
| `enabled` | BOOLEAN | NOT NULL | — | Whether the role is active |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `user_system_role`

**Purpose:** see the `UserSystemRole` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE user_system_role (
  user_id VARCHAR(10),
  system_role_id VARCHAR(10),
  assigned_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  PRIMARY KEY (user_id, system_role_id),
  FOREIGN KEY (user_id) REFERENCES user(id_user),
  FOREIGN KEY (system_role_id) REFERENCES system_role(id_system_role)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `user_id` | VARCHAR(10) | PK (composite), FK | user | Reference to `user` |
| `system_role_id` | VARCHAR(10) | PK (composite), FK | system_role | Reference to `system_role` |
| `assigned_at` | TIMESTAMP | NOT NULL | — | Assignment date |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `permission`

**Purpose:** see the `Permission` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE permission (
  id_permission VARCHAR(10) PRIMARY KEY,
  code VARCHAR(30) NOT NULL UNIQUE,
  name VARCHAR(50) NOT NULL,
  description VARCHAR(100),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_permission` | VARCHAR(10) | PK | — | Identifier |
| `code` | VARCHAR(30) | UNIQUE, NOT NULL | — | Short permission code |
| `name` | VARCHAR(50) | NOT NULL | — | Permission name |
| `description` | VARCHAR(100) | — | — | What this permission allows |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `system_role_permission`

**Purpose:** see the `SystemRolePermission` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE system_role_permission (
  system_role_id VARCHAR(10),
  permission_id VARCHAR(10),
  assigned_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  PRIMARY KEY (system_role_id, permission_id),
  FOREIGN KEY (system_role_id) REFERENCES system_role(id_system_role),
  FOREIGN KEY (permission_id) REFERENCES permission(id_permission)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `system_role_id` | VARCHAR(10) | PK (composite), FK | system_role | Reference to `system_role` |
| `permission_id` | VARCHAR(10) | PK (composite), FK | permission | Reference to `permission` |
| `assigned_at` | TIMESTAMP | NOT NULL | — | Assignment date |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `audit_log`

**Purpose:** see the `AuditLog` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE audit_log (
  id_audit_log VARCHAR(10) PRIMARY KEY,
  user_id VARCHAR(10) NOT NULL,
  action ENUM('CREATE', 'UPDATE', 'DELETE', 'LOGIN', 'LOGIN_FAILED', 'LOGOUT') NOT NULL,
  description VARCHAR(255),
  ip_address VARCHAR(45),
  application VARCHAR(255),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES user(id_user)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_audit_log` | VARCHAR(10) | PK | — | Identifier |
| `user_id` | VARCHAR(10) | FK, NOT NULL | user | Reference to `user` |
| `action` | ENUM('CREATE', 'UPDATE', 'DELETE', 'LOGIN', 'LOGIN_FAILED', 'LOGOUT') | NOT NULL | — | Audited action |
| `description` | VARCHAR(255) | — | — | Additional detail about the audited action |
| `ip_address` | VARCHAR(45) | — | — | Origin IP (IPv4/IPv6) |
| `application` | VARCHAR(255) | — | — | Origin application/client |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

---

## Bounded Context: Home Management

**Owning microservice:** `home-service` (`02-domain/domain-map.md`)

**DB Engine:** MySQL 8 — physical instance **shared** with every other service (decision to be formalized in `ADR-003-shared-database.md`, pending in `05-architecture/decisions/records/`). `home-service` has read/write access only to the tables listed in this section; no other service accesses them directly.

### Table: `home_type`

**Purpose:** see the `HomeType` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE home_type (
  id_home_type VARCHAR(10) PRIMARY KEY,
  name VARCHAR(50) NOT NULL UNIQUE,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_home_type` | VARCHAR(10) | PK | — | Identifier |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL | — | Home type name (House, Apartment, Studio, Other) |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `home`

**Purpose:** see the `Home` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE home (
  id_home VARCHAR(10) PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  home_type_id VARCHAR(10) NOT NULL,
  address VARCHAR(200) NOT NULL,
  access_code VARCHAR(8) NOT NULL UNIQUE,
  description VARCHAR(200),
  creation_date TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (home_type_id) REFERENCES home_type(id_home_type)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_home` | VARCHAR(10) | PK | — | Home identifier |
| `name` | VARCHAR(50) | NOT NULL | — | Home name chosen by the owner |
| `home_type_id` | VARCHAR(10) | FK, NOT NULL | home_type | Reference to `home_type` |
| `address` | VARCHAR(200) | NOT NULL | — | Address |
| `access_code` | VARCHAR(8) | UNIQUE, NOT NULL | — | Code other users enter to join the home |
| `description` | VARCHAR(200) | — | — | Optional free-text description of the home |
| `creation_date` | TIMESTAMP | NOT NULL | — | Creation date |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `home_thresholds`

**Purpose:** see the `HomeThresholds` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE home_thresholds (
  id_threshold VARCHAR(10) PRIMARY KEY,
  home_id VARCHAR(10) NOT NULL UNIQUE,
  daily_limit DOUBLE NOT NULL,
  monthly_limit DOUBLE NOT NULL,
  use_system_default BOOLEAN NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (home_id) REFERENCES home(id_home)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_threshold` | VARCHAR(10) | PK | — | Identifier |
| `home_id` | VARCHAR(10) | FK, UNIQUE, NOT NULL | home | Reference to `home` |
| `daily_limit` | DOUBLE | NOT NULL | — | Daily consumption limit (kWh) |
| `monthly_limit` | DOUBLE | NOT NULL | — | Monthly consumption limit (kWh) |
| `use_system_default` | BOOLEAN | NOT NULL | — | Use system default |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `user_home`

**Purpose:** see the `UserHome` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE user_home (
  user_id VARCHAR(10),
  home_id VARCHAR(10),
  role ENUM('OWNER', 'MEMBER') NOT NULL,
  favorite BOOLEAN NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  PRIMARY KEY (user_id, home_id),
  FOREIGN KEY (home_id) REFERENCES home(id_home)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `user_id` | VARCHAR(10) | PK (composite), FK (cross-context, app-level only) | user | Cross-context reference to `user` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `home_id` | VARCHAR(10) | PK (composite), FK | home | Reference to `home` |
| `role` | ENUM('OWNER', 'MEMBER') | NOT NULL | — | User's role within the home |
| `favorite` | BOOLEAN | NOT NULL | — | Whether the user marked this home as a favorite |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

---

## Bounded Context: Devices

**Owning microservice:** `device-service` (`02-domain/domain-map.md`)

**DB Engine:** MySQL 8 — physical instance **shared** with every other service (decision to be formalized in `ADR-003-shared-database.md`, pending in `05-architecture/decisions/records/`). `device-service` has read/write access only to the tables listed in this section; no other service accesses them directly.

### Table: `appliance_type`

**Purpose:** see the `ApplianceType` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE appliance_type (
  id_appliance_type VARCHAR(10) PRIMARY KEY,
  name VARCHAR(50) NOT NULL UNIQUE,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_appliance_type` | VARCHAR(10) | PK | — | Identifier |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL | — | Appliance type name |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `device`

**Purpose:** see the `Device` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE device (
  id_device VARCHAR(10) PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  appliance_type_id VARCHAR(10) NOT NULL,
  location VARCHAR(50),
  description VARCHAR(200),
  installation_date TIMESTAMP NOT NULL,
  device_code VARCHAR(6) NOT NULL UNIQUE,
  api_key VARCHAR(100) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (appliance_type_id) REFERENCES appliance_type(id_appliance_type)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_device` | VARCHAR(10) | PK | — | Device identifier |
| `name` | VARCHAR(50) | NOT NULL | — | Device name chosen by the owner |
| `appliance_type_id` | VARCHAR(10) | FK, NOT NULL | appliance_type | Reference to `appliance_type` |
| `location` | VARCHAR(50) | — | — | Location within the home |
| `description` | VARCHAR(200) | — | — | Optional free-text description of the device |
| `installation_date` | TIMESTAMP | NOT NULL | — | Installation date |
| `device_code` | VARCHAR(6) | UNIQUE, NOT NULL | — | Short code used to pair the physical device |
| `api_key` | VARCHAR(100) | NOT NULL | — | Device credential used to publish over MQTT |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `device_status_log`

**Purpose:** see the `DeviceStatusLog` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE device_status_log (
  id_device_status VARCHAR(10) PRIMARY KEY,
  device_id VARCHAR(10),
  status ENUM('ONLINE', 'OFFLINE') NOT NULL,
  signal_strength INT,
  last_seen TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (device_id) REFERENCES device(id_device)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_device_status` | VARCHAR(10) | PK | — | Identifier |
| `device_id` | VARCHAR(10) | FK | device | Reference to `device` |
| `status` | ENUM('ONLINE', 'OFFLINE') | NOT NULL | — | Connectivity status recorded at this point in time |
| `signal_strength` | INT | — | — | Signal strength, when the hardware reports it |
| `last_seen` | TIMESTAMP | NOT NULL | — | Last time seen |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `device_home`

**Purpose:** see the `DeviceHome` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE device_home (
  device_id VARCHAR(10),
  home_id VARCHAR(10),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  PRIMARY KEY (device_id, home_id),
  FOREIGN KEY (device_id) REFERENCES device(id_device)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `device_id` | VARCHAR(10) | PK (composite), FK | device | Reference to `device` |
| `home_id` | VARCHAR(10) | PK (composite), FK (cross-context, app-level only) | home | Cross-context reference to `home` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

---

## Bounded Context: Monitoring and Measurements

**Owning microservice:** `measurement-service` (`02-domain/domain-map.md`)

**DB Engine:** MySQL 8 — physical instance **shared** with every other service (decision to be formalized in `ADR-003-shared-database.md`, pending in `05-architecture/decisions/records/`). `measurement-service` has read/write access only to the tables listed in this section; no other service accesses them directly.

### Table: `measurement`

**Purpose:** see the `Measurement` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE measurement (
  id_measurement VARCHAR(10) PRIMARY KEY,
  device_id VARCHAR(10) NOT NULL,
  date_time TIMESTAMP NOT NULL,
  voltage DOUBLE NOT NULL,
  current DOUBLE NOT NULL,
  active_power DOUBLE NOT NULL,
  stored_energy DOUBLE NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_measurement` | VARCHAR(10) | PK | — | Measurement identifier |
| `device_id` | VARCHAR(10) | FK (cross-context, app-level only), NOT NULL | device | Cross-context reference to `device` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `date_time` | TIMESTAMP | NOT NULL | — | Timestamp of the electrical reading |
| `voltage` | DOUBLE | NOT NULL | — | Voltage (V) |
| `current` | DOUBLE | NOT NULL | — | Current (A) |
| `active_power` | DOUBLE | NOT NULL | — | Active power |
| `stored_energy` | DOUBLE | NOT NULL | — | Stored energy |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

---

## Bounded Context: Alerts

**Owning microservice:** `alert-service` (`02-domain/domain-map.md`)

**DB Engine:** MySQL 8 — physical instance **shared** with every other service (decision to be formalized in `ADR-003-shared-database.md`, pending in `05-architecture/decisions/records/`). `alert-service` has read/write access only to the tables listed in this section; no other service accesses them directly.

### Table: `consumption_level`

**Purpose:** see the `ConsumptionLevel` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE consumption_level (
  id_consumption_level VARCHAR(10) PRIMARY KEY,
  name ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL') NOT NULL UNIQUE,
  description VARCHAR(200) NOT NULL,
  min_limit DOUBLE NOT NULL,
  max_limit DOUBLE NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_consumption_level` | VARCHAR(10) | PK | — | Identifier |
| `name` | ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL') | UNIQUE, NOT NULL | — | Consumption level name (LOW, MEDIUM, HIGH, CRITICAL) |
| `description` | VARCHAR(200) | NOT NULL | — | Human-readable explanation of this consumption level |
| `min_limit` | DOUBLE | NOT NULL | — | Lower bound of the range |
| `max_limit` | DOUBLE | NOT NULL | — | Upper bound of the range |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

### Table: `alert`

**Purpose:** see the `Alert` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE alert (
  id_alert VARCHAR(10) PRIMARY KEY,
  home_id VARCHAR(10) NOT NULL,
  device_id VARCHAR(10),
  type ENUM('THRESHOLD', 'CONNECTIVITY') NOT NULL,
  message_key VARCHAR(100) NOT NULL,
  date_time TIMESTAMP NOT NULL,
  alert_status ENUM('PENDING', 'RESOLVED') NOT NULL,
  consumption_level_id VARCHAR(10),
  measurement_id VARCHAR(10),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  FOREIGN KEY (consumption_level_id) REFERENCES consumption_level(id_consumption_level)
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_alert` | VARCHAR(10) | PK | — | Alert identifier |
| `home_id` | VARCHAR(10) | FK (cross-context, app-level only), NOT NULL | home | Cross-context reference to `home` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `device_id` | VARCHAR(10) | FK (cross-context, app-level only) | device | Cross-context reference to `device` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `type` | ENUM('THRESHOLD', 'CONNECTIVITY') | NOT NULL | — | Alert type |
| `message_key` | VARCHAR(100) | NOT NULL | — | Message key (i18n, not plain text) |
| `date_time` | TIMESTAMP | NOT NULL | — | Timestamp the alert was raised |
| `alert_status` | ENUM('PENDING', 'RESOLVED') | NOT NULL | — | Alert lifecycle status |
| `consumption_level_id` | VARCHAR(10) | FK | consumption_level | Reference to `consumption_level` |
| `measurement_id` | VARCHAR(10) | FK (cross-context, app-level only) | measurement | Cross-context reference to `measurement` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

---

## Bounded Context: Recommendations

**Owning microservice:** `recommendation-service` (`02-domain/domain-map.md`)

**DB Engine:** MySQL 8 — physical instance **shared** with every other service (decision to be formalized in `ADR-003-shared-database.md`, pending in `05-architecture/decisions/records/`). `recommendation-service` has read/write access only to the tables listed in this section; no other service accesses them directly.

### Table: `recommendation`

**Purpose:** see the `Recommendation` entity description in `02-domain/entities-and-rules.md`.

```sql
CREATE TABLE recommendation (
  id_recommendation VARCHAR(10) PRIMARY KEY,
  home_id VARCHAR(10) NOT NULL,
  device_id VARCHAR(10),
  message_key VARCHAR(100) NOT NULL,
  date_time TIMESTAMP NOT NULL,
  status ENUM('READ', 'UNREAD') NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Data dictionary:**

| Column | Type | Constraints | References | Description |
|--------|------|-------------|------------|-------------|
| `id_recommendation` | VARCHAR(10) | PK | — | Identifier |
| `home_id` | VARCHAR(10) | FK (cross-context, app-level only), NOT NULL | home | Cross-context reference to `home` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `device_id` | VARCHAR(10) | FK (cross-context, app-level only) | device | Cross-context reference to `device` (see `02-domain/domain-map.md`) — not enforced as a DB constraint |
| `message_key` | VARCHAR(100) | NOT NULL | — | Message key (i18n, not plain text) |
| `date_time` | TIMESTAMP | NOT NULL | — | Timestamp the recommendation was generated |
| `status` | ENUM('READ', 'UNREAD') | NOT NULL | — | Whether the user has read the recommendation |
| `created_at` | TIMESTAMP | NOT NULL | — | Row creation timestamp (audit) |
| `updated_at` | TIMESTAMP | NOT NULL | — | Last update timestamp (audit) |
| `deleted_at` | TIMESTAMP | — | — | Soft-delete marker — NULL means active, a value means deleted |

---

## Migration strategy

Migration tooling, rules, and rollback policy are not repeated here — see
`06-data/migration-strategy.md`.

---

## Correlations

- Domain entities that map to these tables → `02-domain/entities-and-rules.md`
- Bounded contexts and cross-context relationships → `02-domain/domain-map.md`
- Domain events that also touch this data → `02-domain/domain-events.md`
- Naming, ID format, and other conventions → `06-data/modeling-conventions.md`
- How schema changes are rolled out → `06-data/migration-strategy.md`
- Shared-database decision, once formalized → `05-architecture/decisions/records/ADR-003-shared-database.md`
- Service ownership per table → `09-microservices/service-catalog.md`