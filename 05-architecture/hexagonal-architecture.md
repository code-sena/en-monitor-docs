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

## Folder structure — generic, for any service

This is the shape every service in this project follows. Replace `Xxx` with the aggregate or
concept the service owns — this same layout applies to `security-service`, `home-service`,
`device-service`, `measurement-service`, `alert-service`, and `recommendation-service` alike.

```
[service-name]-service
│
├── adapter/
│   ├── in/
│   │   └── web/
│   │       └── XxxController.java           # Translates HTTP requests into use case calls
│   │
│   └── out/
│       ├── client/
│       │   └── XxxClient.java                # Calls another service or an external system
│       │
│       ├── persistence/
│       │   ├── entity/
│       │   │   └── XxxEntity.java            # JPA mapping — never used inside domain/
│       │   ├── mapper/
│       │   │   └── XxxMapper.java            # Translates Entity ↔ Domain Model
│       │   ├── repository/
│       │   │   └── XxxJpaRepository.java     # Spring Data interface
│       │   └── XxxRepositoryAdapter.java     # Implements the domain's outbound port
│       │
│       └── security/                          # Only where the service issues or reads tokens
│           └── XxxTokenProvider.java
│
├── application/
│   ├── port/
│   │   ├── in/                                # Empty by convention — see note below
│   │   └── out/
│   │       └── XxxRepository.java             # Outbound port — an interface, no implementation
│   │
│   └── usecase/
│       └── DoSomethingWithXxxUseCase.java     # Coordinates the domain; called directly by adapters
│
├── domain/
│   ├── exception/
│   │   └── XxxNotFoundException.java
│   ├── model/
│   │   └── Xxx.java                           # Framework-free — no Spring, no JPA
│   └── valueobject/
│       └── XxxStatus.java
│
├── infrastructure/
│   ├── config/
│   │   └── SecurityConfig.java
│   ├── exception/
│   └── observability/
│
└── XxxServiceApplication.java                 # Spring Boot bootstrap
```

---

## Concrete example — `security-service`

The structure already built for Security, following the same layout as the reference above:

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
└── SecurityServiceApplication.java        # Spring Boot bootstrap
```

`home-service`, `device-service`, `measurement-service`, `alert-service`, `recommendation-service`,
and `notification-service` mirror this exact package layout, with their own aggregates, use cases,
and ports in place of Security's.

---

## The Ports

Ports are **interfaces** (abstract contracts). The domain and application layers define them;
adapters implement them.

### Driving Port (Input Port) — a deliberate simplification in this project

The textbook version of this pattern puts a separate interface in `application/port/in/` (e.g.
`DoSomethingWithXxxPort`) that the use case class implements, so a controller depends on the
interface rather than the concrete class. **This project does not do that** — as the folder
structure above shows, `application/port/in/` is empty in every service. Controllers inject the
concrete use case class directly:

```java
// application/usecase/DoSomethingWithXxxUseCase.java — this class itself IS the driving port
@Service
public class DoSomethingWithXxxUseCase {
    public XxxResult execute(XxxCommand command) { ... }
}
```

This is a conscious trade-off, not an oversight: with one implementation per use case and no
plan to swap it, the extra interface adds a layer of indirection without a corresponding benefit.
The cost is that a controller technically depends on a concrete class instead of an abstraction —
acceptable here because no service in this project ever substitutes a different implementation of
a use case at runtime. If that ever changes for a specific use case, that specific one gets its
own `port/in` interface; the rest do not need to change to stay consistent.

**Example (`security-service`):** `AuthenticateUserUseCase` is itself the driving port that
`AuthController` depends on directly — no `AuthenticateUserPort` interface exists anywhere in the
codebase.

### Driven Port (Output Port)

Defines what the domain needs from the outside world — without knowing how it is implemented. The
use case is written entirely against these interfaces, never against a concrete JPA repository,
a hashing library, or any other technology.

```java
// application/port/out/XxxRepository.java
public interface XxxRepository {
    Xxx save(Xxx xxx);
    Optional<Xxx> findById(XxxId id);
}
```

One port per capability the domain needs — never one giant port that bundles unrelated
capabilities together.

**Example (`security-service`):** `UserRepository`, `PasswordHasher`, and `TokenProvider` are
three separate ports, not one combined `SecurityGateway` interface:

```java
// application/port/out/UserRepository.java
public interface UserRepository {
    User save(User user);
    Optional<User> findByEmail(Email email);
}

// application/port/out/PasswordHasher.java
public interface PasswordHasher {
    String hash(String rawPassword);
    boolean matches(String rawPassword, String hash);
}

// application/port/out/TokenProvider.java
public interface TokenProvider {
    String issueAccessToken(UserId userId, Set<String> permissionCodes);
}
```

The full outbound-port inventory for Security also includes `AuditRepository`, `EmailSender`,
`LoginErrorLogRepository`, `PasswordPolicyRepository`, `PasswordResetTokenRepository`,
`PermissionRepository`, `RoleRepository`, `SecurityConfigurationRepository`, and
`SessionRepository` — the same one-port-per-capability rule applied twelve times.

---

## The Adapters

### Primary Adapter — how the Controller translates an HTTP request into a domain use case

The controller's only job is translation: HTTP request → domain command, and domain result → HTTP
response. It contains no business rule — if you find an `if` statement enforcing a business rule
inside a controller, that rule belongs in the domain model or the use case, not here.

```java
// adapter/in/web/XxxController.java
@RestController
@RequestMapping("/xxx")
public class XxxController {

    private final DoSomethingWithXxxUseCase useCase;

    public XxxController(DoSomethingWithXxxUseCase useCase) {
        this.useCase = useCase;
    }

    @PostMapping
    public ResponseEntity<XxxResponseDto> handle(@RequestBody @Valid XxxRequestDto request) {
        // 1. Translate the HTTP DTO into a domain command — no business meaning yet, just shape
        XxxCommand command = XxxCommand.from(request);

        // 2. Call the use case — this line is the entire boundary crossing
        XxxResult result = useCase.execute(command);

        // 3. Translate the domain result back into an HTTP DTO
        return ResponseEntity.ok(XxxResponseDto.from(result));
    }
}
```

Domain exceptions propagate up from the use case uncaught by the controller itself — mapping them
to an HTTP status happens once, centrally, in `infrastructure/exception/` (a
`@ControllerAdvice`), not with a `try/catch` block repeated in every controller method. This is
the concrete answer to the open question `05-architecture/cross-cutting.md` raises under "Error
handling": the mapping table that document says does not exist yet is exactly what belongs in
that `@ControllerAdvice` class.

**Example (`security-service`):** `AuthController.login()` follows this exact shape — it builds
an `AuthenticateUserCommand`, calls `AuthenticateUserUseCase.execute(...)`, and returns an
`AuthResponseDto`. It never checks `failedLoginAttempts` or any other rule itself; if
`InvalidCredentialsException` or `AccountBlockedException` is thrown, the controller lets it
propagate to the shared `@ControllerAdvice`.

### Secondary Adapter — implementing the output Repository

The repository adapter implements the driven port. The domain does not know MySQL, JPA, or
Hibernate exist — it only knows the port interface.

```java
// adapter/out/persistence/entity/XxxEntity.java — persistence entity, maps the table 1:1
@Entity
@Table(name = "xxx")
public class XxxEntity {
    @Id
    private String id;
    // fields matching the table columns — no business logic lives here
}

// adapter/out/persistence/repository/XxxJpaRepository.java — Spring Data, framework-facing only
public interface XxxJpaRepository extends JpaRepository<XxxEntity, String> { }

// adapter/out/persistence/mapper/XxxMapper.java — translates in both directions
@Component
public class XxxMapper {
    public Xxx toDomain(XxxEntity entity) { /* Entity → domain model */ }
    public XxxEntity toEntity(Xxx domain) { /* domain model → Entity */ }
}

// adapter/out/persistence/XxxRepositoryAdapter.java — implements the driven port
@Component
public class XxxRepositoryAdapter implements XxxRepository {

    private final XxxJpaRepository jpaRepository;
    private final XxxMapper mapper;

    public XxxRepositoryAdapter(XxxJpaRepository jpaRepository, XxxMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }

    @Override
    public Xxx save(Xxx xxx) {
        return mapper.toDomain(jpaRepository.save(mapper.toEntity(xxx)));
    }
}
```

One `*Entity`, one `*JpaRepository`, one `*Mapper`, one `*RepositoryAdapter` per aggregate — the
pattern does not change per aggregate, only the fields do.

**Example (`security-service`):** `UserRepositoryAdapter` implements `UserRepository` exactly this
way, backed by `UserEntity`, `UserJpaRepository`, and `UserMapper`. The other eleven repository
ports in Security (`SessionRepository`, `PermissionRepository`, ...) each get their own
`*Entity`/`*JpaRepository`/`*Mapper`/`*RepositoryAdapter` set following the identical pattern.

### A category the generic template does not have: `adapter/out/security/`

Not every driven port is a repository. A port that hashes a password or issues a token is still a
secondary adapter — it just is not persistence:

```java
// adapter/out/security/XxxTokenProvider.java
@Component
public class XxxTokenProvider implements TokenProvider {
    // Wraps whatever library actually issues the token. The domain calls
    // TokenProvider.issueAccessToken(...) and never sees that library's types —
    // swapping the library is a change confined entirely to this one class.
}
```

**Example (`security-service`):** `JwtTokenProvider` implements `TokenProvider` by wrapping a JWT
library, and `PasswordHasherImpl` implements `PasswordHasher` by wrapping Spring Security's
`BCryptPasswordEncoder`. This is exactly what the `PasswordHasher` port protects against, per
`06-data/data-dictionary.md`'s note on `user.password_hash`: the concrete hashing algorithm is
swappable without ever touching a use case.

---

## The Use Case (Application Service)

The use case orchestrates the domain. It calls driven ports. It contains no business rule that
belongs to an Aggregate — those rules live in the domain model, not copy-pasted into the use case.

```java
// application/usecase/DoSomethingWithXxxUseCase.java
@Service
public class DoSomethingWithXxxUseCase {

    private final XxxRepository xxxRepository;
    // ...one field per port this operation actually needs

    public DoSomethingWithXxxUseCase(XxxRepository xxxRepository) {
        this.xxxRepository = xxxRepository;
    }

    public XxxResult execute(XxxCommand command) {
        // 1. Load the aggregate (through the port — this class does not know which database it is)
        Xxx xxx = xxxRepository.findById(command.id()).orElseThrow(XxxNotFoundException::new);

        // 2. Business logic lives in the Aggregate, not here
        xxx.doSomething(command);

        // 3. Persist the updated aggregate state
        xxxRepository.save(xxx);

        return XxxResult.from(xxx);
    }
}
```

**Example (`security-service`):** `AuthenticateUserUseCase` follows this shape, coordinating
`UserRepository`, `PasswordHasher`, `TokenProvider`, `SessionRepository`, and
`RegisterLoginErrorUseCase` to authenticate a user — but the business rule that a `BLOCKED`
account cannot authenticate again (`INV-005`) lives on `User` itself, not inside this use case.
The use case only calls `user.registerFailedAttempt()` and reacts to what the aggregate decides.

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

Concretely: `domain/model/Xxx.java` never imports `jakarta.persistence.*`,
`org.springframework.*`, or anything from `adapter/`, in any of the six services. If a domain
class needs an `@Entity` annotation to compile, that class is not in `domain/` — it belongs in
`adapter/out/persistence/entity/`. **Example (`security-service`):** `domain/model/User.java`
imports nothing from Spring or JPA — the `@Entity` annotation lives on `UserEntity` instead, in
`adapter/out/persistence/entity/`.

### Dependency inversion (DI) in practice

```java
// ✓ Correct — the port lives in application/, the adapter implements it
// In application/port/out/:
public interface XxxRepository { ... }

// In adapter/out/persistence/:
@Component
public class XxxRepositoryAdapter implements XxxRepository { ... }

// Spring's IoC container wires them together automatically by type at startup —
// there is no manual composition root to maintain, but the direction of the dependency
// is identical either way: the use case asks for the port (the interface), Spring hands
// it the adapter (the implementation) at runtime.
@Service
public class DoSomethingWithXxxUseCase {
    public DoSomethingWithXxxUseCase(XxxRepository xxxRepository, /* ... */) { ... }
}
```

---

## Advantages for TDD

Hexagonal architecture is ideal for TDD because:

1. **The domain is testable without framework mocks.** No Spring context, no database, no MQTT
   broker needs to start to test a domain model's business logic.

2. **Driven ports can be faked easily.** In tests, an in-memory port implementation replaces the
   real adapter — no `@DataJpaTest`, no test database.

3. **Invariants are explicit** and tested in isolation.

```java
// Domain unit test — zero external dependencies, zero Spring annotations
class XxxTest {

    @Test
    void enforcesItsOwnInvariant() {
        Xxx xxx = Xxx.create(/* ... */);
        assertThrows(SomeDomainException.class, () -> xxx.doSomethingInvalid());
    }
}

// Use case test with a FAKE port (not a real DB, not @DataJpaTest)
class DoSomethingWithXxxUseCaseTest {

    @Test
    void savesTheUpdatedAggregate() {
        InMemoryXxxRepository fakeRepo = new InMemoryXxxRepository();
        fakeRepo.save(existingXxx);
        DoSomethingWithXxxUseCase useCase = new DoSomethingWithXxxUseCase(fakeRepo);

        useCase.execute(new XxxCommand(/* ... */));

        assertEquals(expectedState, fakeRepo.findById(id).get().state());
    }
}
```

**Example (`security-service`):** `UserTest.blockedAccountCannotAuthenticateAgain()` registers
enough failed attempts to trigger `INV-005` and asserts `User` transitions to `BLOCKED` — with no
Spring context and no database. `AuthenticateUserUseCaseTest` then verifies the use case behaves
correctly against an `InMemoryUserRepository` and a stubbed `PasswordHasher`, never a real one.

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
| Business logic in the Controller (e.g. checking a business rule before calling the use case)                     | If the endpoint changes, the business rule silently changes with it, and no other entry point (a future MQTT consumer, a batch job) gets the same rule | Move the rule into the Aggregate as a method, called from the use case                     |
| A `*RepositoryAdapter` returning `XxxEntity` instead of `Xxx` from a method typed as `XxxRepository` | Breaks the contract — the domain would receive a JPA-managed object it cannot validate its own invariants against | The Mapper always converts before returning from the adapter |
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