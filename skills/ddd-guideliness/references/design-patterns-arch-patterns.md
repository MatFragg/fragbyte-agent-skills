# Design Patterns & Architectural Patterns

Read this when you're implementing DDD and need to map DDD concepts to well-known design patterns and architectural patterns. Each pattern shows **what problem it solves**, **what breaks when you skip it**, and a **real-world example** (from CargoRoute, with notes on how Glottia implements the same pattern).

> **Note:** The Glottia examples come from the Hampcoders platform and are provided here as referential concepts. The CargoRoute column shows how the same pattern applies to this skill's domain.

## Contents

- [Creation patterns](#creation-patterns)
  - [Factory Method](#factory-method)
- [Behavioral patterns](#behavioral-patterns)
  - [Command](#command)
  - [Strategy](#strategy)
  - [Observer](#observer)
- [Structure patterns](#structure-patterns)
  - [Facade](#facade)
- [Enterprise patterns](#enterprise-patterns)
  - [Service Layer](#service-layer)
  - [Repository](#repository)
  - [Resource / DTO](#resource--dto)
  - [Mapper / Assembler](#mapper--assembler)
  - [Unit of Work](#unit-of-work)
- [Architectural patterns](#architectural-patterns)
  - [CQRS](#cqrs)
  - [Layered Architecture](#layered-architecture)
- [Pattern decision tree](#pattern-decision-tree)
- [Pattern-to-code cross-reference](#pattern-to-code-cross-reference)

---

## Creation patterns

### Factory Method

- **When:** You need to create a domain object with invariants that must be checked at creation.
- **Benefit:** Decouples creation logic from the constructor; guarantees valid state from the start.
- **Failure mode:** Without it, `new User("", "")` bypasses validation — invalid state gets persisted.

#### CargoRoute example: `Booking.create()`

```java
@Entity
public class Booking extends AuditableAbstractAggregateRoot<Booking> {
    @Embedded private BookingNumber bookingNumber;
    @Embedded private CustomerId customerId;
    @Embedded private PortCode origin;
    @Embedded private PortCode destination;
    @Enumerated(EnumType.STRING) private BookingStatus status;

    public static Booking create(PlaceBookingCommand command) {
        var booking = new Booking();
        booking.bookingNumber = BookingNumber.generate();
        booking.customerId = command.customerId();
        booking.origin = command.origin();
        booking.destination = command.destination();
        booking.status = BookingStatus.PLACED;
        booking.registerEvent(new BookingPlaced(booking.bookingNumber, booking.customerId));
        return booking;
    }

    private Booking() { }  // JPA
}
```

#### Glottia equivalent: `User.create()` and `UserId.newUserId()`

```java
public static User create(String email, String password) {
    return new User(email, password);  // private constructor + validateInvariants()
}
```

```java
@Embeddable
public record UserId(
    @Column(name = "id") @JsonValue String value
) implements Serializable {

    /**
     * Creates a new unique UserId.
     *
     * @return a new UserId with a "us-" prefix
     */
    public static UserId newUserId() {
        return new UserId("us-" + UUID.randomUUID());
    }

    /**
     * Compact constructor that validates the value format.
     *
     * @param value the user ID string (must start with "us-")
     */
    public UserId {
        if (value == null || !value.startsWith("us-")) {
            throw new IllegalArgumentException("Invalid UserId: " + value);
        }
    }
}
```

This shows **Factory Method** on a *value object* too, not just aggregates — the `newUserId()` static method encapsulates identity generation and format enforcement, so `"us-"` is never accidentally dropped.

| Aspect | CargoRoute | Glottia | Pattern value |
|---|---|---|---|
| Identity generation | `BookingNumber.generate()` (string `BKG-xxxx`) | `UserId.newUserId()` (`"us-" + UUID`) | Type-safe, non-reusable, format-validated |
| Default state | `BookingStatus.PLACED` | `AccessRole.USER` | Invariants enforced from birth |
| Event on creation | `BookingPlaced` | `UserRegisteredEvent` | Side effects happen via events, not inline |

---

## Behavioral patterns

### Command

- **When:** You need to represent an action/decision that will be processed later (write side of CQRS).
- **Benefit:** Decouples the invoker from the handler; commands are immutable, serializable, retryable.
- **Failure mode:** Mixing command logic with query logic — changes happen without intent being explicit.

#### CargoRoute example: `PlaceBookingCommand`

```java
// domain/model/commands/PlaceBookingCommand.java
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
```

#### Glottia equivalent: `SignUpCommand`

```java
public record SignUpCommand(String email, String password) {}
```

| Aspect | CargoRoute | Glottia | Notes |
|---|---|---|---|
| Command type | `PlaceBookingCommand`, `CancelBookingCommand` | `SignUpCommand`, `SignInCommand` | Both use Java records for immutability |
| Handler interface | `BookingCommandService` | `UserCommandService` | Interface in domain, impl in application |
| Controller entry | `POST /api/v1/bookings` | `POST /api/v1/authentication/sign-up` | Controllers are thin — resource → command → handler |

#### What breaks without it

Without command objects, the controller reaches directly into the aggregate, sets fields from raw request params, and calls `save()`. No validation boundary. No audit trail. No retry. No idempotency key.

---

### Strategy

- **When:** You need multiple interchangeable algorithms for the same operation (hashing, routing, pricing).
- **Benefit:** Swap implementations without touching the calling code — e.g., switch from BCrypt to Argon2 hashing.
- **Failure mode:** Hardcoding `new BCryptEncoder()` inside the service class — now you can't swap it, can't test without real hashing.

#### CargoRoute example: `RouteFeasibilityService`

```java
// Outbound Strategy interface (application)
public interface RouteFeasibilityService {
    boolean isFeasible(RouteProposal proposal, CargoWeight weight);
}

// Outbound Strategy impl (infrastructure)
@Service
public class OptimisticFeasibilityService implements RouteFeasibilityService {
    // checks weight + basic port connectivity
}

// Outbound Strategy impl (infrastructure)
@Service
public class ConservativeFeasibilityService implements RouteFeasibilityService {
    // checks weight + 20% buffer + port hours
}
```

#### Glottia equivalent: `HashingService` / `TokenService`

```java
// Outbound Strategy interface (application)
public interface HashingService {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}

// Outbound Strategy impl (infrastructure)
@Service
public class HashingServiceImpl implements BcryptHashingService {
    private final BCryptPasswordEncoder passwordEncoder;
    // ...
}
```

| Strategy interface | CargoRoute | Glottia |
|---|---|---|
| **Hashing** | — (uses Spring Security BCrypt internally) | `HashingService` → `HashingServiceImpl` (BCrypt) |
| **Token** | — (JWT in infrastructure) | `TokenService` → `TokenServiceImpl` (JWT) |
| **Feasibility** | `RouteFeasibilityService` → `OptimisticFeasibilityService` | — |
| **Routing** | `RouteOptimizationService` → `FastRouteService` / `CheapRouteService` | — |

#### What breaks without it

Without strategy interfaces, you can't A/B test routing algorithms. You can't swap hashing without a code change + redeploy. Tests need real external services.

---

### Observer

- **When:** You need decoupled reaction to domain events — other parts of the system should react without the originator knowing about them.
- **Benefit:** Adds side effects (emails, projections, analytics) without touching the originating aggregate.
- **Failure mode:** Calling `emailService.send()` directly inside an aggregate method — now every save sends an email, even in tests.

#### CargoRoute example: `BookingConfirmedHandler`

```java
// application/internal/eventhandlers/BookingConfirmedHandler.java
@Service
public class BookingConfirmedHandler {
    private final TrackingProjection trackingProjection;
    private final NotificationService notifications;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, fallbackExecution = true)
    public void on(BookingConfirmed event) {
        // Update the customer-facing tracking view
        trackingProjection.handle(event);

        // Notify the customer
        notifications.sendBookingConfirmed(
            event.bookingNumber(),
            event.customerId()
        );
    }
}
```

#### Glottia equivalent: `UserRegisteredEventHandler`

```java
// profiles/application/internal/eventhandlers/UserRegisteredEventHandler.java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, fallbackExecution = true)
public void on(UserRegisteredEvent event) {
    if (!"USER".equals(event.accessRole())) { return; }
    var command = new CreateProfileFromRegistrationCommand(event.userId().value());
    profileCommandService.handle(command);
}
```

| Event | CargoRoute | Glottia | Handlers |
|---|---|---|---|
| `BookingConfirmed` | Tracking updates, customer notified | — (not yet) | `BookingConfirmedHandler` |
| `CargoLoadedOnVessel` | Tracking updates, billing starts | — | `CargoLoadedHandler` |
| `BookingCanceled` | Refund initiated | — | `BookingCanceledHandler` |
| `UserRegistered` | — | Profile created | `UserRegisteredEventHandler` |
| `EncounterCompleted` | Analytics updated | Analytics updated | `EncounterCompletedEventHandler` |

**Why `AFTER_COMMIT` matters:** The tracking projection must only update if the booking save actually committed. If the transaction rolls back, no event should fire. `@TransactionalEventListener(phase = AFTER_COMMIT)` guarantees this.

---

## Structure patterns

### Facade

- **When:** You need to simplify access to a complex subsystem (a bounded context) from the outside.
- **Benefit:** Downstream contexts interact through a clean interface — they don't import your domain types, repositories, or entities.
- **Failure mode:** Other teams import your `BookingRepository` and `BookingStatus` enum directly. When you change your model, their code breaks.

#### CargoRoute example: `BookingContextFacade`

```java
// interfaces/acl/BookingContextFacade.java
public interface BookingContextFacade {
    boolean isVesselAtCapacity(VoyageNumber voyageNumber, CargoWeight additionalWeight);
    BookingSummary getBookingSummary(BookingNumber bookingNumber);
}

// application/acl/BookingContextFacadeImpl.java
@Service
public class BookingContextFacadeImpl implements BookingContextFacade {
    private final BookingQueryService queryService;

    @Override
    public boolean isVesselAtCapacity(VoyageNumber voyage, CargoWeight weight) {
        return queryService.handle(new CheckVesselCapacityQuery(voyage, weight));
    }

    @Override
    public BookingSummary getBookingSummary(BookingNumber bookingNumber) {
        return queryService.handle(new GetBookingSummaryQuery(bookingNumber));
    }
}
```

#### Glottia equivalent: `IamContextFacade`

```java
// interfaces/acl/IamContextFacade.java
public interface IamContextFacade {
    Optional<String> createUser(String email, String password);
    Optional<String> fetchUserIdByEmail(String email);
    Optional<String> fetchEmailByUserId(String userId);
}
```

| Facade interface | CargoRoute | Glottia | Consumers |
|---|---|---|---|
| **Booking** | `BookingContextFacade` | — | Routing, PortOps, Billing |
| **Tracking** | `TrackingContextFacade` | — | Notifications, Customer Portal |
| **IAM** | — | `IamContextFacade` | Profiles, Encounters, Engagement |
| **Profiles** | — | `ProfilesContextFacade` | Encounters, Venues, Subscriptions |

#### What breaks without it

Without facades, PortOps injects `BookingRepository` directly. When we refactor `Booking` to split cargo details into a separate entity, PortOps breaks. The ACL (via facade) isolates the model change.

---

## Enterprise patterns

### Service Layer

- **When:** You need to centralize application logic — orchestration, transactions, coordination between aggregates.
- **Benefit:** Keeps the domain pure; all use-case entry points go through one layer.
- **Failure mode:** Controllers doing orchestration (`load booking → check route → save → publish`) — now that logic is scattered and untested.

#### CargoRoute example: `BookingCommandServiceImpl`

```java
// application/internal/commandservices/BookingCommandServiceImpl.java
@Service
@Transactional
public class BookingCommandServiceImpl implements BookingCommandService {
    private final BookingRepository bookings;
    private final RouteFeasibilityService feasibility;
    private final BookingContextFacade bookingFacade;

    @Override
    @Transactional
    public BookingNumber handle(PlaceBookingCommand command) {
        // 1. Validate feasibility (calls out to Routing context via ACL)
        if (!bookingFacade.isVesselAtCapacity(command.preferredVoyage(), command.cargoWeight())) {
            throw new VesselFullException(command.preferredVoyage());
        }

        // 2. Create the aggregate (encapsulates invariants)
        var booking = Booking.create(command);

        // 3. Persist (events are buffered, not published yet)
        bookings.save(booking);

        // 4. Domain events auto-publish on commit by Spring Data
        return booking.getBookingNumber();
    }
}
```

#### Glottia equivalent: `UserCommandServiceImpl`

```java
@Transactional
public Optional<ImmutablePair<User, String>> handle(SignUpCommand command) {
    if (userRepository.existsByEmail(command.email())) {
        throw new EmailAlreadyExistsException("Email already exists");
    }
    var hashedPassword = hashingService.encode(command.password());
    var user = User.create(command.email(), hashedPassword);
    userRepository.save(user);
    // ... generate token
}
```

| Aspect | CargoRoute | Glottia | Key insight |
|---|---|---|---|
| Interface location | `domain/services` | `domain/services` | Port is in the domain |
| Implementation location | `application/internal/...` | `application/internal/...` | Impl is in application |
| Transaction boundary | `@Transactional` on handler | `@Transactional` on handler | All side effects in one txn |
| Cross-context calls | Via `bookingFacade` (ACL) | Via `externalProfilesService` | Never reach into another context's internals |

---

### Repository

- **When:** You need to persist or retrieve whole aggregates by identity.
- **Benefit:** The domain never knows how persistence works — JPA, Mongo, in-memory, it doesn't care.
- **Failure mode:** Controllers calling `entityManager.persist()` directly — now the domain model leaks into persistence concerns, and tests need a database.

#### CargoRoute example: `BookingRepository`

```java
// domain/repositories/BookingRepository.java
public interface BookingRepository {
    Optional<Booking> findByBookingNumber(BookingNumber bookingNumber);
    List<Booking> findByCustomerId(CustomerId customerId);
    List<Booking> findConfirmedForVoyage(VoyageNumber voyageNumber);
    Booking save(Booking booking);
}
```

```java
// infrastructure/persistence/jpa/repositories/JpaBookingRepository.java
@Repository
public interface JpaBookingRepository extends JpaRepository<Booking, Long>, BookingRepository {
    // Spring Data generates the implementation for findByBookingNumber, etc.
}
```

#### Glottia equivalent: `UserRepository`

```java
@Repository
public interface UserRepository extends JpaRepository<User, UserId> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

| Repository | Aggregate Root | Finders (ubiquitous language) |
|---|---|---|
| `BookingRepository` | `Booking` | `findByBookingNumber`, `findByCustomerId`, `findConfirmedForVoyage` |
| `ShipmentRepository` | `Shipment` | `findByVoyageNumber`, `findInTransit` |
| `UserRepository` | `User` | `findByEmail`, `existsByEmail` |

#### What breaks without it

Without repository interfaces in the domain, the application layer depends on `JpaRepository`. You can't test command services without a database. You can't switch from MySQL to MongoDB without touching business logic.

---

### Resource / DTO

- **When:** You need to transfer data across the network or between layers without exposing domain internals.
- **Benefit:** The API contract is stable — the domain model can evolve independently.
- **Failure mode:** Returning JPA entities from REST endpoints — lazy-loading exceptions, N+1 queries, internal fields leaked to clients.

#### CargoRoute example: `BookingResource`

```java
// interfaces/rest/resources/BookingResource.java
public record BookingResource(
    String bookingNumber,
    String customerEmail,
    String origin,
    String destination,
    String status,
    String eta
) { }
```

#### Glottia equivalent: `UserResource`, `AuthenticatedUserResource`

```java
public record UserResource(String id, String email) {}
public record AuthenticatedUserResource(String id, String email, String token) {}
```

| Layer | CargoRoute | Glottia | Purpose |
|---|---|---|---|
| **interfaces/rest/resources** | `BookingResource`, `PlaceBookingResource` | `UserResource`, `SignUpResource` | API contract — JSON shape |
| **domain/model/commands** | `PlaceBookingCommand` | `SignUpCommand` | Write intent — domain rules |
| **domain/model/queries** | `GetBookingQuery`, `FindBookingsForVoyageQuery` | `GetUserByIdQuery` | Read intent |

#### What breaks without it

If `Booking` entity is returned directly from a REST endpoint, adding `@JsonIgnore` to hide a field breaks the API contract. Refactoring the entity's internal structure (splitting `cargoDetails` into its own entity) changes the JSON shape and breaks clients.

---

### Mapper / Assembler

- **When:** You need to translate between domain objects and DTOs/resources.
- **Benefit:** Keeps the translation logic in one place — the assembler — rather than scattered across controllers.
- **Failure mode:** Controllers doing `customer.getName()` + `order.getAmount()` and building response maps inline — now every response is built differently and there's no single place to fix serialization bugs.

#### CargoRoute example: `BookingResourceFromEntityAssembler`

```java
// interfaces/rest/transform/BookingResourceFromEntityAssembler.java
public class BookingResourceFromEntityAssembler {
    public static BookingResource toResource(Booking booking) {
        return new BookingResource(
            booking.getBookingNumber().value(),
            booking.getCustomerId().value(),
            booking.getOrigin().value(),
            booking.getDestination().value(),
            booking.getStatus().name(),
            booking.getEta() != null ? booking.getEta().toString() : null
        );
    }
}
```

```java
// interfaces/rest/transform/PlaceBookingCommandFromResourceAssembler.java
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

#### Glottia equivalent: `UserResourceFromEntityAssembler`

```java
public static UserResource toResourceFrom(User user) {
    return new UserResource(user.getId().value(), user.getEmail());
}
```

| Assembler | CargoRoute | Glottia | Direction |
|---|---|---|---|
| Entity → Resource | `BookingResourceFromEntityAssembler` | `UserResourceFromEntityAssembler` | Outgoing |
| Resource → Command | `PlaceBookingCommandFromResourceAssembler` | `SignUpCommandFromResourceAssembler` | Incoming |
| Entity → AuthenticatedResource | `AuthenticatedUserResourceFromEntityAssembler` | — | Outgoing (with token) |

---

### Unit of Work

- **When:** You need to ensure multiple related persistence operations commit atomically.
- **Benefit:** Either all changes persist or none do — no partial states.
- **Failure mode:** Saving entities in separate transactions — an error halfway through leaves the system in an inconsistent state.

#### CargoRoute example: `@Transactional` on command handlers

```java
@Service
@Transactional  // ← Unit of Work at the service level
public class BookingCommandServiceImpl implements BookingCommandService {

    @Override
    @Transactional  // ← Unit of Work at the method level
    public BookingNumber handle(PlaceBookingCommand command) {
        var booking = Booking.create(command);
        bookings.save(booking);           // buffered, not flushed yet

        // Events registered via registerEvent() are published on commit
        // If anything above throws, the entire transaction rolls back

        return booking.getBookingNumber();
    }
}
```

#### Glottia equivalent: `@Transactional` on `UserCommandServiceImpl`

```java
@Service
@Transactional
public class UserCommandServiceImpl implements UserCommandService {
    // @Transactional on class + @Transactional on individual methods
}
```

| Concern | CargoRoute | Glottia |
|---|---|---|
| Transaction annotation | `@Transactional` on class + methods | `@Transactional` on class |
| Rollback behavior | Automatic on unchecked exceptions | Automatic on unchecked exceptions |
| Read-only queries | `@Transactional(readOnly = true)` on query service | `@Transactional(readOnly = true)` on query service |
| Event publishing | `@TransactionalEventListener(phase = AFTER_COMMIT)` waits for commit | Same pattern |

#### What breaks without it

Without `@Transactional`, saving a `Booking` and then publishing `BookingConfirmed` via `@EventListener` fires the event even if the save failed and rolled back. Other contexts react to an event for a booking that doesn't exist.

---

## Architectural patterns

### CQRS

- **When:** Read and write needs genuinely diverge — complex queries, different scaling, or read models that span aggregates.
- **Benefit:** Optimize write side for invariants, read side for query performance.
- **Failure mode:** One model trying to serve both — the aggregate gets bloated with query methods, or queries drag in data the invariants don't need.

#### CargoRoute: Booking CQRS

```java
// Write side
public interface BookingCommandService {
    BookingNumber handle(PlaceBookingCommand command);
    void handle(CancelBookingCommand command);
}

// Read side
public interface BookingQueryService {
    Optional<Booking> handle(GetBookingQuery query);
    List<Booking> handle(FindBookingsForVoyageQuery query);
}
```

#### Glottia equivalent: IAM CQRS

```java
// Write side
public interface UserCommandService {
    Optional<ImmutablePair<User, String>> handle(SignUpCommand command);
}

// Read side  
public interface UserQueryService {
    List<User> handle(GetAllUsersQuery query);
    Optional<User> handle(GetUserByIdQuery query);
}
```

| Aspect | CargoRoute | Glottia |
|---|---|---|
| Command interface | `BookingCommandService` | `UserCommandService` |
| Query interface | `BookingQueryService` | `UserQueryService` |
| Read-only controller | — | `UsersController` (`@GetMapping`) |
| Write controller | `BookingsController` (`@PostMapping`) | `AuthenticationController` (`@PostMapping`) |
| Read model | `TrackingProjection` (listens to events) | — (direct query through repository) |

#### When NOT to use CQRS

Many bounded contexts are well served by a single model. Apply CQRS **per bounded context**, where it earns its keep.

---

### Layered Architecture

- **When:** You need to separate concerns so the domain stays pure, testable, and framework-independent.
- **Benefit:** Domain logic can be tested without Spring, databases, or HTTP. Each layer has a clear job.
- **Failure mode:** Domain classes importing `javax.persistence` annotations directly — now you can't test them without JPA, and the domain model is tied to the persistence technology.

#### The four-layer map

| Layer | CargoRoute package | Glottia package | Responsibility |
|---|---|---|---|
| **Interfaces** | `interfaces/` | `interfaces/` | REST controllers, DTOs, assemblers, ACL facade interface |
| **Application** | `application/` | `application/` | Command/Query services, facades, outbound services |
| **Domain** | `domain/` | `domain/` | Entities, value objects, commands, queries, events, domain services |
| **Infrastructure** | `infrastructure/` | `infrastructure/` | JPA repositories, external API clients, Spring Security |

#### Dependency flow

```
┌─────────────────────────────────────────┐
│  Interfaces Layer (REST Controllers)     │
│  → depends on → Application Layer         │
├─────────────────────────────────────────┤
│  Application Layer (Services, Facades)   │
│  → depends on → Domain (ports)           │
│  → depends on → Infrastructure (impls)   │
├─────────────────────────────────────────┤
│  Domain Layer (Entities, VOs, Events)    │
│  → depends on NOTHING                      │
├─────────────────────────────────────────┤
│  Infrastructure Layer (JPA, Clients)     │
│  → implements → Domain ports             │
└─────────────────────────────────────────┘
```

#### CargoRoute: Booking bounded context layer breakdown

```
com.cargoroute.booking/
├── interfaces/
│   ├── rest/
│   │   ├── controllers/       → BookingsController
│   │   ├── resources/         → PlaceBookingResource, BookingResource
│   │   └── transform/         → Assemblers (resource ↔ command/entity)
│   └── acl/                   → BookingContextFacade (interface)
├── application/
│   ├── internal/
│   │   ├── commandservices/    → BookingCommandServiceImpl
│   │   ├── queryservices/      → BookingQueryServiceImpl
│   │   └── eventhandlers/      → BookingConfirmedHandler
│   └── acl/                   → BookingContextFacadeImpl
├── domain/
│   ├── model/
│   │   ├── aggregates/          → Booking (aggregate root)
│   │   ├── valueobjects/        → BookingNumber, PortCode, CargoWeight
│   │   ├── commands/            → PlaceBookingCommand, CancelBookingCommand
│   │   ├── queries/             → GetBookingQuery, FindBookingsForVoyageQuery
│   │   └── events/              → BookingPlaced, BookingConfirmed, BookingCanceled
│   ├── services/                → BookingCommandService, BookingQueryService (ports)
│   └── exceptions/              → BookingNotFoundException, VesselFullException
└── infrastructure/
    └── persistence/jpa/
        └── repositories/        → JpaBookingRepository
```

---

## Pattern decision tree

When you're modeling and don't know which pattern to reach for:

```
Are you creating a domain object that needs invariants?
  → Factory Method (create() with validation + event publishing)

Do you need to capture user intent as an immutable, serializable object?
  → Command (record with compact constructor validation)

Do you have multiple interchangeable algorithms for the same thing?
  → Strategy (interface in domain, impls in outbound)

Do other parts of the system need to react to something that happened?
  → Observer (domain event + @EventListener)

Does another bounded context need to call into this one?
  → Facade (interface in interfaces/acl, impl in application/acl)

Do you have cross-aggregate coordination?
  → Service Layer (@Transactional orchestrator)
    Is it a read?
      → Query Service (read-only)
    Is it a write?
      → Command Service (transactional, holds no business rules)

Do you need to persist/retrieve an aggregate by identity?
  → Repository (interface in domain, Spring Data impl in outbound)

Are you transferring data over the network without exposing domain internals?
  → Resource/DTO (record in interfaces/rest/resources)

Do you need to translate between domain objects and DTOs?
  → Mapper/Assembler (static methods in interfaces/rest/transform)

Do you need atomicity across multiple operations?
  → Unit of Work (@Transactional, AFTER_COMMIT event listeners)

Do reads and writes have genuinely different needs?
  → CQRS (separate command/query interfaces + potentially separate models)

Are technical concerns (web, persistence, external services) leaking into the domain?
  → Layered Architecture (enforce dependency direction)
```

---

## Pattern-to-code cross-reference

### CargoRoute

```
Factory Method:
  - Booking.create()                     → booking/domain/model/aggregates/Booking.java
  - BookingNumber.generate()              → booking/domain/model/valueobjects/BookingNumber.java

Command:
  - PlaceBookingCommand                   → booking/domain/model/commands/PlaceBookingCommand.java
  - CancelBookingCommand                  → booking/domain/model/commands/CancelBookingCommand.java

Strategy:
  - RouteFeasibilityService               → booking/domain/services/RouteFeasibilityService.java
  - OptimisticFeasibilityService          → booking/infrastructure/services/...

Observer:
  - BookingPlaced                         → booking/domain/model/events/BookingPlaced.java
  - BookingConfirmed                      → booking/domain/model/events/BookingConfirmed.java
  - BookingConfirmedHandler               → booking/application/internal/eventhandlers/BookingConfirmedHandler.java

Facade:
  - BookingContextFacade                  → booking/interfaces/acl/BookingContextFacade.java
  - BookingContextFacadeImpl              → booking/application/acl/BookingContextFacadeImpl.java

Service Layer:
  - BookingCommandService (interface)     → booking/domain/services/BookingCommandService.java
  - BookingCommandServiceImpl             → booking/application/internal/commandservices/
  - BookingQueryService (interface)       → booking/domain/services/BookingQueryService.java
  - BookingQueryServiceImpl               → booking/application/internal/queryservices/

Repository:
  - BookingRepository (interface)         → booking/domain/repositories/BookingRepository.java
  - JpaBookingRepository                  → booking/infrastructure/persistence/jpa/repositories/

Resource/DTO:
  - BookingResource                       → booking/interfaces/rest/resources/BookingResource.java
  - PlaceBookingResource                  → booking/interfaces/rest/resources/PlaceBookingResource.java

Mapper/Assembler:
  - BookingResourceFromEntityAssembler    → booking/interfaces/rest/transform/...
  - PlaceBookingCommandFromResourceAssembler → booking/interfaces/rest/transform/...

Unit of Work:
  - @Transactional on BookingCommandServiceImpl → booking/application/internal/commandservices/

CQRS:
  - BookingCommandService                 → write side
  - BookingQueryService                   → read side

Layered Architecture:
  - booking/interfaces/                   → inbound adapters
  - booking/application/                  → service layer
  - booking/domain/                       → domain model
  - booking/infrastructure/               → outbound adapters
```

### Glottia (reference)

```
Factory Method:
  - User.create()                         → iam/domain/model/aggregates/User.java
  - SignUpCommand (record)                → iam/domain/model/commands/SignUpCommand.java

Command:
  - SignUpCommand                         → iam/domain/model/commands/SignUpCommand.java
  - SignInCommand                         → iam/domain/model/commands/SignInCommand.java
  - RefreshTokenCommand                   → iam/domain/model/commands/RefreshTokenCommand.java

Strategy:
  - HashingService                        → iam/application/internal/outboundservices/hashing/
  - HashingServiceImpl (BCrypt)            → iam/infrastructure/hashing/bcrypt/
  - TokenService                          → iam/application/internal/outboundservices/tokens/
  - TokenServiceImpl (JWT)                → iam/infrastructure/tokens/jwt/

Observer:
  - UserRegisteredEvent                   → iam/domain/model/events/UserRegisteredEvent.java
  - UserEmailChangedEvent                 → iam/domain/model/events/UserEmailChangedEvent.java
  - UserPasswordChangedEvent              → iam/domain/model/events/UserPasswordChangedEvent.java
  - UserAccessRoleChangedEvent            → iam/domain/model/events/UserAccessRoleChangedEvent.java
  - UserRegisteredEventHandler            → profiles/application/internal/eventhandlers/

Facade:
  - IamContextFacade                      → iam/interfaces/acl/IamContextFacade.java
  - IamContextFacadeImpl                  → iam/application/acl/IamContextFacadeImpl.java
  - ProfilesContextFacade                 → profiles/interfaces/acl/
  - EncountersContextFacade               → encounters/interfaces/acl/
  - EngagementContextFacade               → engagement/interfaces/acl/
  - NotificationsContextFacade            → notifications/interfaces/acl/
  - PromotionsContextFacade               → promotions/interfaces/acl/
  - SubscriptionsContextFacade            → subscriptions/interfaces/acl/
  - VenuesContextFacade                   → venues/interfaces/acl/
  - VerificationContextFacade             → verification/interfaces/acl/

Service Layer:
  - UserCommandService (interface)        → iam/domain/services/
  - UserCommandServiceImpl                → iam/application/internal/commandservices/
  - UserQueryService (interface)          → iam/domain/services/
  - UserQueryServiceImpl                  → iam/application/internal/queryservices/

Repository:
  - UserRepository                      → iam/infrastructure/persistence/jpa/repositories/

Resource/DTO:
  - UserResource                        → iam/interfaces/rest/resources/
  - SignInResource                      → iam/interfaces/rest/resources/
  - SignUpResource                      → iam/interfaces/rest/resources/
  - AuthenticatedUserResource           → iam/interfaces/rest/resources/
  - RefreshedTokenResource              → iam/interfaces/rest/resources/

Mapper/Assembler:
  - UserResourceFromEntityAssembler     → iam/interfaces/rest/transform/
  - AuthenticatedUserResourceFromEntityAssembler → iam/interfaces/rest/transform/
  - SignUpCommandFromResourceAssembler  → iam/interfaces/rest/transform/
  - SignInCommandFromResourceAssembler  → iam/interfaces/rest/transform/
  - RefreshTokenCommandFromResourceAssembler → iam/interfaces/rest/transform/

Unit of Work:
  - @Transactional on UserCommandServiceImpl → iam/application/internal/commandservices/

CQRS:
  - UserCommandService                  → write side (AuthenticationController)
  - UserQueryService                    → read side (UsersController)

Layered Architecture:
  - interfaces/                         → inbound adapters
  - application/                        → service layer
  - domain/                             → domain model
  - infrastructure/                     → outbound adapters
```

### Pattern mapping: CargoRoute → Glottia

| DDD concept | CargoRoute implementation | Glottia equivalent | Glottia file |
|---|---|---|---|
| Aggregate root | `Booking` (entity + behavior) | `User` | `iam/domain/model/aggregates/User.java` |
| Value object | `BookingNumber`, `PortCode`, `CargoWeight` | `UserId` | `shared/domain/model/valueobjects/UserId.java` |
| Domain event | `BookingPlaced`, `BookingConfirmed` | `UserRegisteredEvent` | `iam/domain/model/events/UserRegisteredEvent.java` |
| Command | `PlaceBookingCommand`, `CancelBookingCommand` | `SignUpCommand` | `iam/domain/model/commands/SignUpCommand.java` |
| Query | `GetBookingQuery`, `FindBookingsForVoyageQuery` | `GetUserByIdQuery`, `GetUserByEmailQuery` | `iam/domain/model/queries/` |
| Domain service | `RouteFeasibilityService` | — (hashing/token are outbound) | `booking/domain/services/` |
| Repository | `BookingRepository` (Spring Data) | `UserRepository` (Spring Data) | `iam/infrastructure/persistence/jpa/` |
| Command service | `BookingCommandServiceImpl` (`@Transactional`) | `UserCommandServiceImpl` (`@Transactional`) | `iam/application/internal/commandservices/` |
| Query service | `BookingQueryServiceImpl` (`readOnly`) | `UserQueryServiceImpl` (`readOnly`) | `iam/application/internal/queryservices/` |
| Facade (ACL) | `BookingContextFacade` | `IamContextFacade` | `iam/interfaces/acl/` |
| Event handler | `BookingConfirmedHandler` (`@EventListener`) | `UserRegisteredEventHandler` (`@EventListener`) | `profiles/application/internal/eventhandlers/` |
| Resource/DTO | `BookingResource` | `UserResource` | `iam/interfaces/rest/resources/` |
| Assembler | `BookingResourceFromEntityAssembler` | `UserResourceFromEntityAssembler` | `iam/interfaces/rest/transform/` |
