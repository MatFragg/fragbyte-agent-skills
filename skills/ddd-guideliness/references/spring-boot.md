# Implementing DDD in Spring Boot

Read this when writing DDD code in **Spring Boot / Java**. It assumes you know *what* the building blocks are (see `SKILL.md` and `references/tactical-patterns.md`); here we show *how to express them idiomatically*, walking through what CargoRoute's Booking service looked like **before** we knew better and **after** the refactor.

Works on Spring Boot 3.x with Java 17+. Persistence annotations live in `jakarta.persistence.*`.

## Contents

- [Package structure](#package-structure)
- [The shared kernel](#the-shared-kernel)
- [Value objects](#value-objects)
- [Aggregate root and entities](#aggregate-root-and-entities)
- [Commands and queries](#commands-and-queries)
- [Command and query services](#command-and-query-services)
- [Repositories](#repositories)
- [Outbound services](#outbound-services)
- [Marker interfaces for Spring DI](#marker-interfaces-for-spring-di)
- [Domain events](#domain-events)
- [Anti-corruption layer](#anti-corruption-layer)
- [Interfaces (REST)](#interfaces-rest)
- [Domain exceptions and error handling](#domain-exceptions-and-error-handling)
- [Identity and persistence: preferences](#identity-and-persistence-preferences)
- [Testing each layer](#testing-each-layer)
- [Common pitfalls](#common-pitfalls)
- [Quick reference](#quick-reference)

## Package structure

Give each **bounded context** its own package, split into four layers, with dependencies pointing inward toward `domain`:

```
com.cargoroute.booking
├── interfaces                         // inbound adaptors — the outside drives the context
│   ├── rest
│   │   ├── controllers                // REST controllers
│   │   ├── resources                  // request/response DTOs (records)
│   │   ├── transform                  // assemblers: resource <-> command / entity
│   │   └── advice                     // optional: BC-scoped exception handler
│   └── acl                            // facade interface this context exposes to other contexts
├── application                        // use-case orchestration (no business rules)
│   ├── acl                            // this context's facade implementation
│   └── internal
│       ├── commandservices            // command service implementations
│       ├── queryservices              // query service implementations
│       ├── eventhandlers              // react to domain events
│       └── outboundservices           // outbound ports (optional — only when context calls out)
│           ├── {concept}/             // technology port per concept (hashing/, tokens/, llm/)
│           │   └── {Concept}Service.java
│           └── acl/
│               └── External{BC}Service.java  // calls another context's facade
├── domain                             // the domain model + its ports (depends on nothing)
│   ├── model
│   │   ├── aggregates
│   │   ├── valueobjects
│   │   ├── commands                   // command types (domain)
│   │   ├── queries                    // query types (domain)
│   │   └── events                     // domain events
│   ├── services                       // command/query service interfaces (ports)
│   └── exceptions                     // domain-specific exceptions
└── infrastructure                     // outbound adaptors — the context reaches out
    ├── persistence
    │   └── jpa
    │       └── repositories           // Spring Data repositories
    └── {technology}/                  // technology-specific adapters
        └── {implementation}/          // e.g., bcrypt/, jwt/, sfs/, anthropic/
            └── services/              // marker interface + impl class
```

- **`interfaces`** — inbound adaptors. REST controllers, listeners, CLI. They translate external input into application calls. No business logic. The `acl/` subpackage holds the facade interface this context publishes for others.
- **`application`** — application services. They orchestrate use cases: load aggregates, invoke behavior, manage transactions. Coordinate, hold no business rules. The `outboundservices/` subpackage holds technology-agnostic port interfaces — concept subpackages (`hashing/`, `tokens/`, `llm/`) for external dependencies, and `acl/` exclusively for `External{BC}Service` classes that call other contexts' facades.
- **`domain`** — the model. Aggregates, value objects, domain events, service *interfaces* (ports), exceptions. Every business rule lives here.
- **`infrastructure`** — outbound adaptors. Repositories, external API clients, framework integrations. Follows the `infrastructure/{technology}/{implementation}/` pattern (e.g., `infrastructure/hashing/bcrypt/`, `infrastructure/authorization/sfs/`).

## The shared kernel

A `shared` package holds the small **shared kernel** every context reuses — base classes, common REST resources, cross-cutting config. Keep it small and stable; it must never hold business rules.

`AuditableAbstractAggregateRoot` is the base for aggregate roots that need a generated id + audit timestamps:

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class AuditableAbstractAggregateRoot<T extends AbstractAggregateRoot<T>>
    extends AbstractAggregateRoot<T> {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @CreatedDate @Column(nullable = false, updatable = false)
    private Date createdAt;
    @LastModifiedDate @Column(nullable = false)
    private Date updatedAt;
}
```

Enable auditing with `@EnableJpaAuditing` on a configuration class. A generic response resource:

```java
public record MessageResource(String message) { }
```

## Value objects

Model value objects as **immutable** types. A Java `record` with validation in its compact constructor is the natural fit. Map them as JPA `@Embeddable`.

### CargoRoute: CargoWeight, PortCode, BookingNumber

```java
@Embeddable
public record CargoWeight(BigDecimal amount, WeightUnit unit) {
    public CargoWeight {
        if (amount == null || unit == null)
            throw new IllegalArgumentException("weight requires amount and unit");
        if (amount.signum() < 0)
            throw new IllegalArgumentException("weight cannot be negative");
    }
    public CargoWeight add(CargoWeight other) {
        if (!unit.equals(other.unit))
            throw new IllegalArgumentException("cannot add different units");
        return new CargoWeight(amount.add(other.amount), unit);
    }
    public CargoWeight() { this(null, null); }  // JPA hydration
}

@Embeddable
public record PortCode(String value) {
    public PortCode {
        if (value == null || value.isBlank())
            throw new IllegalArgumentException("port code required");
    }
}

@Embeddable
public record BookingNumber(String value) {
    public static BookingNumber generate() {
        return new BookingNumber("BKG-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase());
    }
    public BookingNumber {
        if (value == null || !value.matches("BKG-[A-F0-9-]+"))
            throw new IllegalArgumentException("invalid booking number format");
    }
}

// Use value objects for typed identifiers too, so a reference to another aggregate
// stays type-safe and meaningful — never a bare Long
@Embeddable
public record CustomerId(Long value) {
    public CustomerId {
        if (value == null || value < 1) throw new IllegalArgumentException("invalid customer id");
    }
}
```

A record fits most VOs, but use an `@Embeddable` **class** when the VO has to map a JPA association or collection (records are final).

### What this saved us

Using `String origin` and `String destination` everywhere, a developer passed `destination` where `origin` was expected in a route check. The route looked valid — London to Southampton on a vessel leaving from Southampton. The cargo never moved. Typed `PortCode` VOs make this a compile error.

### Test: VO validation
```java
@Test
void rejectsNegativeWeight() {
    assertThatThrownBy(() -> new CargoWeight(
        new BigDecimal("-1"), WeightUnit.TONNES))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test
void rejectsMixingUnits() {
    var tonnes = new CargoWeight(new BigDecimal("10"), WeightUnit.TONNES);
    var kg = new CargoWeight(new BigDecimal("500"), WeightUnit.KILOGRAMS);
    assertThatThrownBy(() -> tonnes.add(kg))
        .isInstanceOf(IllegalArgumentException.class);
}
```

## Aggregate root and entities

The aggregate root is a JPA `@Entity`. Give it real **behavior**, enforce invariants inside it, and **reference other aggregates by their typed id value object** — never `@ManyToOne` to another aggregate root.

### CargoRoute: the naive Booking (before)

```java
@Entity
public class Booking {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;                    // raw Long — swappable with any other Long
    private String customerEmail;       // no validation
    private String originPort;          // raw string — typo-prone
    private String destinationPort;     // easy to mix
    private String status;              // "CONFIRMED"? "CONFIRMD"?
    
    // 10 setters for everything. No behavior.
    // ApplicationService checks: if (status != "CONFIRMED") throw ...
}
```

Every rule lives in a service layer. Every typo in a status string is a production bug. Rules are optional — if you forget to call the service, the invariant is gone.

### After: the idiomatic Booking aggregate

```java
@Entity
public class Booking extends AuditableAbstractAggregateRoot<Booking> {
    @Embedded private BookingNumber bookingNumber;
    @Embedded private CustomerId customerId;
    @Embedded private PortCode origin;
    @Embedded private PortCode destination;
    @Enumerated(EnumType.STRING)
    private BookingStatus status;
    @Embedded private CargoWeight cargoWeight;

    protected Booking() { }  // JPA

    public Booking(PlaceBookingCommand command) {
        this.bookingNumber = new BookingNumber("BKG-" + randomSuffix());
        this.customerId = command.customerId();
        this.origin = command.origin();
        this.destination = command.destination();
        this.cargoWeight = command.cargoWeight();
        this.status = BookingStatus.PLACED;
        registerEvent(new BookingPlaced(bookingNumber, customerId));
    }

    public void confirm(RouteFeasibilityService feasibility, RouteProposal proposal) {
        if (status != BookingStatus.PLACED)
            throw new IllegalStateException("can only confirm a placed booking");
        if (!feasibility.isFeasible(proposal, cargoWeight))
            throw new IllegalStateException("proposed route cannot handle cargo weight");
        status = BookingStatus.CONFIRMED;
        registerEvent(new BookingConfirmed(bookingNumber, proposal.routeId(), proposal.vessel()));
    }

    public void loadCargo() {
        if (status != BookingStatus.CONFIRMED)
            throw new IllegalStateException("cannot load an unconfirmed booking");
        status = BookingStatus.LOADED;
        registerEvent(new CargoLoadedOnVessel(bookingNumber));
    }

    public void cancel(String reason) {
        if (status == BookingStatus.LOADED)
            throw new IllegalStateException("cannot cancel a loaded booking");
        status = BookingStatus.CANCELED;
        registerEvent(new BookingCanceled(bookingNumber, reason));
    }
}
```

### What this fixed

| Before | After | Saved us from |
|---|---|---|
| `Long id` — mixable with any other Long | `BookingNumber` — typed | The $4M ship-to-wrong-port incident |
| `String status` — typo = broken flow | `BookingStatus` enum — compiler-enforced | Three failed deployments |
| Rules in a service layer — bypassable | Rules in the aggregate — unbreakable | The booking canceled after loading |

### Test: the invariant that used to be a service-level if-check
```java
@Test
void cannotCancelAfterCargoLoaded() {
    var booking = new Booking(placeBookingCommand());
    booking.confirm(feasibilityService, routeProposal());
    booking.loadCargo();

    assertThatThrownBy(booking::cancel)
        .isInstanceOf(IllegalStateException.class)
        .hasMessage("cannot cancel a loaded booking");
}
```

## Commands and queries

Make **commands** and **queries** first-class types in the domain, as records that validate their own input:

```java
public record PlaceBookingCommand(
    CustomerId customerId,
    PortCode origin,
    PortCode destination,
    CargoWeight cargoWeight,
    VoyageNumber preferredVoyage
) {
    public PlaceBookingCommand {
        if (customerId == null) throw new IllegalArgumentException("customerId required");
        if (origin == null || destination == null) throw new IllegalArgumentException("route required");
    }
}

public record GetBookingQuery(BookingNumber bookingNumber) { }
public record FindBookingsForVoyageQuery(VoyageNumber voyageNumber) { }
```

## Command and query services

Split the application layer along the command/query line (CQRS — see `tactical-patterns.md`). Declare the **service interfaces in the domain** (`domain/services`) as ports, and **implement them in application**.

### Command service — orchestrates, holds no rules

```java
// domain/services — the port
public interface BookingCommandService {
    BookingNumber handle(PlaceBookingCommand command);
    void handle(CancelBookingCommand command);
}

// application/internal/commandservices — the implementation
@Service
public class BookingCommandServiceImpl implements BookingCommandService {
    private final BookingRepository bookings;

    @Override
    @Transactional
    public BookingNumber handle(PlaceBookingCommand command) {
        var booking = new Booking(command);
        bookings.save(booking);
        return booking.getBookingNumber();
    }

    @Override
    @Transactional
    public void handle(CancelBookingCommand command) {
        var booking = bookings.findByBookingNumber(command.bookingNumber())
            .orElseThrow(() -> new BookingNotFoundException(command.bookingNumber()));
        booking.cancel(command.reason());
        // events auto-published by AbstractAggregateRoot on save
    }
}
```

### Query service — reads, no mutations

```java
// domain/services — the port
public interface BookingQueryService {
    Optional<Booking> handle(GetBookingQuery query);
    List<Booking> handle(FindBookingsForVoyageQuery query);
}

@Service
public class BookingQueryServiceImpl implements BookingQueryService {
    private final BookingRepository bookings;

    @Override
    public Optional<Booking> handle(GetBookingQuery query) {
        return bookings.findByBookingNumber(query.bookingNumber());
    }

    @Override
    public List<Booking> handle(FindBookingsForVoyageQuery query) {
        return bookings.findConfirmedForVoyage(query.voyageNumber());
    }
}
```

## Repositories

One repository per aggregate root. The simplest approach: a Spring Data repository in `infrastructure`:

```java
// infrastructure/persistence/jpa/repositories
public interface BookingRepository extends JpaRepository<Booking, Long>,
                                           BookingQueryOperations {
    Optional<Booking> findByBookingNumber(BookingNumber bookingNumber);
    List<Booking> findConfirmedForVoyage(VoyageNumber voyageNumber);
}
```

`JpaRepository` gives you `save`, `findById`; add finders named in the ubiquitous language.

> **Alternative — a domain port.** To keep the domain and application free of framework dependencies, declare a plain interface in `domain`:
>
> ```java
> public interface BookingRepository {
>     Optional<Booking> findByBookingNumber(BookingNumber number);
>     List<Booking> findConfirmedForVoyage(VoyageNumber voyage);
>     Booking save(Booking booking);
> }
> // infrastructure satisfies it:
> interface JpaBookingRepository extends JpaRepository<Booking, Long>, BookingRepository { }
> ```
>
> Prefer this when isolating the domain matters. The cost is one extra interface.

### When to inject the repository directly

In a modular monolith with DDD, injecting the repository directly (without a domain port) is correct when:

1. **No business rules in the repository.** It only offers standard finders (`findByXxx`, `existsByXxx`) and the inherited `JpaRepository` methods. There is no domain logic to protect.
2. **YAGNI.** Adding a port interface "just in case" you migrate databases later is over-engineering. Introduce it when the migration actually happens.
3. **The `@Entity` IS the aggregate root.** There is no separate domain model and persistence model — JPA annotations and domain behavior live on the same class. The repository is a detail of persistence, not a domain concept.
4. **Spring Data JPA IS the abstraction.** `JpaRepository` already abstracts over Hibernate and the database provider. Adding another layer of indirection does not buy testability — Mockito mocks the repository interface just as easily either way.

**When to add a port:** if the repository starts encapsulating business rules (e.g., complex queries with domain invariants), if the team decides to support multiple databases simultaneously, or if testability demands a domain-level contract.

## Outbound services

When a bounded context calls an external technology (hashing, tokens, payments, LLMs, email), define a **technology-agnostic port interface** in the application layer and implement it in infrastructure.

```text
application/internal/outboundservices/{concept}/     → Port interface
infrastructure/{technology}/{implementation}/        → Marker interface + adapter
```

**Rules:**

1. The port interface must be technology-agnostic — no framework imports.
2. Multiple implementations of the same port are allowed (strategy pattern).
3. The marker interface extends the port and may add technology-specific methods.
4. The actual implementation class has an `Impl` suffix.

### CargoRoute example: hashing service

```text
application/internal/outboundservices/hashing/
  └── HashingService.java                  ← port (interface)

infrastructure/hashing/bcrypt/
  ├── BcryptHashingService.java            ← marker interface
  └── services/
      └── HashingServiceImpl.java          ← implementation
```

```java
// Port — technology-agnostic
public interface HashingService {
    String encode(String rawPassword);
    boolean matches(String rawPassword, String encodedPassword);
}

// Marker interface — extends port + Spring framework interface
public interface BcryptHashingService extends HashingService, PasswordEncoder { }

// Implementation
@Service
public class HashingServiceImpl implements BcryptHashingService {
    private final PasswordEncoder delegate = new BCryptPasswordEncoder();

    @Override
    public String encode(String rawPassword) {
        return delegate.encode(rawPassword);
    }

    @Override
    public boolean matches(String rawPassword, String encodedPassword) {
        return delegate.matches(rawPassword, encodedPassword);
    }
}
```

The application layer depends on `HashingService` (the port). The infrastructure detail (BCrypt, Spring Security's `PasswordEncoder`) stays behind the marker interface. Swapping to Argon2 means adding a new `infrastructure/hashing/argon2/` package — no application code changes.

**When to skip:** if the external dependency is a thin wrapper with no swappability (e.g., a simple utility), injecting it directly is fine. The port adds value when the technology is likely to change or when testing requires a mock boundary.

## Marker interfaces for Spring DI

When a port interface and a Spring framework interface share method names, Spring cannot disambiguate which bean to inject. A **marker interface** — a zero-method interface that extends both — resolves this cleanly.

### The problem

`HashingService` (port) declares `encode` and `matches`. Spring Security's `PasswordEncoder` declares the same methods. If `HashingServiceImpl` implements `HashingService` directly, Spring cannot inject it where `PasswordEncoder` is needed:

```java
// Ambiguous — which bean?
@Bean
public PasswordEncoder passwordEncoder() { ... }
```

### The solution

```java
public interface BcryptHashingService extends HashingService, PasswordEncoder { }
```

This marker interface tells Spring: "this bean serves both roles." `HashingServiceImpl` implements `BcryptHashingService`, and Spring can inject it wherever either `HashingService` or `PasswordEncoder` is requested.

### Why not the alternatives

| Alternative | Problem |
|---|---|
| `@Primary` | Hides the ambiguity instead of solving it — the wrong bean might win silently |
| `@Qualifier("bcrypt")` | Couples consumers to a magic string — now every injection site must know the implementation name |
| `@Qualifier` + constants | More boilerplate than the marker interface, same coupling |

The marker interface is explicit: whoever reads `BcryptHashingService` understands it unifies two contracts. Consumers ask for `HashingService`; the implementation detail stays in infrastructure.

The same pattern applies to any port with a framework-aligned interface — e.g., `BearerTokenService extends TokenService` when the token implementation also satisfies Spring's token contract.

## Domain events

Model each event as a class extending Spring's `ApplicationEvent`, named in past tense:

```java
@Getter
public class BookingConfirmed extends ApplicationEvent {
    private final BookingNumber bookingNumber;
    private final RouteId routeId;
    private final VoyageNumber vessel;

    public BookingConfirmed(BookingNumber bookingNumber, RouteId routeId, VoyageNumber vessel) {
        super(bookingNumber);
        this.bookingNumber = bookingNumber;
        this.routeId = routeId;
        this.vessel = vessel;
    }
}
```

The aggregate raises it via `registerEvent`. Because it extends `AbstractAggregateRoot`, Spring Data **publishes registered events automatically when the aggregate is saved**. Handle them in `application/internal/eventhandlers`:

```java
@Service
public class BookingConfirmedHandler {
    @EventListener
    public void on(BookingConfirmed event) {
        // trigger downstream: notify Tracking, update Route assignment
    }
}
```

Use `@TransactionalEventListener` when the handler must run only after commit (e.g., sending an email that must not fire if the save rolls back).

## Anti-corruption layer

When this context needs something from **another** bounded context, do not import its model. The provider exposes a **facade interface**; the consumer translates the result into its own value objects.

```java
// Vessel Scheduling exposes
public interface VesselSchedulingFacade {
    VoyageNumber findViableVoyage(PortCode origin, PortCode destination, CargoWeight weight);
}

// Booking consumes — translates to own VOs
@Service
public class ExternalVesselService {
    private final VesselSchedulingFacade scheduling;
    public Optional<RouteProposal> proposeRoute(PortCode origin, PortCode destination, CargoWeight weight) {
        var voyage = scheduling.findViableVoyage(origin, destination, weight);
        return voyage == null ? Optional.empty()
            : Optional.of(new RouteProposal(voyage, computeEta(origin, destination)));
    }
}
```

### Outbound vs inbound ACL

- **Outbound (this context consumes):** The consumer defines the facade interface in `interfaces/acl/`. The provider implements it in `application/acl/`. The consumer's `External{BC}Service` (in `application/internal/outboundservices/acl/`) is the only class that injects the facade. All other application code delegates to it.
- **Inbound (this context exposes):** Define the facade interface in `interfaces/acl/`. Implement it in `application/acl/`. Other contexts depend on this interface — never on your repositories, aggregates, or internal services.

### Bidirectional ACL in a monolith

Two bounded contexts can define facades that the other consumes:

```text
Booking → ExternalVesselService → VesselSchedulingFacade
VesselScheduling → ExternalBookingService → BookingContextFacade
```

This bidirectional dependency is acceptable in a monolith when:

- Dependencies are to interfaces, not implementations.
- Each context controls its own facade.
- Communication is synchronous and in-process.

In microservices, these facade interfaces become HTTP/gRPC clients — the interface stays, the adapter changes.

### Cross-context reference data

Some contexts need **read-only data** owned by another context to enrich their responses (e.g., Booking shows the vessel name; Tracking shows the port address). Apply this pattern:

1. **Provider VO:** The provider defines a simple record in `domain/model/valueobjects/` (e.g., `VesselInfo`). If the VO is used by 3+ bounded contexts, place it in `shared/domain/model/valueobjects/`.
2. **Facade returns the VO:** The provider's `XxxContextFacade` returns the VO directly, or returns primitives when only a single field is needed.
3. **Consumer mapping:** The consumer's `ExternalXxxService` (in `application/internal/outboundservices/acl/`) calls the facade. If the consumer needs only a subset of fields, it maps to its own minimal VO in `domain/model/valueobjects/`.
4. **No domain services:** Non-aggregate reference data does NOT get its own service interface in `domain/services/`. The `ExternalXxxService` is an application-layer service, not a domain service.

**Primitives (single field needed):**

```java
// Provider: VesselSchedulingFacade
Optional<String> fetchVesselName(String voyageNumber);

// Consumer: ExternalVesselService
String fetchVesselName(String voyageNumber) {
    return vesselSchedulingFacade.fetchVesselName(voyageNumber).orElse("");
}
```

**Provider VO → Consumer minimal VO (multiple fields needed):**

```java
// Provider: domain/model/valueobjects/VesselInfo.java
public record VesselInfo(String id, String name, String imoNumber, boolean active) {}

// Provider: VesselSchedulingFacade
VesselInfo fetchVesselInfo(String vesselId);

// Consumer: domain/model/valueobjects/VesselBookingInfo.java
public record VesselBookingInfo(String vesselId, String name, String imoNumber) {}

// Consumer: ExternalVesselService
VesselBookingInfo fetchVesselBookingInfo(String vesselId) {
    VesselInfo info = vesselSchedulingFacade.fetchVesselInfo(vesselId);
    return new VesselBookingInfo(info.id(), info.name(), info.imoNumber());
}
```

**Shared VO (used by 3+ contexts):**

```java
// shared/domain/model/valueobjects/VoyageContext.java
public record VoyageContext(String voyageNumber, String vesselId, String routeId) {}

// Provider: VesselSchedulingFacade
VoyageContext fetchVoyageContext(String voyageNumber);

// Consumer: ExternalVesselService
VoyageContext fetchVoyageContext(String voyageNumber) {
    return vesselSchedulingFacade.fetchVoyageContext(voyageNumber);
}
```

This keeps a single point of access per external context. When moving to microservices, the `ExternalXxxService` becomes an HTTP client while the VOs stay unchanged.

## Interfaces (REST)

Three parts: **`resources`** (DTOs), **`transform`** (assemblers), and the **controller**.

**Resources** — request/response DTOs, no domain types:

```java
// interfaces/rest/resources
public record PlaceBookingResource(
    @NotNull Long customerId,
    @NotBlank String origin,
    @NotBlank String destination,
    @NotNull BigDecimal cargoWeight,
    @NotBlank String weightUnit
) { }

public record BookingResource(
    String bookingNumber, String status, String origin, String destination
) { }
```

**Transformers** — static assemblers, resource ↔ command/entity:

```java
// interfaces/rest/transform
public class PlaceBookingCommandFromResourceAssembler {
    public static PlaceBookingCommand toCommand(PlaceBookingResource resource) {
        return new PlaceBookingCommand(
            new CustomerId(resource.customerId()),
            new PortCode(resource.origin()),
            new PortCode(resource.destination()),
            new CargoWeight(resource.cargoWeight(), WeightUnit.valueOf(resource.weightUnit()))
        );
    }
}
```

**The controller** — thin orchestrator. Resource → assembler → command → service → id → query → entity → assembler → resource:

```java
// interfaces/rest/controllers
@RestController
@RequestMapping("/api/v1/bookings")
class BookingsController {
    private final BookingCommandService commands;
    private final BookingQueryService queries;

    @PostMapping
    ResponseEntity<BookingResource> placeBooking(@RequestBody PlaceBookingResource resource) {
        var command = PlaceBookingCommandFromResourceAssembler.toCommand(resource);
        var bookingNumber = commands.handle(command);
        return queries.handle(new GetBookingQuery(bookingNumber))
            .map(b -> ResponseEntity.created(URI.create("/api/v1/bookings/" + b.getBookingNumber().value()))
                   .body(BookingResourceFromEntityAssembler.toResource(b)))
            .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @GetMapping("/{number}")
    ResponseEntity<BookingResource> getBooking(@PathVariable String number) {
        return queries.handle(new GetBookingQuery(new BookingNumber(number)))
            .map(b -> ResponseEntity.ok(BookingResourceFromEntityAssembler.toResource(b)))
            .orElseGet(() -> ResponseEntity.notFound().build());
    }
}
```

## Domain exceptions and error handling

Express failures in the **ubiquitous language**. Define domain-specific exceptions in `domain/exceptions`:

```java
// domain/exceptions
public class BookingNotFoundException extends RuntimeException {
    public BookingNotFoundException(BookingNumber number) {
        super("Booking " + number.value() + " not found");
    }
}
```

No HTTP status codes inside them — keeps the domain pure. Translate at the edge:

```java
// interfaces/rest/advice
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(BookingNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    ErrorResponse handle(BookingNotFoundException ex) {
        return ErrorResponse.create(ex, HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

### BC-scoped exception handling

A **global** handler in `shared/` maps common exceptions (`ResourceNotFoundException`, `ForbiddenException`, `IllegalArgumentException`). A **BC-specific** handler maps exceptions unique to that context:

```java
// booking/interfaces/rest/advice/BookingExceptionHandler.java
@RestControllerAdvice(basePackages = "com.cargoroute.booking.interfaces.rest")
class BookingExceptionHandler {
    @ExceptionHandler(VesselFullException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    ErrorResponse handle(VesselFullException ex) {
        return ErrorResponse.create(ex, HttpStatus.CONFLICT, ex.getMessage());
    }
}
```

**Rules:**

- Never redefine shared kernel exceptions in a bounded context.
- BC-scoped handlers import from `shared.domain.exceptions`.
- Only create BC-specific exceptions when the semantic meaning differs from existing shared exceptions.

## Identity and persistence: preferences

- **Identity.** Three approaches, in order of recommendation:
  1. **Typed id VO with application-side generation** — `BookingNumber.generate()` produces `"BKG-" + UUID`, or Glottia's `UserId.newUserId()` produces `"us-" + UUID.randomUUID()`, both at construction time, mapped as `@EmbeddedId`. Keeps identity type-safe *and* avoids a database round-trip to retrieve the generated key.
  2. **Typed id VO as `@EmbeddedId`** — if the ID is externally assigned (e.g., a booking reference the customer chooses), use a typed VO that validates format at construction time. No `@GeneratedValue` needed.
  3. **Surrogate `Long` with `@GeneratedValue`** — the shared kernel's `AuditableAbstractAggregateRoot` approach. Use this when you need auto-increment IDs, but **always** use typed VOs for cross-aggregate references (`CustomerId`, not bare `Long`).
- **Repository.** Spring Data directly is simplest; declare a domain port when isolating the domain matters.
- **JPA in the domain.** Annotating entities with JPA is fine for most projects. The cost is a soft dependency on the framework. For maximum isolation, keep the domain as plain Java and map to a persistence model in `infrastructure`.

Non-negotiable: business rules and invariants stay in the domain, and the domain never depends on `interfaces` or `application`.

## Testing each layer

Match the test style to what the layer actually does:

- **`domain`** — plain JUnit unit tests, no Spring context. Construct the aggregate, call behavior, assert on state/exceptions/raised events. Should run in milliseconds.
- **`application`** — plain unit tests with a mocked repository (verify the right method was called), or `@DataJpaTest` when you want to exercise persistence.
- **`infrastructure`** — `@DataJpaTest` against an in-memory database: confirm mappings, typed-id handling, and finders.
- **`interfaces`** — `@WebMvcTest` (controller sliced, services mocked) or `@SpringBootTest` with `MockMvc`.

A domain test that needs `@SpringBootTest` to pass is usually a sign business logic leaked into a Spring-managed component.

## Common pitfalls

- **`@ManyToOne` to another aggregate root.** Reintroduces "reach into another aggregate's object graph." Use a typed id VO instead.
- **Business rules inside `@Service` command classes.** If a command service contains an `if` deciding whether an operation is *allowed*, it belongs inside the aggregate.
- **Publishing domain events manually instead of via `registerEvent` + save.** Bypasses the transactional guarantee — events publish only if the save succeeds.
- **`@EventListener` for handlers that call external systems.** Fires before commit; use `@TransactionalEventListener` for anything with an external side effect.
- **Skipping the ACL** by injecting another context's repository directly. This is exactly what the anti-corruption layer exists to prevent.
- **Fat controllers** that build responses by hand from multiple service calls with conditional logic.

## Quick reference

| DDD concept | Spring Boot idiom |
|---|---|
| Entity / Aggregate Root | `@Entity` class, behavior methods, `AbstractAggregateRoot` |
| Value Object | `record`, `@Embeddable`, validated in compact constructor |
| Typed identifier | `record` `@Embeddable` wrapping the raw id |
| Repository | Spring Data interface extending `JpaRepository<Agg, Id>` |
| Domain Event | Class extending `ApplicationEvent`, raised via `registerEvent` |
| Event handler | `@Service` with `@EventListener` / `@TransactionalEventListener` |
| Command / Query | `record` in `domain/model/commands` or `queries` |
| Command/Query Service | Interface in `domain/services`, impl in `application/internal/...` |
| Anti-corruption layer | Facade interface (`interfaces/acl`) + outbound service (`outboundservices/acl/External{BC}Service`) |
| Outbound service (tech port) | Interface in `outboundservices/{concept}/`, adapter in `infrastructure/{tech}/{impl}/` |
| Marker interface | Extends port + framework interface (e.g., `BcryptHashingService extends HashingService, PasswordEncoder`) |
| Domain exception | `RuntimeException` subclass in `domain/exceptions`, mapped by `@RestControllerAdvice` |
| BC-scoped exception handler | `@RestControllerAdvice(basePackages = "...interfaces.rest")` in `{context}/interfaces/rest/advice/` |
