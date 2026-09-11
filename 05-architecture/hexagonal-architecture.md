# Hexagonal Architecture (Ports & Adapters)

> Hexagonal architecture, proposed by Alistair Cockburn, organizes a service so that the
> **business domain is completely independent** of the surrounding technology.
> The database, the web framework, the message broker — all are interchangeable details.
> What matters is the business logic, which lives at the center.

> **Stack note:** all six microservices in this project (`security-service`, `home-service`,
> `device-service`, `measurement-service`, `alert-service`, `recommendation-service`) are built on
> **Java 21 + Spring Boot** (`01-context/overview.md`). There is no second stack to support, so
> this document gives the Java/Spring examples directly instead of deferring to a separate
> `_stacks/java-spring.md` file — there would be nothing left in this document if the stack-specific
> part were split out.

---

## The problem it solves

```
❌ Traditional layered architecture:

  [HTTP Controller]
       ↓
  [Service]
       ↓
  [Repository]
       ↓
  [Database]

Problem: The "Service" mixes business logic with framework calls.
If you change the framework, you break the business. If you want to test the business,
you need to simulate the database.
```

```
✓ Hexagonal Architecture:

  [HTTP Controller]  [MQTT Consumer]  [Test]  ← Primary Adapters (enter the hexagon)
          │                 │           │
          └─────────────────┴───────────┘
                       │
             [Use Case class]  ← the driving port, called directly (see note below)
                       │
               ┌───────────────┐
               │               │
               │    DOMAIN     │  ← Pure business logic, no Spring, no JPA
               │               │
               └───────────────┘
                       │
                 [Driven Port]  ← Interface the domain needs from the outside world
                       │
          ┌────────────┴───────────────┐
          │                            │
  [Persistence Adapter]     [Security Adapter (JWT, hashing)]  ← Secondary Adapters (exit the hexagon)
```

---

## Folder structure — the project standard

This is the actual structure of `security-service`, and it is the standard every one of the six
microservices in this project follows — not a suggestion to adapt, a layout to copy:

```
src/main/java/.../security/
├── adapter/
│   ├── in/
│   │   └── web/                          # Primary adapters — one Controller per aggregate/concern
│   │       ├── AuditController.java
│   │       ├── AuthController.java
│   │       ├── PasswordController.java
│   │       ├── PermissionController.java
│   │       ├── RoleController.java
│   │       ├── SecurityConfigurationController.java
│   │       ├── SessionController.java
│   │       └── UserController.java
│   └── out/                              # Secondary adapters — implement application/port/out
│       ├── client/
│       │   └── EmailSenderClient.java     # implements EmailSender
│       ├── persistence/
│       │   ├── entity/                   # @Entity classes — persistence entities, never the domain model
│       │   ├── mapper/                   # Entity ↔ domain model translation
│       │   ├── repository/               # Spring Data JpaRepository interfaces (framework-facing)
│       │   └── *RepositoryAdapter.java    # implements application/port/out/*Repository
│       └── security/
│           ├── JwtTokenProvider.java      # implements TokenProvider
│           └── PasswordHasherImpl.java    # implements PasswordHasher
├── application/
│   ├── port/
│   │   ├── in/                            # (empty — see "Driving Port" note below)
│   │   └── out/                           # Driven ports — what the domain needs from outside
│   │       ├── AuditRepository.java
│   │       ├── EmailSender.java
│   │       ├── LoginErrorLogRepository.java
│   │       ├── PasswordHasher.java
│   │       ├── PasswordPolicyRepository.java
│   │       ├── PasswordResetTokenRepository.java
│   │       ├── PermissionRepository.java
│   │       ├── RoleRepository.java
│   │       ├── SecurityConfigurationRepository.java
│   │       ├── SessionRepository.java
│   │       ├── TokenProvider.java
│   │       └── UserRepository.java
│   └── usecase/                           # One class per business operation — see table below
│       ├── AuthenticateUserUseCase.java
│       ├── CreateUserUseCase.java
│       ├── ...
├── domain/
│   ├── exception/                         # Domain exceptions — no HTTP status here, see Controllers
│   │   ├── AccountBlockedException.java
│   │   ├── AccountInactiveException.java
│   │   ├── InvalidCredentialsException.java
│   │   ├── TokenAlreadyUsedException.java
│   │   ├── TokenExpiredException.java
│   │   └── UserNotFoundException.java
│   ├── model/                             # Aggregates and Entities — framework-free
│   │   ├── User.java
│   │   ├── Session.java
│   │   ├── Role.java
│   │   ├── Permission.java
│   │   └── ...
│   └── valueobject/
│       ├── AuditAction.java
│       ├── LoginErrorType.java
│       └── UserStatus.java
├── infrastructure/
│   ├── config/                            # Spring wiring only — never business rules
│   │   ├── AuthorizationServerConfig.java
│   │   ├── OpenApiConfig.java
│   │   └── SecurityConfig.java
│   ├── exception/                         # @ControllerAdvice — maps domain exceptions → HTTP
│   └── observability/
└── MsSecurityApplication.java             # Spring Boot bootstrap
```

`home-service`, `device-service`, `measurement-service`, `alert-service`, and
`recommendation-service` mirror this exact package layout, with their own aggregates, use cases,
and ports in place of Security's.

---

## The Ports

Ports are **interfaces** (abstract contracts). The domain and application layers define them;
adapters implement them.

### Driving Port (Input Port) — a deliberate simplification in this project

The textbook version of this pattern puts a separate interface in `application/port/in/` (e.g.
`AuthenticateUserPort`) that the use case class implements, so a controller depends on the
interface rather than the concrete class. **This project does not do that** — as the folder
structure above shows, `application/port/in/` is empty. Controllers inject the concrete use case
class directly:

```java
// application/usecase/AuthenticateUserUseCase.java — this class itself IS the driving port
@Service
public class AuthenticateUserUseCase {
    public AuthenticationResult execute(AuthenticateUserCommand command) { ... }
}
```

This is a conscious trade-off, not an oversight: with one implementation per use case and no
plan to swap it, the extra interface adds a layer of indirection without a corresponding benefit.
The cost is that a controller technically depends on a concrete class instead of an abstraction —
acceptable here because nothing in this project ever substitutes a different implementation of
`AuthenticateUserUseCase` at runtime. If that ever changes for a specific use case, that specific
one gets its own `port/in` interface; the rest do not need to change to stay consistent.

### What the Security domain can do — its public API from the outside

Every use case below is something a caller (a Controller today) can ask the Security domain to
do. This is the domain's complete public surface — anything not on this list is not something an
outside caller can make the Security domain do directly.

| Use case                             | What it does                                                                 |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `CreateUserUseCase`                  | Registers a new `Person` + `User` pair                                       |
| `AuthenticateUserUseCase`            | Verifies credentials, issues an access/refresh token pair, opens a `Session` |
| `AuthorizeUserUseCase`               | Checks whether an already-authenticated caller holds a given `Permission`    |
| `RefreshSessionUseCase`              | Exchanges a valid refresh token for a new access token                       |
| `LogoutUseCase`                      | Closes the current session                                                   |
| `RevokeSessionUseCase`               | Invalidates a specific session (e.g. from another device)                    |
| `RequestPasswordResetUseCase`        | Issues a `PasswordResetToken` and triggers the reset email                   |
| `ResetPasswordUseCase`               | Consumes a valid, unused, unexpired reset token to set a new password        |
| `UpdateUserProfileUseCase`           | Updates a user's `Person`-level data                                         |
| `UpdateUserConfigurationUseCase`     | Updates notification/theme/language preferences                              |
| `DeleteAccountUseCase`               | Deletes a user's account                                                     |
| `AssignRoleUseCase`                  | Grants a `SystemRole` to a user                                              |
| `AssignPermissionToRoleUseCase`      | Grants a `Permission` to a `SystemRole`                                      |
| `UpdatePasswordPolicyUseCase`        | Changes the enforced password rules                                          |
| `UpdateSecurityConfigurationUseCase` | Changes a tunable security parameter                                         |
| `RegisterAuditUseCase`               | Writes one `AuditLog` entry                                                  |
| `RegisterLoginErrorUseCase`          | Writes one `LoginErrorLog` entry and applies the lockout rule                |

### Driven Port (Output Port)

Defines what the domain needs from the outside world — without knowing how it is implemented.
This is "the interior perspective without knowing how it's implemented": the use case class below
is written entirely against these interfaces, never against `UserJpaRepository`, `BCrypt`, or any
other concrete technology.

```java
// application/port/out/UserRepository.java
public interface UserRepository {
    User save(User user);
    Optional<User> findById(UserId id);
    Optional<User> findByEmail(Email email);
    boolean existsByEmail(Email email);
}

// application/port/out/PasswordHasher.java
public interface PasswordHasher {
    String hash(String rawPassword);
    boolean matches(String rawPassword, String hash);
}

// application/port/out/TokenProvider.java
public interface TokenProvider {
    String issueAccessToken(UserId userId, Set<String> permissionCodes);
    String issueRefreshToken(SessionId sessionId);
    Optional<UserId> validateAccessToken(String token);
}
```

The full inventory for Security: `AuditRepository`, `EmailSender`, `LoginErrorLogRepository`,
`PasswordHasher`, `PasswordPolicyRepository`, `PasswordResetTokenRepository`,
`PermissionRepository`, `RoleRepository`, `SecurityConfigurationRepository`, `SessionRepository`,
`TokenProvider`, `UserRepository` — one port per capability the domain needs, never one giant
port that bundles unrelated capabilities together.

---

## The Adapters

### Primary Adapter — how the Controller translates an HTTP request into a domain use case

The controller's only job is translation: HTTP request → domain command, and domain result → HTTP
response. It contains no business rule — if you find an `if` statement enforcing a business rule
inside a controller, that rule belongs in the domain model or the use case, not here.

```java
// adapter/in/web/AuthController.java
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthenticateUserUseCase authenticateUserUseCase;

    public AuthController(AuthenticateUserUseCase authenticateUserUseCase) {
        this.authenticateUserUseCase = authenticateUserUseCase;
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponseDto> login(@RequestBody @Valid LoginRequestDto request) {
        // 1. Translate the HTTP DTO into a domain command — no business meaning yet, just shape
        AuthenticateUserCommand command =
            new AuthenticateUserCommand(request.email(), request.password());

        // 2. Call the use case — this line is the entire boundary crossing
        AuthenticationResult result = authenticateUserUseCase.execute(command);

        // 3. Translate the domain result back into an HTTP DTO — the client never sees
        //    AuthenticationResult, User, or Session directly
        AuthResponseDto response =
            new AuthResponseDto(result.accessToken(), result.refreshToken());
        return ResponseEntity.ok(response);
    }
}
```

Domain exceptions (`InvalidCredentialsException`, `AccountBlockedException`, ...) propagate up
from the use case uncaught by the controller itself — mapping them to an HTTP status happens once,
centrally, in `infrastructure/exception/` (a `@ControllerAdvice`), not with a `try/catch` block
repeated in every controller method. This is the concrete answer to the open question
`05-architecture/cross-cutting.md` raises under "Error handling": the mapping table that document
says does not exist yet is exactly what belongs in that `@ControllerAdvice` class.

### Secondary Adapter — implementing the output Repository

The repository adapter implements the driven port. The domain does not know MySQL, JPA, or
Hibernate exist — it only knows `UserRepository`.

```java
// adapter/out/persistence/entity/UserEntity.java — persistence entity, maps the table 1:1
@Entity
@Table(name = "user")
public class UserEntity {
    @Id
    private String idUser;
    private String idPerson;
    private String passwordHash;
    private String email;
    private boolean emailVerified;
    private LocalDateTime dateOfRegistation;
    @Enumerated(EnumType.STRING)
    private UserStatusEntity status;
    private int failedLoginAttempts;
    private LocalDateTime lastLoginAt;
    // getters/setters — no business logic lives here
}

// adapter/out/persistence/repository/UserJpaRepository.java — Spring Data, framework-facing only
public interface UserJpaRepository extends JpaRepository<UserEntity, String> {
    Optional<UserEntity> findByEmail(String email);
    boolean existsByEmail(String email);
}

// adapter/out/persistence/mapper/UserMapper.java — translates in both directions
@Component
public class UserMapper {
    public User toDomain(UserEntity entity) { /* UserEntity → User aggregate */ }
    public UserEntity toEntity(User domain) { /* User aggregate → UserEntity */ }
}

// adapter/out/persistence/UserRepositoryAdapter.java — implements the driven port
@Component
public class UserRepositoryAdapter implements UserRepository {

    private final UserJpaRepository jpaRepository;
    private final UserMapper mapper;

    public UserRepositoryAdapter(UserJpaRepository jpaRepository, UserMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }

    @Override
    public User save(User user) {
        UserEntity saved = jpaRepository.save(mapper.toEntity(user));
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<User> findByEmail(Email email) {
        return jpaRepository.findByEmail(email.value()).map(mapper::toDomain);
    }

    @Override
    public boolean existsByEmail(Email email) {
        return jpaRepository.existsByEmail(email.value());
    }

    @Override
    public Optional<User> findById(UserId id) {
        return jpaRepository.findById(id.value()).map(mapper::toDomain);
    }
}
```

Every other repository port (`SessionRepository`, `PermissionRepository`, ...) is implemented the
same way: one `*Entity`, one `*JpaRepository`, one `*Mapper`, one `*RepositoryAdapter` — the
pattern does not change per aggregate, only the fields do.

### Secondary Adapter — a category the generic template does not have: `adapter/out/security/`

Not every driven port is a repository. `TokenProvider` and `PasswordHasher` are adapters too, just
not persistence ones:

```java
// adapter/out/security/PasswordHasherImpl.java
@Component
public class PasswordHasherImpl implements PasswordHasher {

    private final PasswordEncoder encoder; // Spring Security's BCryptPasswordEncoder

    public PasswordHasherImpl(PasswordEncoder encoder) {
        this.encoder = encoder;
    }

    @Override
    public String hash(String rawPassword) {
        return encoder.encode(rawPassword);
    }

    @Override
    public boolean matches(String rawPassword, String hash) {
        return encoder.matches(rawPassword, hash);
    }
}

// adapter/out/security/JwtTokenProvider.java
@Component
public class JwtTokenProvider implements TokenProvider {
    // Wraps a JWT library. The domain calls TokenProvider.issueAccessToken(...) and
    // never sees a JWT library type — swapping JJWT for Nimbus JOSE is a change confined
    // entirely to this one class.
}
```

This is exactly what the `PasswordHasher` outbound port already protects against, per
`06-data/data-dictionary.md`'s note on `user.password_hash`: the concrete hashing algorithm is
swappable without ever touching `AuthenticateUserUseCase` or `CreateUserUseCase`.

---

## The Use Case (Application Service)

The use case orchestrates the domain. It calls driven ports. It contains no business rule that
belongs to an Aggregate — those rules live in the domain model (`User.authenticate(...)`,
`INV-004`, `INV-005`), not copy-pasted into the use case.

```java
// application/usecase/AuthenticateUserUseCase.java
@Service
public class AuthenticateUserUseCase {

    private final UserRepository userRepository;
    private final PasswordHasher passwordHasher;
    private final TokenProvider tokenProvider;
    private final SessionRepository sessionRepository;
    private final RegisterLoginErrorUseCase registerLoginErrorUseCase;

    public AuthenticateUserUseCase(UserRepository userRepository,
                                    PasswordHasher passwordHasher,
                                    TokenProvider tokenProvider,
                                    SessionRepository sessionRepository,
                                    RegisterLoginErrorUseCase registerLoginErrorUseCase) {
        this.userRepository = userRepository;
        this.passwordHasher = passwordHasher;
        this.tokenProvider = tokenProvider;
        this.sessionRepository = sessionRepository;
        this.registerLoginErrorUseCase = registerLoginErrorUseCase;
    }

    public AuthenticationResult execute(AuthenticateUserCommand command) {
        // 1. Load the aggregate (through the port — this class does not know it is MySQL)
        User user = userRepository.findByEmail(command.email())
            .orElseThrow(() -> {
                registerLoginErrorUseCase.execute(LoginErrorType.USER_NOT_FOUND, null, command.ip());
                return new UserNotFoundException();
            });

        // 2. Business logic lives in the Aggregate, not here — INV-005: BLOCKED cannot authenticate
        if (!passwordHasher.matches(command.rawPassword(), user.passwordHash())) {
            user.registerFailedAttempt(); // may transition status to BLOCKED — see INV-005
            userRepository.save(user);
            registerLoginErrorUseCase.execute(LoginErrorType.INVALID_PASSWORD, user.id(), command.ip());
            throw new InvalidCredentialsException();
        }
        user.registerSuccessfulLogin(); // resets failed_login_attempts to 0

        // 3. Persist the updated aggregate state
        userRepository.save(user);

        // 4. Issue tokens and open a session (through ports — no JWT library imported here)
        Session session = Session.open(user.id(), command.ip(), command.userAgent());
        sessionRepository.save(session);
        String accessToken = tokenProvider.issueAccessToken(user.id(), user.permissionCodes());
        String refreshToken = tokenProvider.issueRefreshToken(session.id());

        return new AuthenticationResult(accessToken, refreshToken);
    }
}
```

---

## The Dependency Rule

> **Dependencies always point inward.**
> `domain/` does not import anything from `application/` or `infrastructure/`.
> `infrastructure/` imports from `domain/` and `application/` (never the other way around).

```
adapter/ (infrastructure) → application/ → domain/
                                              ↑
                                CANNOT import anything from application/ or adapter/
```

Concretely for this project: `domain/model/User.java` never imports `jakarta.persistence.*`,
`org.springframework.*`, or anything from `adapter/`. If a domain class needs an `@Entity`
annotation to compile, that class is not in `domain/` — it belongs in
`adapter/out/persistence/entity/`.

### Dependency inversion (DI) in practice

```java
// ✓ Correct — the port lives in application/, the adapter implements it
// In application/port/out/:
public interface UserRepository { ... }

// In adapter/out/persistence/:
@Component
public class UserRepositoryAdapter implements UserRepository { ... }

// Spring's IoC container wires them together automatically by type at startup —
// there is no manual container.ts to maintain, but the direction of the dependency
// is identical to the manual-DI version: the use case asks for UserRepository (the
// interface), Spring hands it a UserRepositoryAdapter (the implementation) at runtime.
@Service
public class AuthenticateUserUseCase {
    public AuthenticateUserUseCase(UserRepository userRepository, /* ... */) { ... }
}
```

---

## Advantages for TDD

Hexagonal architecture is ideal for TDD because:

1. **The domain is testable without framework mocks.** No Spring context, no database, no MQTT
   broker needs to start to test `User`'s business logic.

2. **Driven ports can be faked easily.** In tests, an in-memory `UserRepository` implementation
   replaces `UserRepositoryAdapter` — no `@DataJpaTest`, no test database.

3. **Invariants are explicit** and tested in isolation.

```java
// Domain unit test — zero external dependencies, zero Spring annotations
class UserTest {

    @Test
    void blockedAccountCannotAuthenticateAgain() {
        User user = User.register(personId, "some-hash", email);
        for (int i = 0; i < MAX_ATTEMPTS; i++) {
            user.registerFailedAttempt();
        }
        assertEquals(UserStatus.BLOCKED, user.status()); // INV-005

        assertThrows(AccountBlockedException.class, user::assertCanAuthenticate);
    }

    @Test
    void successfulLoginResetsFailedAttempts() {
        User user = User.register(personId, "some-hash", email);
        user.registerFailedAttempt();
        user.registerSuccessfulLogin();
        assertEquals(0, user.failedLoginAttempts());
    }
}

// Use case test with a FAKE repository (not a real DB, not @DataJpaTest)
class AuthenticateUserUseCaseTest {

    @Test
    void savesTheUpdatedUserAndIssuesTokensOnSuccess() {
        InMemoryUserRepository fakeUserRepo = new InMemoryUserRepository();
        fakeUserRepo.save(existingUser);
        PasswordHasher fakeHasher = (raw, hash) -> true; // stubbed, matches always
        TokenProvider fakeTokenProvider = new InMemoryTokenProvider();

        AuthenticateUserUseCase useCase = new AuthenticateUserUseCase(
            fakeUserRepo, fakeHasher, fakeTokenProvider, fakeSessionRepo, registerLoginErrorUseCase);

        AuthenticationResult result = useCase.execute(new AuthenticateUserCommand(email, "pw"));

        assertNotNull(result.accessToken());
        assertEquals(0, fakeUserRepo.findByEmail(email).get().failedLoginAttempts());
    }
}
```

> See the full TDD guide in `11-quality/tdd-guide.md`.

---

## Hexagonal Architecture Checklist

When reviewing a PR or a new service, verify:

- [ ] `domain/` has no imports from `application/` or `adapter/`
- [ ] `domain/` has no imports from frameworks (`org.springframework.*`, `jakarta.persistence.*`)
- [ ] Every repository/external-capability interface lives in `application/port/out/`
- [ ] `application/port/in/` stays empty **on purpose** — use case classes are the driving port
      directly (see the note above). A PR that adds a new `port/in` interface for one specific
      use case is fine if justified; a PR that adds one "for consistency" with no runtime reason
      to substitute the implementation is the anti-pattern to push back on
- [ ] Mappers (`toDomain` / `toEntity`) live in `adapter/out/persistence/mapper/`, never in `domain/`
- [ ] HTTP request/response DTOs live in `adapter/in/web/`, never in `domain/` or `application/`
- [ ] `@Entity` classes live in `adapter/out/persistence/entity/`, never used as the domain model
- [ ] There is a unit test for each Aggregate invariant (`INV-XXX` in `02-domain/entities-and-rules.md`)

---

## Common mistakes (anti-patterns)

| Anti-pattern                                                                                                      | Why it is bad                                                                                                                                          | Solution                                                                                   |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `import jakarta.persistence.Entity` on a class in `domain/model/`                                                 | Couples the domain to JPA — the domain now cannot exist without a JPA provider on the classpath                                                        | Keep `@Entity` classes in `adapter/out/persistence/entity/`; map with a dedicated Mapper   |
| Business logic in the Controller (e.g. checking `failedLoginAttempts` in `AuthController`)                        | If the endpoint changes, the business rule silently changes with it, and no other entry point (a future MQTT consumer, a batch job) gets the same rule | Move the rule into the `User` aggregate as a method, called from the use case              |
| A `*RepositoryAdapter` returning `UserEntity` instead of `User` from a method typed as `UserRepository`           | Breaks the contract — the domain would receive a JPA-managed object it cannot validate its own invariants against                                      | The Mapper always converts before returning from the adapter                               |
| A use case with 8+ port dependencies in its constructor                                                           | It is probably doing more than one business operation                                                                                                  | Split into smaller use cases, one operation each                                           |
| Catching a domain exception inside every Controller method                                                        | Duplicates the same mapping logic six, eight, sixteen times across the service                                                                         | One `@ControllerAdvice` in `infrastructure/exception/` maps every domain exception once    |
| A new `port/in` interface added "just in case" for a use case with one implementation and no plan to add a second | Adds indirection with no benefit — the opposite of what this project already decided for `port/in`                                                     | Keep the use case class as the driving port unless a concrete need to substitute it exists |

---

## References and correlations

- Bounded Contexts → `02-domain/domain-map.md`
- Entities, invariants, and Value Objects (`INV-004`, `INV-005`, `Threshold`, `ApiKey`, ...) → `02-domain/entities-and-rules.md`
- Domain events published/consumed by use cases → `02-domain/domain-events.md`
- Complementary patterns already cataloged for this project (Repository, Use Case, Observer, Factory Method, Singleton, Idempotent Consumer, DLQ, plus Saga/Outbox as not-yet-adopted) → `05-architecture/pattern-guide.md`
- Error-handling gap this document's `@ControllerAdvice` example resolves → `05-architecture/cross-cutting.md`
- Term definitions (Port, Adapter, Use Case, Persistence Entity, Mapper, DTO) → `01-context/glossary.md`
- Per-service documentation once each of the six services is filled in → `09-microservices/services/` (currently only the generic `_template/service/` scaffold exists)
- TDD applied to this same hexagonal structure → `11-quality/tdd-guide.md`
