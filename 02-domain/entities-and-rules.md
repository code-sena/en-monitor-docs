# Entities, Value Objects, and Business Rules
---

## 1. Tactical DDD concepts

### Entity
An **Entity** is defined by its identity, not by its attributes. Two entities with the same ID represent the same domain object, even if their other data changes.

```
✓ Entity: User (the same user remains the same even if the email, status, or login attempts change)
✓ Entity: Home (a home keeps its identity even when its name, thresholds, or members change)
✓ Entity: Device (the same physical device remains the same even if its connectivity status or assigned home changes)
✗ Not an entity: AccessCode (it is compared by its value, not by its identity, so it is a Value Object)
```

### Value Object (VO)
A **Value Object** is defined by its attributes and has no identity of its own. It is immutable: if one value changes, a new VO is created.

```
✓ Value Object: Address (5th Street #10-20, Neiva, Huila)
✓ Value Object: Email (user@example.com)
✓ Value Object: AccessCode (A1B2C3D4)
✓ Value Object: DeviceCode (DEV-2048)
✓ Value Object: ApiKey (device_api_key_8f4a2c1e)
✓ Value Object: MessageKey (ALERT_HIGH_CONSUMPTION)
```

### Aggregate
An **Aggregate** groups entities and VOs that must remain consistent as one unit. It has an **Aggregate Root**, which is the main entry point.

```
Home (Aggregate Root)
  ├── Device[] (Entities inside the aggregate)
  ├── HomeThresholds (Value Object)
  ├── UserHome[] (Entities that link users to the home)
  └── AccessCode (Value Object for joining the home)
```
**Golden rule of the Aggregate:** Transactions do not cross aggregate boundaries. If you need to modify two aggregates in one operation, use a Domain Event and a Saga.

### Business Rules
**Business Rules** (invariants) are constraints that must always hold. These rules live in the Aggregate Root and are validated on every domain operation.

---

## 2. System entities

### Entity: Person

**Context:** Security

**Description:** Represents the personal identity of a human being. It does not authenticate and does not store credentials.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_person` | `VARCHAR(10)` | Unique person identifier | Yes | Primary key |
| `name` | `VARCHAR(100)` | First name | Yes | Not null |
| `last_name` | `VARCHAR(100)` | Last name | Yes | Not null |
| `cellphone` | `VARCHAR(15)` | Phone number | No | Optional |
| `address` | `VARCHAR(200)` | Address | No | Optional |
| `profile_image` | `VARCHAR(255)` | Image URL or path | No | Optional |

**Lifecycle / States:**
```
[Registered] ──(profile updated)──▶ [Profile Updated]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `Registered` | Personal data has been captured | → `Profile Updated` |
| `Profile Updated` | One or more personal fields were changed | → `Profile Updated` |

**Invariants:**

```
INV-001: A person must have a first name and last name.
INV-002: A person must not store credentials or tokens. Authentication state belongs to `User`, not to `Person`.
```

---

### Entity: User

**Context:** Security

**Description:** Represents the system account associated with a `Person`. It owns credentials, account status, and login history.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_user` | `VARCHAR(10)` | Unique user identifier | Yes | Primary key |
| `id_person` | `VARCHAR(10)` | Link to `Person` | Yes | FK, unique, not null |
| `password_hash` | `VARCHAR(255)` | Password hash | Yes | Not null |
| `email` | `VARCHAR(255)` | Main email address | Yes | Unique, not null |
| `email_verified` | `BOOLEAN` | Whether the email has been verified | Yes | Not null |
| `date_of_registration` | `DATETIME` | Registration date | Yes | Not null |
| `status` | `UserStatus` | Account state | Yes | Not null |
| `failed_login_attempts` | `INT` | Consecutive failed login attempts | Yes | Not null |
| `last_login_at` | `DATETIME` | Last successful login | No | Optional |

**Lifecycle / States:**

```
ACTIVE ──(login failure)──▶ INACTIVE
ACTIVE ──(lockout policy)──▶ BLOCKED
INACTIVE ──(recovery)──▶ ACTIVE
BLOCKED ──(admin action)──▶ ACTIVE
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Account enabled for authentication | → `INACTIVE`, → `BLOCKED` |
| `INACTIVE` | Account temporarily unusable | → `ACTIVE` |
| `BLOCKED` | Account blocked for security reasons | → `ACTIVE` |

**Invariants:**

```
INV-003: The email must be unique across the system.
INV-004: Passwords are never stored in plain text.
INV-005: `BLOCKED` state must prevent new authentication attempts.
INV-006: A user may have multiple active sessions, but only one primary identity.
```

**Code example (Java):**

```java
// Domain model — framework-free, no Spring or JPA annotations
public class User {

    private final UserId id;
    private final PersonId personId;
    private String passwordHash;
    private final Email email;
    private UserStatus status;
    private int failedLoginAttempts;

    private User(UserId id, PersonId personId, String passwordHash, Email email) {
        this.id = id;
        this.personId = personId;
        this.passwordHash = passwordHash;
        this.email = email;
        this.status = UserStatus.ACTIVE;
        this.failedLoginAttempts = 0;
    }

    public static User register(PersonId personId, String passwordHash, Email email) {
        // INV-004: passwords are never stored in plain text — the hash arrives already hashed
        return new User(UserId.newId(), personId, passwordHash, email);
    }

    public void registerFailedLogin(PasswordPolicy policy) {
        // INV-005: a BLOCKED account must not authenticate again
        if (this.status == UserStatus.BLOCKED) {
            throw new AccountBlockedException();
        }
        this.failedLoginAttempts++;
        if (this.failedLoginAttempts >= policy.maxFailedAttempts()) {
            this.status = UserStatus.BLOCKED;
        }
    }
}
```

---

### Entity: UserConfiguration

**Context:** Security

**Description:** Stores the user's personal preferences such as notifications, theme, language, and social provider.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_configuration` | `VARCHAR(10)` | Unique identifier | Yes | Primary key |
| `id_user` | `VARCHAR(10)` | User that owns this configuration | Yes | FK, unique |
| `notify_by_email` | `BOOLEAN` | Receive notifications by email | Yes | Not null |
| `notify_by_push` | `BOOLEAN` | Receive push notifications | Yes | Not null |
| `color_theme` | `VARCHAR(50)` | Visual theme | No | Optional |
| `language` | `VARCHAR(50)` | Preferred language | No | Optional |
| `social_provider` | `VARCHAR(50)` | Social login provider | No | Optional |


**Lifecycle / States:**

```
DEFAULT ──(preferences edited)──▶ UPDATED ──(customization saved)──▶ CUSTOMIZED
    ▲                                                                    │
    └──────────────────(restore default preferences)───────────────────┘
CUSTOMIZED ──(preferences changed again)──▶ UPDATED
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `DEFAULT` | Configuration is created with default preferences | → `UPDATED`, → `CUSTOMIZED` |
| `UPDATED` | User has changed one or more preferences | → `CUSTOMIZED`, → `DEFAULT` |
| `CUSTOMIZED` | User has explicitly personalized the configuration | → `UPDATED` |

**Invariants:**

```
INV-007: Each user can have at most one personal configuration.
INV-008: Notification settings must always be explicit and never null.
```

---

### Entity: PasswordResetToken

**Context:** Security

**Description:** Token used for password recovery, with one-time use and an expiration date.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_reset_token` | `VARCHAR(10)` | Token identifier | Yes | Primary key |
| `id_user` | `VARCHAR(10)` | User associated with the token | Yes | FK, not null |
| `reset_token` | `VARCHAR(100)` | Random recovery token | Yes | Unique, not null |
| `used` | `BOOLEAN` | Whether the token has already been used | Yes | Not null |
| `created_at` | `DATETIME` | Creation timestamp | Yes | Not null |
| `expiration_at` | `DATETIME` | Expiration timestamp | Yes | Not null |

**Lifecycle / States:**

```
[DRAFT] ──(token generated)──▶ [ACTIVE] ──(token used)──▶ [CANCELLED]
                               │
                        (token expired)
                               ▼
                           [CANCELLED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `DRAFT` | Token has been created but not yet published for use | → `ACTIVE`, → `CANCELLED` |
| `ACTIVE` | Token is available for password recovery | → `CANCELLED` |
| `CANCELLED` | Token has been invalidated or expired without completing recovery | Terminal state |

**Invariants:**

```
INV-009: An expired token cannot be used to change the password.
INV-010: A used token cannot be reused.
```

---

### Entity: UserSession

**Context:** Security

**Description:** Represents a persisted authenticated session for a user. It is the revocable entity that supports refresh tokens and expiration.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_user_session` | `VARCHAR(10)` | Session identifier | Yes | Primary key |
| `id_user` | `VARCHAR(10)` | User that owns the session | Yes | FK, not null |
| `refresh_token` | `VARCHAR(255)` | Persisted token used for renewal | Yes | Unique, not null |
| `revoked` | `BOOLEAN` | Whether the session was revoked | Yes | Not null |
| `ip_address` | `VARCHAR(45)` | Client IP address | No | Optional |
| `user_agent` | `VARCHAR(150)` | Browser or client agent | No | Optional |
| `created_at` | `DATETIME` | Creation date | Yes | Not null |
| `expiration_at` | `DATETIME` | Expiration date | Yes | Not null |
| `closed_at` | `DATETIME` | Closure date | No | Optional |

**Lifecycle / States:**

```
[ACTIVE] ──(logout)──▶ [CLOSED]
[ACTIVE] ──(revoke by system)──▶ [REVOKED]
[ACTIVE] ──(token expired)──▶ [EXPIRED]
[EXPIRED] ──(recovery attempt denied)──▶ [CLOSED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Valid authenticated session currently in use | → `CLOSED`, → `REVOKED`, → `EXPIRED` |
| `EXPIRED` | Session has reached its expiration time | → `CLOSED` |
| `REVOKED` | Session was invalidated by the system or by the user | Terminal state |
| `CLOSED` | Session is ended and no longer valid | Terminal state |

**Invariants:**

```
INV-011: A revoked session must not be considered valid.
INV-012: The refresh token must be unique per session.
```

---

### Entity: SecurityConfiguration

**Context:** Security

**Description:** Global security configuration that applies to the system as a whole rather than to an individual user.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_security_configuration` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `config_name` | `VARCHAR(100)` | Configuration name | Yes | Unique, not null |
| `config_value` | `VARCHAR(100)` | Configuration value | Yes | Not null |
| `description` | `VARCHAR(255)` | Description | No | Optional |

**Lifecycle / States:**

```
[DRAFT] ──(approved by admin)──▶ [ACTIVE] ──(configuration updated)──▶ [UPDATED]
                                              │
                                  (deactivate or revert to defaults)
                                              ▼
                                          [DISABLED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `DRAFT` | Configuration is being defined before publication | → `ACTIVE` |
| `ACTIVE` | Configuration is currently enforced in the system | → `UPDATED`, → `DISABLED` |
| `UPDATED` | Configuration has been modified and requires revalidation | → `ACTIVE`, → `DISABLED` |
| `DISABLED` | Configuration is temporarily disabled by policy | Terminal state |

**Invariants:**

```
INV-013: Global security configuration values must always align with the security policy.
INV-014: Security configuration belongs to the system, not to a specific user.
```

---

### Entity: PasswordPolicy

**Context:** Security

**Description:** Defines the minimum rules a password must meet before being accepted by the system.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_password_policy` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `min_length` | `INT` | Minimum length | Yes | Not null |
| `max_length` | `INT` | Maximum length | Yes | Not null |
| `require_uppercase` | `BOOLEAN` | Require uppercase letters | Yes | Not null |
| `require_numbers` | `BOOLEAN` | Require numbers | Yes | Not null |
| `require_symbols` | `BOOLEAN` | Require symbols | Yes | Not null |
| `expiration_days` | `INT` | Password validity period | Yes | Not null |

**Lifecycle / States:**

```
[DRAFT] ──(policy approved)──▶ [ACTIVE] ──(policy revised)──▶ [UPDATED]
                               │
                  (policy retired)
                               ▼
                           [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `DRAFT` | New password policy being prepared | → `ACTIVE` |
| `ACTIVE` | Policy currently enforced in the system | → `UPDATED`, → `ARCHIVED` |
| `UPDATED` | Revised version awaiting validation | → `ACTIVE`, → `ARCHIVED` |
| `ARCHIVED` | Policy is no longer active | Terminal state |

**Invariants:**

```
INV-015: An invalid password cannot be persisted as a valid hash.
INV-016: Password policies apply both at registration time and at password reset time.
```

---

### Entity: LoginErrorLog

**Context:** Security

**Description:** Records authentication errors for auditing and failed-login analysis.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_login_error` | `VARCHAR(10)` | Log identifier | Yes | Primary key |
| `id_user` | `VARCHAR(10)` | Affected user, if any | No | FK |
| `error_type` | `LoginErrorType` | Error type | Yes | Not null |
| `description` | `VARCHAR(500)` | Event description | No | Optional |
| `ip_address` | `VARCHAR(45)` | Origin IP address | No | Optional |
| `created_at` | `DATETIME` | Event timestamp | Yes | Not null |

**Lifecycle / States:**

```
[RECORDED] ──(error retained for audit)──▶ [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `RECORDED` | Authentication error has been captured and stored | → `ARCHIVED` |
| `ARCHIVED` | Error log is retained for security analysis and audit | Terminal state |

**Invariants:**

```
INV-017: Login error records are immutable once created.
INV-018: The error classification must always be a valid `LoginErrorType` value.
```

---

### Entity: SystemRole

**Context:** Security

**Description:** Groups global permissions for the platform. It is not the same as `Home Role`.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_system_role` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `name` | `VARCHAR(50)` | Role name | Yes | Unique, not null |
| `description` | `VARCHAR(200)` | Description | No | Optional |
| `enabled` | `BOOLEAN` | Whether the role is active | Yes | Not null |
| `created_at` | `DATETIME` | Creation timestamp | Yes | Not null |

**Lifecycle / States:**

```
[ACTIVE] ──(role disabled)──▶ [DISABLED]
[DISABLED] ──(role re-enabled)──▶ [ACTIVE]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Role is available for assignment | → `DISABLED` |
| `DISABLED` | Role cannot be assigned to new users | → `ACTIVE` |

**Invariants:**

```
INV-019: A disabled role cannot be assigned to new users.
INV-020: The role name must be unique across the platform.
```

---

### Entity: UserSystemRole

**Context:** Security

**Description:** Many-to-many relationship between users and system roles.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_user` | `VARCHAR(10)` | User | Yes | PK/FK |
| `id_system_role` | `VARCHAR(10)` | System role | Yes | PK/FK |
| `assigned_at` | `DATETIME` | Assignment timestamp | Yes | Not null |

**Lifecycle / States:**

```
[ASSIGNED] ──(role revoked)──▶ [REVOKED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ASSIGNED` | Role is currently assigned to the user | → `REVOKED` |
| `REVOKED` | Assignment has been removed | Terminal state |

**Invariants:**

```
INV-021: A user can have multiple roles, but each assignment is unique.
```

---

### Entity: Permission

**Context:** Security

**Description:** An individual authorizable action represented by a stable code.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_permission` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `code` | `VARCHAR(30)` | Stable permission code | Yes | Unique, not null |
| `name` | `VARCHAR(50)` | Display name | Yes | Not null |
| `description` | `VARCHAR(100)` | Description | No | Optional |

**Lifecycle / States:**

```
[ACTIVE] ──(deprecated by policy)──▶ [DEPRECATED]
[DEPRECATED] ──(retired from platform)──▶ [RETIRED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Permission is currently valid for authorization checks | → `DEPRECATED` |
| `DEPRECATED` | Permission remains defined but is no longer recommended | → `RETIRED` |
| `RETIRED` | Permission is removed from active authorization policies | Terminal state |

**Invariants:**

```
INV-022: Permission codes must be stable and reusable for authorization checks.
```

---

### Entity: SystemRolePermission

**Context:** Security

**Description:** Relationship between a global role and the permissions it grants.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_system_role` | `VARCHAR(10)` | System role | Yes | PK/FK |
| `id_permission` | `VARCHAR(10)` | Assigned permission | Yes | PK/FK |
| `assigned_at` | `DATETIME` | Assignment timestamp | Yes | Not null |

**Lifecycle / States:**

```
[ASSIGNED] ──(permission removed)──▶ [REMOVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ASSIGNED` | Permission is currently linked to the role | → `REMOVED` |
| `REMOVED` | Link has been revoked and is no longer active | Terminal state |

---

### Entity: AuditLog

**Context:** Security

**Description:** Immutable log of relevant system events for auditing purposes.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_audit_log` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `id_user` | `VARCHAR(10)` | Responsible user | Yes | FK, not null |
| `action` | `AuditAction` | Action performed | Yes | Not null |
| `description` | `VARCHAR(255)` | Event description | No | Optional |
| `ip_address` | `VARCHAR(45)` | Origin IP | No | Optional |
| `application` | `VARCHAR(255)` | Application that emitted the event | No | Optional |
| `created_at` | `DATETIME` | Event timestamp | Yes | Not null |

**Lifecycle / States:**

```
[RECORDED] ──(event archived for traceability)──▶ [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `RECORDED` | Audit event has been created and persisted | → `ARCHIVED` |
| `ARCHIVED` | Audit event is preserved for compliance and historical review | Terminal state |

**Invariants:**

```
INV-023: Audit records are never deleted or updated after creation.
```

---

### Entity: HomeType

**Context:** Home Management

**Description:** Catalog of home types such as house, apartment, studio, etc.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_home_type` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `name` | `VARCHAR(50)` | Home type name | Yes | Unique, not null |

**Lifecycle / States:**

```
[ACTIVE] ──(catalog entry deprecated)──▶ [DEPRECATED]
[DEPRECATED] ──(retired from catalog)──▶ [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Home type is available for selection | → `DEPRECATED` |
| `DEPRECATED` | Home type remains in the catalog but is no longer recommended | → `ARCHIVED` |
| `ARCHIVED` | Home type is no longer available for use | Terminal state |

---

### Entity: Home

**Context:** Home Management

**Description:** Aggregate root of the home context. It groups people, devices, thresholds, alerts, and recommendations for a dwelling.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_home` | `VARCHAR(10)` | Unique home identifier | Yes | Primary key |
| `name` | `VARCHAR(50)` | Home name | Yes | Not null |
| `id_home_type` | `VARCHAR(10)` | Home type | Yes | FK, not null |
| `address` | `VARCHAR(200)` | Home address | Yes | Not null |
| `access_code` | `VARCHAR(8)` | Code to join the home | Yes | Unique, not null |
| `description` | `VARCHAR(200)` | Description | No | Optional |
| `creation_date` | `DATETIME` | Creation date | Yes | Not null |

**Lifecycle / States:**

```
[ACTIVE] ──(home settings edited)──▶ [UPDATED] ──(changes saved)──▶ [ACTIVE]
                         │
               (home archived or deactivated)
                         ▼
                       [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Home is currently operating and visible to its members | → `UPDATED`, → `ARCHIVED` |
| `UPDATED` | Home data has been modified and the new version is being validated | → `ACTIVE` |
| `ARCHIVED` | Home is no longer active in the system | Terminal state |

**Invariants:**

```
INV-024: A home must have a valid home type.
INV-025: `access_code` must be unique across homes.
INV-026: A home can have multiple members and multiple devices, but remains one aggregate.
```

---

### Entity: HomeThresholds

**Context:** Home Management

**Description:** Stores the configured consumption limits for a home.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_threshold` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `id_home` | `VARCHAR(10)` | Related home | Yes | FK, unique, not null |
| `daily_limit` | `DOUBLE` | Daily consumption limit | Yes | Not null |
| `monthly_limit` | `DOUBLE` | Monthly consumption limit | Yes | Not null |
| `use_system_defaults` | `BOOLEAN` | Whether the home uses system defaults | Yes | Not null |

**Lifecycle / States:**

```
[DEFAULT] ──(custom thresholds created)──▶ [CUSTOMIZED]
[CUSTOMIZED] ──(restore system defaults)──▶ [DEFAULT]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `DEFAULT` | Home is using system-wide default thresholds | → `CUSTOMIZED` |
| `CUSTOMIZED` | Home has personalized threshold values | → `DEFAULT` |

**Invariants:**

```
INV-027: A home has at most one threshold configuration.
INV-028: If `use_system_defaults = true`, the home must use the system defaults.
```

---

### Entity: UserHome

**Context:** Home Management

**Description:** Relationship between a user and a home, including the role that user holds within that home.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_user` | `VARCHAR(10)` | User | Yes | PK/FK |
| `id_home` | `VARCHAR(10)` | Home | Yes | PK/FK |
| `role` | `HomeRole` | Role inside the home | Yes | Not null |
| `favorite` | `BOOLEAN` | Whether the home is marked as favorite | Yes | Not null |

**Lifecycle / States:**

```
[ACTIVE] ──(role or favorite updated)──▶ [UPDATED]
[UPDATED] ──(changes saved)──▶ [ACTIVE]
[ACTIVE] ──(user removed from home)──▶ [REMOVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | User is currently associated with the home | → `UPDATED`, → `REMOVED` |
| `UPDATED` | Membership metadata has changed and needs validation | → `ACTIVE` |
| `REMOVED` | User no longer belongs to the home | Terminal state |

**Invariants:**

```
INV-029: The home role is independent from the global system role.
INV-030: A user can belong to multiple homes, each with a different role.
```

---

### Entity: ApplianceType

**Context:** Devices

**Description:** Classifies the type of appliance being measured, such as refrigerator, air conditioner, lighting, etc.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_appliance_type` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `name` | `VARCHAR(50)` | Appliance type name | Yes | Unique, not null |

**Lifecycle / States:**

```
[ACTIVE] ──(type deprecated)──▶ [DEPRECATED]
[DEPRECATED] ──(type retired)──▶ [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Appliance type is available for assignment | → `DEPRECATED` |
| `DEPRECATED` | Appliance type remains defined but is no longer recommended | → `ARCHIVED` |
| `ARCHIVED` | Appliance type is no longer available for use | Terminal state |

---

### Entity: Device

**Context:** Devices

**Description:** Physical measurement device. This is the equipment that installs on a load to measure its electrical consumption.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_device` | `VARCHAR(10)` | Unique device identifier | Yes | Primary key |
| `name` | `VARCHAR(50)` | Device name | Yes | Not null |
| `id_appliance_type` | `VARCHAR(10)` | Type of appliance measured | Yes | FK, not null |
| `location` | `VARCHAR(50)` | Device location | No | Optional |
| `description` | `VARCHAR(200)` | Description | No | Optional |
| `installation_date` | `DATETIME` | Installation date | Yes | Not null |
| `device_code` | `VARCHAR(6)` | Short device code | Yes | Unique, not null |
| `api_key` | `VARCHAR(100)` | Device authentication key | Yes | Not null |

**Lifecycle / States:**

```
[REGISTERED] ──(device connected)──▶ [ACTIVE] ──(device stops reporting)──▶ [INACTIVE]
                         │
                 (device rejected)
                         ▼
                      [REJECTED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `REGISTERED` | Device has been created and accepted by the system | → `ACTIVE`, → `REJECTED` |
| `ACTIVE` | Device is connected and publishing measurements | → `INACTIVE`, → `REJECTED` |
| `INACTIVE` | Device is temporarily unavailable or not reporting | → `ACTIVE`, → `REJECTED` |
| `REJECTED` | Device was invalidated or blocked | Terminal state |

**Invariants:**

```
INV-031: A device must be associated with a valid appliance type.
INV-032: Each device has its own API key and does not reuse user credentials.
INV-033: If a device is moved to another home, a new `DeviceHome` relationship is recorded.
```

---

### Entity: DeviceStatusLog

**Context:** Devices

**Description:** History of the device's connection state and the last known signal quality.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_device_status` | `VARCHAR(10)` | Log identifier | Yes | Primary key |
| `id_device` | `VARCHAR(10)` | Related device | Yes | FK |
| `status` | `DeviceStatus` | Device state | Yes | Not null |
| `signal_strength` | `INT` | Signal quality | No | Optional |
| `last_seen` | `DATETIME` | Last time it responded | Yes | Not null |

**Lifecycle / States:**

```
[RECORDED] ──(status retained for history)──▶ [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `RECORDED` | Status change has been captured | → `ARCHIVED` |
| `ARCHIVED` | Status log retained for diagnostics | Terminal state |

**Invariants:**

```
INV-034: Device status is derived from actual device behavior, not from user intent.
```

---

### Entity: DeviceHome

**Context:** Devices

**Description:** Many-to-many relationship between devices and homes.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_device` | `VARCHAR(10)` | Device | Yes | PK/FK |
| `id_home` | `VARCHAR(10)` | Home | Yes | PK/FK |

**Lifecycle / States:**

```
[ASSIGNED] ──(device moved to another home)──▶ [REASSIGNED]
[REASSIGNED] ──(device removed from home)──▶ [REMOVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ASSIGNED` | Device is currently linked to the home | → `REASSIGNED`, → `REMOVED` |
| `REASSIGNED` | Device was reassigned to a different home | → `REMOVED` |
| `REMOVED` | Device-home relationship has been removed | Terminal state |

---

### Entity: Measurement

**Context:** Monitoring and Measurements

**Description:** A single electrical reading produced by a device at a given point in time.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_measurement` | `VARCHAR(10)` | Measurement identifier | Yes | Primary key |
| `id_device` | `VARCHAR(10)` | Producing device | Yes | FK, not null |
| `date_time` | `DATETIME` | Measurement timestamp | Yes | Not null |
| `voltage` | `DOUBLE` | Voltage | Yes | Not null |
| `current` | `DOUBLE` | Current | Yes | Not null |
| `active_power` | `DOUBLE` | Active power | Yes | Not null |
| `stored_energy` | `DOUBLE` | Accumulated energy | Yes | Not null |

**Lifecycle / States:**

```
[CREATED] ──(stored in the repository)──▶ [STORED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `CREATED` | Measurement has been produced but not yet persisted | → `STORED` |
| `STORED` | Measurement has been persisted and is immutable | Terminal state |

**Invariants:**

```
INV-035: A measurement is immutable once stored.
INV-036: A measurement must come from a valid, authenticated device.
```

---

### Entity: ConsumptionLevel

**Context:** Monitoring and Measurements

**Description:** Numerical classification of a consumption value. Used to categorize the risk associated with that value.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_consumption_level` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `name` | `RiskConsumption` | Level name | Yes | Unique, not null |
| `description` | `VARCHAR(200)` | Description of the range | Yes | Not null |
| `min_limit` | `DOUBLE` | Lower bound | Yes | Not null |
| `max_limit` | `DOUBLE` | Upper bound | Yes | Not null |

**Lifecycle / States:**

```
[ACTIVE] ──(range revised)──▶ [UPDATED]
[UPDATED] ──(new range published)──▶ [ACTIVE]
[ACTIVE] ──(level retired)──▶ [ARCHIVED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `ACTIVE` | Consumption level is currently used in alert logic | → `UPDATED`, → `ARCHIVED` |
| `UPDATED` | Level definition was modified | → `ACTIVE` |
| `ARCHIVED` | Level is no longer used in current alert rules | Terminal state |

**Invariants:**

```
INV-037: Ranges cannot overlap between levels.
INV-038: `min_limit` and `max_limit` must define a valid interval.
```

---

### Entity: Alert

**Context:** Alerts

**Description:** Notification raised when a relevant condition occurs, such as excessive consumption or a device that stops reporting.

**Cross-context references:** `homeId`, `deviceId`, `consumptionLevelId` and `measurementId` reference other bounded contexts at application level only; they stay plain `String`s because those aggregates cannot be imported here.

**Type consistency (MER note on `alert`):**
- **THRESHOLD alerts** must carry `consumptionLevelId` and `measurementId` (`ALERT-INV-007`).
- **CONNECTIVITY alerts** must carry neither (`ALERT-INV-008`).
- **DEVICE alerts** are informational, e.g. a module was linked to the home (`ALERT-INV-009`).
- **LIMIT alerts** are home-scoped, carrying no device (`ALERT-INV-010`).

The rule is enforced here, in the domain, so no persistence path can bypass it. Prefer the `threshold`, `connectivity`, `device`, and `limit` factories: they make an inconsistent alert unrepresentable by construction.

**Invariants:**

```
INV-039: An alert is created in `PENDING` and later resolved.
INV-040: `AlertSeverity` is not persisted; it is derived at runtime.
INV-041: A threshold alert must reference a valid `ConsumptionLevel` owned by Monitoring and Measurements, or a valid measurement.
INV-042: A connectivity alert must not carry `consumptionLevelId` nor `measurementId`.
INV-043: A device alert must not carry `consumptionLevelId` nor `measurementId`.
INV-044: A limit alert must not carry `consumptionLevelId` nor `measurementId` nor `deviceId`.
```

**Factories (use these instead of direct construction to enforce type consistency):**

| Factory | Parameters | Returns |
|---------|-----------|---------|
| `Alert.threshold(idAlert, homeId, deviceId, messageKey, dateTime, consumptionLevelId, measurementId)` | All as `String`; consumptionLevelId and measurementId required for THRESHOLD | New `Alert` with `type = THRESHOLD` |
| `Alert.connectivity(idAlert, homeId, deviceId, messageKey, dateTime)` | All as `String`; deviceId may be null | New `Alert` with `type = CONNECTIVITY` |
| `Alert.device(idAlert, homeId, deviceId, messageKey, dateTime)` | All as `String`; informational, resolving means "seen" | New `Alert` with `type = DEVICE` |
| `Alert.limit(idAlert, homeId, messageKey, dateTime)` | All as `String`; home-scoped, no device | New `Alert` with `type = LIMIT` |

**Lifecycle / States:**

```
[PENDING] ──(alert emitted)──▶ [ACTIVE] ──(condition resolved)──▶ [RESOLVED]
                              │
                  (alert dismissed or invalidated)
                              ▼
                           [CANCELLED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `PENDING` | Alert has been created but not yet acted on | → `ACTIVE`, → `CANCELLED` |
| `ACTIVE` | Alert is currently open and requires attention | → `RESOLVED`, → `CANCELLED` |
| `RESOLVED` | Alert has been handled successfully | Terminal state |
| `CANCELLED` | Alert was dismissed, invalidated, or superseded | Terminal state |

---

---

### Entity: Notification

**Context:** Notification

**Description:** A record that an outbound message was attempted, over which channel, and whether it succeeded. Owned by the Notification context, it answers the question no other module can: was a message actually sent, and did it fail?

**The message body is deliberately not stored.** A password recovery secret travels in that body, and a column holding it in clear text would undo the decision to keep only a hash elsewhere. What is stored instead is `messageKey`: a stable name for the wording used, which is enough to explain the row to somebody reading it during an audit and is not itself the message.

**Source identification:** The message originates from another bounded context, named by `sourceType` and `sourceId` rather than by a foreign key. This avoids database-level coupling between modules. The identifiers are therefore references without integrity: nothing here can check that the row they name still exists, and nothing here should.

**User reference:** `userId` is the one reference that is not optional, because a message with no recipient on file cannot be traced back to anybody. `homeId` is optional for messages that belong to an account and not to a home, such as a recovery code.

**Invariants:**

```
NOTIF-INV-001: a notification is created PENDING, never directly SENT.
NOTIF-INV-002: SENT carries the instant it was sent.
NOTIF-INV-003: FAILED carries a reason.
NOTIF-INV-004: SENT carries no reason, because it did not fail.
NOTIF-INV-005: a notification always names a user.
```

**readAt is part of the documented model and the column exists, but nothing writes it yet:** no endpoint reports that a message was opened, so the field is always null. It is kept because the model asks for it and adding the column later is a migration; the one thing to know is that a non-null value cannot be expected today.

**Lifecycle / States:**

```
[PENDING] ──(markAsSent)──▶ [SENT]
                │              │
                │          (markAsFailed)
                ▼              ▼
             [FAILED]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `PENDING` | A delivery attempt has not been made yet | → `SENT`, → `FAILED` |
| `SENT` | The message was successfully delivered | Terminal for this delivery attempt |
| `FAILED` | The message delivery failed | Terminal for this delivery attempt |

**Factory:**

| Factory | Parameters | Returns |
|---------|-----------|---------|
| `Notification.pending(idNotification, userId, homeId, channel, sourceType, sourceId, messageKey)` | `userId` required; `homeId` and `sourceId` optional; channel and sourceType required | New `Notification` with `deliveryStatus = PENDING` |

**Methods:**

| Method | Parameters | Effect |
|---------|-----------|--------|
| `markAsSent(sentAt)` | `sentAt` required | Sets `deliveryStatus = SENT`; clears `failureReason`; stores `sentAt` |
| `markAsFailed(reason)` | `reason` optional (truncated to column width) | Sets `deliveryStatus = FAILED`; stores `failureReason` |
| `markAsRead(readAt)` | `readAt` required; throws `IllegalStateException` if `deliveryStatus != SENT` | Stores `readAt`; only meaningful once the message has been sent |

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_notification` | `VARCHAR(10)` | Identifier | Yes | Primary key |
| `user_id` | `VARCHAR(10)` | The account the message is for | Yes | FK, not null |
| `home_id` | `VARCHAR(10)` | The home it concerns, optional | No | FK |
| `channel` | `NotificationChannel` | Delivery channel | Yes | Not null |
| `source_type` | `NotificationSourceType` | The context the message originates from | Yes | Not null |
| `source_id` | `VARCHAR(10)` | The row in that context, optional | No | FK |
| `message_key` | `VARCHAR(100)` | Stable name of the wording used | Yes | Not null |
| `delivery_status` | `NotificationStatus` | Current delivery status | Yes | Not null |
| `sent_at` | `LOCALDATE TIME` | When it was sent, present only when sent | Yes | — |
| `read_at` | `LOCALDATE TIME` | When it was read, absent while unread | No | Always null until sent |
| `failure_reason` | `VARCHAR(200)` | Why the message did not go out, present only when failed | No | — |
| `deleted_at` | `LOCALDATE TIME` | Soft delete marker | No | Terminal state |

**Methods (continued):**

| Method | Parameters | Effect |
|---------|-----------|--------|
| `failureReason(String reason)` | Raw reason from driver/mail server | Returns the reason, trimmed and cut to `MAX_REASON` (200) characters |
| `idNotification()` | — | Returns `id_notification` |
| `userId()` | — | Returns `user_id` |
| `homeId()` | — | Returns `home_id` |
| `channel()` | — | Returns `channel` |
| `sourceType()` | — | Returns `source_type` |
| `sourceId()` | — | Returns `source_id` |
| `messageKey()` | — | Returns `message_key` |
| `deliveryStatus()` | — | Returns `delivery_status` |
| `failureReason()` | — | Returns `failure_reason` |
| `sentAt()` | — | Returns `sent_at` |
| `readAt()` | — | Returns `read_at` |
| `deletedAt()` | — | Returns `deleted_at` |

---

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| `id_recommendation` | `VARCHAR(10)` | Recommendation identifier | Yes | Primary key |
| `id_home` | `VARCHAR(10)` | Affected home | Yes | FK, not null |
| `id_device` | `VARCHAR(10)` | Related device, if applicable | No | FK |
| `message_key` | `VARCHAR(100)` | Translatable message key | Yes | Not null |
| `date_time` | `DATETIME` | Creation timestamp | Yes | Not null |
| `consumption_level_id` | `VARCHAR(10)` | Related consumption level | No | FK |
| `status` | `RecommendationStatus` | Read/unread state | Yes | Not null |

**Cross-context references:** `idHome` and `idDevice` reference other bounded contexts at application level only; they stay plain `String`s because those aggregates cannot be imported here.

**Type consistency:** Recommendations are generated when a consumption pattern meets a threshold or when behavioral analysis identifies an opportunity. The `consumptionLevelId` links to a level defined in Monitoring and Measurements.

**Lifecycle / States:**

```
[UNREAD] ──(recommendation opened)──▶ [READ]
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| `UNREAD` | Recommendation has been created and not yet reviewed | → `READ` |
| `READ` | Recommendation has been opened or acknowledged by the user | Terminal state |

**Invariants:**

```
INV-042: A recommendation must be either `READ` or `UNREAD`.
INV-043: A recommendation informs the user; it does not block the flow.
INV-044: A recommendation may reference a `ConsumptionLevel` owned by Monitoring and Measurements, or carry none.
INV-045: A recommendation must reference a valid home owned by Home Management.
```

**Factory:**

| Factory | Parameters | Returns |
|---------|-----------|---------|
| `Recommendation(idRecommendation, idHome, idDevice, messageKey, dateTime, consumptionLevelId, status)` | All as `String`; consumptionLevelId and deviceId optional | New `Recommendation` with given status |

**Methods:**

| Method | Parameters | Effect |
|---------|-----------|--------|
| `markAsRead()` | — | Changes `status` from `UNREAD` to `READ`; throws `IllegalStateException` if already `READ` |
| `idRecommendation()` | — | Returns `id_recommendation` |
| `homeId()` | — | Returns `id_home` |
| `deviceId()` | — | Returns `id_device` |
| `messageKey()` | — | Returns `message_key` |
| `consumptionLevelId()` | — | Returns `consumption_level_id` |
| `status()` | — | Returns `status` |

---

## 3. System Value Objects

### Value Object: Email

**Description:** User email address, validated and normalized.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `String` | Normalized, lowercase email address |

**Validation rules:**

```
- Must follow the standard email format (user@domain.extension).
- Must be normalized to lowercase.
```

**Example:**

```java
// Value Object — immutable, validated in the constructor
public final class Email {

    private final String value;

    public Email(String rawEmail) {
        if (rawEmail == null || !rawEmail.matches("^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$")) {
            throw new DomainException("Invalid email: " + rawEmail);
        }
        this.value = rawEmail.toLowerCase();
    }

    public String value() {
        return value;
    }

    @Override
    public boolean equals(Object other) {
        return other instanceof Email && this.value.equals(((Email) other).value);
    }
}
```

### Value Object: AccessCode

**Description:** Short 8-character code used to join a home.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `String(8)` | Access code used to join a home |

**Validation rules:**

```
- Must be unique across the system.
- Must use the exact configured length and be generated by the domain.
```

### Value Object: Threshold

**Description:** A single consumption limit configured for a home, expressed in kilowatt-hours.
`HomeThresholds` holds two of these — one for the daily limit and one for the monthly limit —
instead of two unrelated numbers, so the validation and comparison rule lives in one place.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `Double` | Limit expressed in kWh |

**Validation rules:**

```
- Must be strictly greater than zero — a threshold of 0 or less has no domain meaning.
- Exposes isExceededBy(consumption: Double): Boolean, so the comparison rule is
  never duplicated wherever a threshold is checked against a measured consumption.
```

### Value Object: DeviceCode

**Description:** Unique short code used to identify and authenticate a device.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `String` | Unique code identifying a device |

**Validation rules:**

```
- Must be unique.
- Must be generated and not reused by another device.
```

### Value Object: ApiKey

**Description:** Secret key assigned to a device to authenticate it against the backend.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `String` | Secret key used to authenticate a device |

**Validation rules:**

```
- Must be stored as a secret.
- Must never be exposed in responses or logs.
- Must expose a masked representation (e.g. toString()) when displayed or logged,
  so accidental logging never leaks the real value.
```

### Value Object: ActivePower

**Description:** Instantaneous real power drawn by a load, in watts. Wrapping it in its own type
makes it impossible to pass an `ActivePower` where an `Energy` value is expected, or vice versa —
the two are easy to confuse as plain numbers but mean physically different things (see
"Terms that must not be confused" in `02-domain/domain-map.md`).

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `Double` | Power expressed in watts (W) |

**Validation rules:**

```
- Must not be negative.
```

### Value Object: Energy

**Description:** Accumulated electrical consumption over time, in kilowatt-hours. Used by
`Measurement.stored_energy`. Kept as a distinct type from `ActivePower` for the same reason.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `Double` | Energy expressed in kilowatt-hours (kWh) |

**Validation rules:**

```
- Must not be negative.
- Must be monotonically non-decreasing across consecutive measurements for the same device,
  since accumulated energy cannot go backward.
```

### Value Object: MessageKey

**Description:** Identifier of a translatable message for alerts or recommendations.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `value` | `String` | Key referencing an entry in the message catalog |

**Validation rules:**

```
- Must point to a system message catalog.
- Must not store literal text directly in the database.
```

---

## 4. System Aggregates

### Aggregate: User Aggregate

**Aggregate Root:** `User`

**Internal entities:**
- `Person` — personal identity data
- `UserConfiguration` — user preferences
- `UserSession` — authenticated sessions
- `PasswordResetToken` — password recovery tokens

**Value Objects:**
- `Email`

**Aggregate invariants:**

```
AGGR-INV-001: A user must always have an associated `Person`.
AGGR-INV-002: A user can have at most one personal configuration record.
AGGR-INV-003: Sessions are a set of revocable records, not one global token.
```

**Why these objects form an aggregate:**
> The user is the primary entity of the security context. Identity, credentials, sessions, preferences, and recovery flows must remain coherent as a single unit of business responsibility.

---

### Aggregate: Home Aggregate

**Aggregate Root:** `Home`

**Internal entities:**
- `HomeThresholds` — home-level limits
- `UserHome` — membership within the home

**Value Objects:**
- `AccessCode`

**Aggregate invariants:**

```
AGGR-INV-004: Each home must have a single threshold configuration.
AGGR-INV-005: The role within a home is independent from the global system role.
AGGR-INV-006: A home may have multiple members and multiple devices.
```

**Why these objects form an aggregate:**
> The home is the main organizational unit of the business domain. Its thresholds, members, and configuration must stay together so that devices, alerts, and recommendations are attributed correctly.

---

### Aggregate: Device Aggregate

**Aggregate Root:** `Device`

**Internal entities:**
- `DeviceStatusLog` — device connection history
- `DeviceHome` — relationships with homes

**Value Objects:**
- `DeviceCode`, `ApiKey`

**Aggregate invariants:**

```
AGGR-INV-007: A device must have a valid appliance type and valid API key.
AGGR-INV-008: The state history must not replace the identity of the device.
AGGR-INV-009: A device may belong to different homes over time.
```

**Why these objects form an aggregate:**
> The device identity, authentication, and connectivity history are one cohesive unit that changes for reasons different from the measurements it emits.

---

### Aggregate: Alert Aggregate

**Aggregate Root:** `Alert`

**Internal entities:**
- (none — `Alert` has no entities of its own; it references other aggregates by ID)

**Referenced by ID (owned by a different bounded context, never embedded):**
- `ConsumptionLevel` (owned by Monitoring and Measurements) — via `id_consumption_level`
- `Measurement` (owned by Monitoring and Measurements) — via `id_measurement`
- `Home` (owned by Home Management) — via `id_home`
- `Device` (owned by Devices) — via `id_device`

**Value Objects:**
- `MessageKey`

**Aggregate invariants:**

```
AGGR-INV-010: A threshold alert must reference a valid consumption level owned by Monitoring and Measurements.
AGGR-INV-011: Severity is not persisted; it is derived at runtime.
```

**Why these objects form an aggregate:**
> The alert is a single-entity aggregate: it groups the condition that occurred and points to the context that explains it, without owning any of that context's data. Severity is calculated at runtime, so storage remains simpler than evaluation.

---

## 5. Summary table of tactical building blocks

| Name | Type | Bounded Context | Aggregate Root? |
|------|------|----------------|----------------|
| `Person` | Entity | Security | No |
| `User` | Entity | Security | Yes |
| `UserConfiguration` | Entity | Security | No |
| `PasswordResetToken` | Entity | Security | No |
| `UserSession` | Entity | Security | No |
| `SecurityConfiguration` | Entity | Security | No |
| `PasswordPolicy` | Entity | Security | No |
| `LoginErrorLog` | Entity | Security | No |
| `SystemRole` | Entity | Security | No |
| `Permission` | Entity | Security | No |
| `UserSystemRole` | Entity | Security | No |
| `SystemRolePermission` | Entity | Security | No |
| `AuditLog` | Entity | Security | No |
| `HomeType` | Entity | Home Management | No |
| `Home` | Entity | Home Management | Yes |
| `HomeThresholds` | Entity | Home Management | No |
| `UserHome` | Entity | Home Management | No |
| `ApplianceType` | Entity | Devices | No |
| `Device` | Entity | Devices | Yes |
| `DeviceStatusLog` | Entity | Devices | No |
| `DeviceHome` | Entity | Devices | No |
| `Measurement` | Entity | Monitoring and Measurements | Yes |
| `ConsumptionLevel` | Entity | Monitoring and Measurements | No |
| `Alert` | Entity | Alerts | Yes |
| `Recommendation` | Entity | Recommendations | Yes |
| `Email` | Value Object | Security | N/A |
| `AccessCode` | Value Object | Home Management | N/A |
| `Threshold` | Value Object | Home Management | N/A |
| `DeviceCode` | Value Object | Devices | N/A |
| `ApiKey` | Value Object | Devices | N/A |
| `ActivePower` | Value Object | Monitoring and Measurements | N/A |
| `Energy` | Value Object | Monitoring and Measurements | N/A |
| `MessageKey` | Value Object | Alerts / Recommendations | N/A |

---

## 6. Domain Services

A **Domain Service** encapsulates business logic that does not naturally belong to a single entity.

### `ConsumptionClassificationService`
**Context:** Monitoring and Measurements
**Purpose:** Classifies a measurement into a consumption level such as `LOW`, `MEDIUM`, `HIGH`, or `CRITICAL` based on the ranges defined by `ConsumptionLevel`.

### `AlertEvaluationService`
**Context:** Alerts
**Purpose:** Decides whether a reading or a connectivity loss should raise an alert, and whether the alert is `THRESHOLD` or `CONNECTIVITY`.

### `RecommendationGenerationService`
**Context:** Recommendations
**Purpose:** Generates recommendations based on the home's historical consumption patterns or device behavior.

### `TokenLifecycleService`
**Context:** Security
**Purpose:** Manages creation, validation, renewal, and revocation of sessions and authentication tokens.

### `HomeMembershipService`
**Context:** Home Management
**Purpose:** Handles access to a home, role-based permissions, and user-to-home associations.

```java
// Domain Service — framework-free, no Spring or JPA dependency
public class ConsumptionClassificationService {

    public RiskConsumption classify(double value, List<ConsumptionLevel> levels) {
        return levels.stream()
            .filter(level -> value >= level.getMinLimit() && value <= level.getMaxLimit())
            .findFirst()
            .map(ConsumptionLevel::getName)
            .orElseThrow(() -> new DomainException("No consumption level matches the given value"));
    }
}
```

---

## 7. Relationship rules and business constraints

### Cross-context relationships
- `Security` is upstream for identity and authorization.
- `Home Management` defines membership and home-scoped roles.
- `Devices` owns the measured hardware and its connectivity state.
- `Monitoring and Measurements` is the append-only source of electrical readings.
- `Alerts` reacts to threshold violations and device silence.
- `Recommendations` reacts to consumption history and behavioral patterns.

### Important invariants
- No bounded context should read another context's tables directly.
- The shared physical database is only a persistence detail; business ownership is still contained in each context.
- An alert or recommendation may reference a home, a device, and/or a measurement, but it must never become the source of truth for those entities.

---

## 8. Correlation with code

| Domain artifact | Suggested code location | Responsibility |
|----------------|--------------------------|----------------|
| `User`, `Person`, `UserSession`, `Email` | `security/domain/` | Identity and session lifecycle |
| `SystemRole`, `Permission`, `AuditLog` | `security/domain/access/` | RBAC and auditing |
| `Home`, `HomeThresholds`, `UserHome`, `AccessCode`, `Threshold` | `home/domain/` | Home membership and policy |
| `Device`, `DeviceStatusLog`, `DeviceHome`, `DeviceCode`, `ApiKey` | `devices/domain/` | Hardware lifecycle and ownership |
| `Measurement`, `ConsumptionLevel`, `ActivePower`, `Energy` | `measurements/domain/` | Immutable electrical readings and consumption classification |
| `Alert`, `MessageKey` | `alerts/domain/` | Reactive notifications |
| `Recommendation`, `MessageKey` | `recommendations/domain/` | Proactive suggestions |

> `MessageKey` is defined locally in both `alerts/domain/` and `recommendations/domain/` with the
> same shape. It is intentionally duplicated rather than shared as a library, since this project
> does not use a Shared Kernel between bounded contexts (see `02-domain/domain-map.md`).

> The exact folder structure may vary depending on the stack, but the domain organization above should be preserved to protect the business model.