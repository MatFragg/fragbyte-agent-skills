# Design Patterns & Architectural Patterns

Read this when you're implementing DDD and need to map DDD concepts to well-known design patterns and architectural patterns. Each pattern shows **what problem it solves**, **what breaks when you skip it**, and a **CargoRoute example** to ground it in this skill's domain.

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
- [Pattern-to-concept cross-reference](#pattern-to-concept-cross-reference)

---

## Creation patterns

### Factory Method

- **When:** You need to create a domain object with invariants that must be checked at creation.
- **Benefit:** Decouples creation logic from the constructor; guarantees valid state from the start.
- **Failure mode:** Without it, you can construct an object with empty required fields — invalid state gets persisted.

#### CargoRoute example: `Booking.create()`

A Booking can't exist without a booking number, customer, route, and cargo weight. The `create()` factory method enforces all of these at construction time and registers a domain event:

```
Booking.create(PlaceBookingCommand):
  → generates unique BookingNumber (e.g., "BKG-XXXX")
  → assigns customerId, origin, destination, cargoWeight
  → sets status = PLACED
  → registers BookingPlaced event
  → returns self

Invariants enforced:
  - bookingNumber is never null or empty
  - status starts as PLACED
  - domain event is registered, not published directly
```

This also works on value objects — `BookingNumber.generate()` encapsulates identity generation and format validation so the `"BKG-"` prefix is never accidentally dropped:

```
BookingNumber.generate():
  → creates "BKG-" + random suffix
  → validates format on construction
```

**CargoRoute pattern value:**

| Aspect | CargoRoute |
|---|---|
| Identity generation | `BookingNumber.generate()` (string `BKG-xxxx`) | Ensures valid format from birth |
| Default state | `BookingStatus.PLACED` | Invariants enforced from creation |
| Event on creation | `BookingPlaced` registered | Side effects happen via events, not inline |

---

## Behavioral patterns

### Command

- **When:** You need to represent an action/decision that will be processed later (write side of CQRS).
- **Benefit:** Decouples the invoker from the handler; commands are immutable, serializable, retryable.
- **Failure mode:** Mixing command logic with query logic — changes happen without intent being explicit.

#### CargoRoute example: `PlaceBookingCommand`

A command captures the user's intent to create a booking:

```
[PlaceBookingCommand]  ← immutable input
  - customerId: CustomerId
  - origin: PortCode
  - destination: PortCode
  - cargoWeight: CargoWeight
  - preferredVoyage: VoyageNumber

Validation at construction:
  - customerId is not null
  - origin and destination are not null
```

The controller converts a request to this command, then hands it to a command service.

#### What breaks without it

Without command objects, the controller reaches directly into the aggregate, sets fields from raw request params, and calls `save()`. No validation boundary. No audit trail. No retry. No idempotency key.

---

### Strategy

- **When:** You need multiple interchangeable algorithms for the same operation (routing, feasibility checking, hashing).
- **Benefit:** Swap implementations without touching the calling code.
- **Failure mode:** Hardcoding one algorithm inside a service class — now you can't swap it or test with alternatives.

#### CargoRoute example: `RouteFeasibilityService`

Checking whether cargo can fit on a vessel requires comparing weight, dimensions, route distance, and capacity — spanning `CargoWeight`, `RouteSpecification`, and `VesselCapacity`. No single entity owns this check, so it's a service:

```
[RouteFeasibilityService]  ← interface in domain
  + isFeasible(proposal, cargoWeight): boolean

Implementations:
  [OptimisticFeasibilityService]  → checks weight + basic port connectivity
  [ConservativeFeasibilityService]  → checks weight + 20% buffer + port hours
```

The Booking aggregate receives the service as a parameter to its `confirm()` method — the algorithm is injected, not hardcoded.

#### What breaks without it

Without strategy interfaces, you can't A/B test routing algorithms. You can't swap feasibility rules without a code change + redeploy. Tests need real external services.

---

### Observer

- **When:** You need decoupled reaction to domain events — other parts of the system should react without the originator knowing about them.
- **Benefit:** Adds side effects (emails, projections, analytics) without touching the originating aggregate.
- **Failure mode:** Calling notification logic directly inside an aggregate method — now every save sends an email, even in tests.

#### CargoRoute example: `BookingConfirmedHandler`

When a booking is confirmed, the Tracking context and Notifications context both need to react:

```
[Booking] -- raises --> [BookingConfirmed Event]
                                       ↳ [TrackingProjection]  → updates tracking view
                                       ↳ [NotificationService]  → sends customer email
```

The handler listens for the event and orchestrates the follow-up:

```
BookingConfirmedHandler.on(BookingConfirmed):
  → updates tracking view with route and vessel info
  → triggers notification to customer
  → both happen AFTER the transaction commits
```

#### Why "after commit" matters

The tracking projection must only update if the booking save actually committed. If the transaction rolls back, no event should fire. Event handlers should run only after successful commit — this prevents other contexts from reacting to a booking that doesn't exist.

---

## Structure patterns

### Facade

- **When:** You need to simplify access to a complex subsystem (a bounded context) from the outside.
- **Benefit:** Downstream contexts interact through a clean interface — they don't import your domain types, repositories, or entities.
- **Failure mode:** Other teams import your repository directly. When you change your model, their code breaks.

#### CargoRoute example: `BookingContextFacade`

Port Operations needs to check vessel capacity. Instead of importing Booking's internals, it goes through the facade:

```
[PortOps Context]
  └─ needs to know → [BookingContextFacade]
      
[BookingContextFacade]  ← interface in interfaces/acl
  + isVesselAtCapacity(voyage, cargoWeight): boolean
  + getBookingSummary(bookingNumber): BookingSummary

[BookingContextFacadeImpl]  ← implements facade
  └─ delegates to → [BookingQueryService]
```

The facade returns primitives and simple DTOs — never CargoRoute's domain types.

#### What breaks without it

Without facades, PortOps injects `BookingRepository` directly. When we refactor `Booking` to split cargo details into a separate entity, PortOps breaks. The ACL (via facade) isolates the model change.

---

## Enterprise patterns

### Service Layer

- **When:** You need to centralize application logic — orchestration, transactions, coordination between aggregates.
- **Benefit:** Keeps the domain pure; all use-case entry points go through one layer.
- **Failure mode:** Controllers doing orchestration (`load booking → check route → save → publish`) — now that logic is scattered and untested.

#### CargoRoute example: `BookingCommandServiceImpl`

```
[PlaceBookingCommand]  → [BookingCommandService] (interface)
                          ↑ implements
                  [BookingCommandServiceImpl]  (orchestrator, application layer)
                    
Flow:
  1. Check feasibility via ACL (cross-context call)
  2. Create Booking aggregate from command
  3. Save aggregate (events buffered, not published yet)
  4. Return bookingNumber
  5. Events auto-publish on successful commit
```

| Aspect | CargoRoute |
|---|---|
| Interface location | `domain/services` (port in the domain) | Abstraction before implementation |
| Implementation location | `application/internal/commandservices` | Impl in application layer |
| Transaction boundary | On the orchestrator method | All side effects in one atomic unit |
| Cross-context calls | Via ACL facade | Never reach into another context's internals |

---

### Repository

- **When:** You need to persist or retrieve whole aggregates by identity.
- **Benefit:** The domain never knows how persistence works — the repository interface is a clean contract; any storage technology can satisfy it.
- **Failure mode:** Application services depending on a concrete framework repository — now you can't test without a database, and can't switch databases.

#### CargoRoute example: `BookingRepository`

```
[BookingRepository]  ← interface in domain/services
  + findByBookingNumber(number): Optional<Booking>
  + findConfirmedForVoyage(voyage): List<Booking>
  + save(booking): void

Repository is satisfied by an implementation in outbound infrastructure
(e.g., a Spring Data interface, a JPA repository, a Mongo repository, or an in-memory test impl).
```

| Repository | Aggregate Root | Finders (ubiquitous language) |
|---|---|---|
| `BookingRepository` | `Booking` | `findByBookingNumber`, `findConfirmedForVoyage` |
| `ShipmentRepository` | `Shipment` | `findByVoyageNumber`, `findInTransit` |

#### What breaks without it

Without repository interfaces in the domain, the application layer depends on a persistence-framework type. You can't test command services without a database. You can't switch from MySQL to MongoDB without touching business logic.

---

### Resource / DTO

- **When:** You need to transfer data across the network or between layers without exposing domain internals.
- **Benefit:** The API contract is stable — the domain model can evolve independently.
- **Failure mode:** Returning domain entities directly from API endpoints — internal fields leak to clients, lazy-loading exceptions, JSON shape changes break consumers.

#### CargoRoute example: `BookingResource`

```
[BookingResource]  ← record/DTO in interfaces/rest/resources
  - bookingNumber: String
  - customerEmail: String
  - origin: String
  - destination: String
  - status: String
  - eta: String

Never contains domain-only types (like BookingNumber VO, CargoWeight VO).
Only primitives and simple types for the wire.
```

| Layer | CargoRoute | Purpose |
|---|---|---|
| `interfaces/rest/resources` | `BookingResource`, `PlaceBookingResource` | API contract — JSON shape |
| `domain/model/commands` | `PlaceBookingCommand` | Write intent — domain rules |
| `domain/model/queries` | `GetBookingQuery`, `FindBookingsForVoyageQuery` | Read intent |

#### What breaks without it

If `Booking` entity is returned directly from a REST endpoint, adding a field to hide breaks the API contract. Refactoring the entity's internal structure changes the JSON shape and breaks clients.

---

### Mapper / Assembler

- **When:** You need to translate between domain objects and DTOs/resources.
- **Benefit:** Keeps the translation logic in one place — the assembler — rather than scattered across controllers.
- **Failure mode:** Controllers doing field-by-field mapping manually — now every response is built differently and there's no single place to fix serialization bugs.

#### CargoRoute example: `BookingResourceFromEntityAssembler`

```
[BookingResourceFromEntityAssembler]
  + toResourceFromEntity(booking: Booking): BookingResource
    → booking.getBookingNumber().value() → bookingNumber
    → booking.getOrigin().value()       → origin
    → booking.getStatus().name()        → status
    → booking.getEta()?.toString()      → eta

[PlaceBookingCommandFromResourceAssembler]
  + toCommandFromResource(resource: PlaceBookingResource): PlaceBookingCommand
    → new CustomerId(resource.customerId())
    → new PortCode(resource.origin())
    → new PortCode(resource.destination())
    → new CargoWeight(resource.cargoWeight(), resource.weightUnit())
```

| Assembler | Direction |
|---|---|
| Entity → Resource | `BookingResourceFromEntityAssembler` | Outgoing |
| Resource → Command | `PlaceBookingCommandFromResourceAssembler` | Incoming |

---

### Unit of Work

- **When:** You need to ensure multiple related persistence operations commit atomically.
- **Benefit:** Either all changes persist or none do — no partial states. Events publish only if the transaction commits.
- **Failure mode:** Saving entities without transactional boundaries — an error halfway through leaves the system in an inconsistent state, and events fire even on rollback.

#### CargoRoute example: Transactional command handler

```
@TransactionBoundary on command handler:

[PlaceBookingCommand] → [BookingCommandServiceImpl]
  1. create Booking aggregate   ← in-memory
  2. register BookingPlaced event  ← buffered
  3. save to repository      ← buffered, not flushed yet
  4. commit transaction      ← flushes to DB + publishes events

If anything throws → entire transaction rolls back → no event published
```

| Concern | CargoRoute |
|---|---|
| Transaction scope | All operations in one handler method |
| Rollback behavior | Automatic on errors |
| Read-only queries | Separate service, no writes |
| Event publishing | After commit only — never on rollback |

#### What breaks without it

Without transactional boundaries, saving a `Booking` and then notifying Tracking via a domain event fires the notification even if the save failed and rolled back. Other contexts react to an event for a booking that doesn't exist.

---

## Architectural patterns

### CQRS

- **When:** Read and write needs genuinely diverge — complex queries, different scaling, or read models that span aggregates.
- **Benefit:** Optimize write side for invariants, read side for query performance.
- **Failure mode:** One model trying to serve both — the aggregate gets bloated with query methods, or queries drag in data the invariants don't need.

#### CargoRoute: Booking CQRS

```
Write side:
  [BookingCommandService]
    + handle(PlaceBookingCommand) → BookingNumber
    + handle(CancelBookingCommand) → void

Read side:
  [BookingQueryService]
    + handle(GetBookingQuery) → Optional<Booking>
    + handle(FindBookingsForVoyageQuery) → List<Booking>
```

**CargoRoute uses CQRS in Tracking:**

- **Write side:** Booking aggregate handles `ConfirmBooking`, publishes `BookingConfirmed`
- **Read side:** TrackingProjection listens to `BookingConfirmed`, `CargoLoadedOnVessel`, etc., maintaining a denormalized table: `(bookingNumber, currentPort, status, eta)`

#### When NOT to use CQRS

Many bounded contexts are well served by a single model. Apply CQRS **per bounded context**, where it earns its keep.

---

### Layered Architecture

- **When:** You need to separate concerns so the domain stays pure, testable, and framework-independent.
- **Benefit:** Domain logic can be tested without web frameworks, databases, or HTTP. Each layer has a clear job.
- **Failure mode:** Domain classes coupled to infrastructure frameworks — now you can't test them without the framework, and the domain model is tied to the persistence technology.

#### The four-layer map

| Layer | Responsibility | CargoRoute package |
|---|---|---|
| **Interfaces** | REST controllers, DTOs, assemblers, ACL facade interface | `interfaces/` |
| **Application** | Command/Query services, facades, outbound services | `application/` |
| **Domain** | Entities, value objects, commands, queries, events, domain services | `domain/` |
| **Infrastructure** | Repository implementations, external API clients | `infrastructure/` |

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
│  Infrastructure Layer (Repos, Clients)   │
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
    └── persistence/
        └── repositories         → BookingRepository (framework impl)
```

---

## Pattern decision tree

When you're modeling and don't know which pattern to reach for:

```
Are you creating a domain object that needs invariants?
  → Factory Method (create() with validation + event registration)

Do you need to capture an intent to change as an immutable object?
  → Command (validated record/object passed to a handler)

Do you have multiple interchangeable algorithms for the same thing?
  → Strategy (interface in domain, multiple impls in outbound)

Do other parts of the system need to react to something that happened?
  → Observer (domain event + event handler that runs after commit)

Does another bounded context need to call into this one?
  → Facade (interface in interfaces/acl, impl in application/acl)

Do you have cross-aggregate coordination?
  → Service Layer (transactional orchestrator)
  Is it a read?
    → Query Service (read-only)
  Is it a write?
    → Command Service (transactional, holds no business rules)

Do you need to persist/retrieve an aggregate by identity?
  → Repository (interface in domain, impl in infrastructure)

Are you transferring data over the network without exposing domain internals?
  → Resource/DTO (in interfaces layer)

Do you need to translate between domain objects and DTOs?
  → Mapper/Assembler (in interfaces/transform)

Do you need atomicity across multiple operations?
  → Unit of Work (transactional boundary around the use case)

Do reads and writes have genuinely different needs?
  → CQRS (separate command/query interfaces + potentially separate models)

Are technical concerns (web, persistence, external services) leaking into the domain?
  → Layered Architecture (enforce dependency direction inward)
```

---

## Pattern-to-concept cross-reference

### DDD building blocks → patterns → examples

| DDD concept | Pattern(s) | CargoRoute example |
|---|---|---|
| Aggregate root with behavior | Factory Method | `Booking.create()`, `Booking.confirm()`, `Booking.cancel()` |
| Value object identity | Factory Method | `BookingNumber.generate()`, `BookingNumber.of("BKG-123")` |
| Intent to change state | Command | `PlaceBookingCommand`, `CancelBookingCommand` |
| Intent to read state | Query | `GetBookingQuery`, `FindBookingsForVoyageQuery` |
| Multiple algorithms | Strategy | `RouteFeasibilityService` (optimistic vs. conservative) |
| Something happened | Observer (Domain Event) | `BookingConfirmed` → `BookingConfirmedHandler` |
| Cross-context access | Facade (ACL) | `BookingContextFacade` consumed by PortOps |
| Cross-aggregate coordination | Service Layer | `BookingCommandServiceImpl` orchestrates create + feasibility + save |
| Persist/retrieve aggregate | Repository | `BookingRepository.findByBookingNumber()` |
| Network transfer | Resource/DTO | `BookingResource`, `PlaceBookingResource` |
| Translate between layers | Mapper/Assembler | `BookingResourceFromEntityAssembler` |
| Atomic multi-step operation | Unit of Work | Transactional boundary on the orchestrator |
| Read ≠ Write needs | CQRS | `BookingCommandService` + `BookingQueryService` |
| Isolate domain | Layered Architecture | Four-layer package structure |

### Layer ownership

| Layer | Owns | Depends on |
|---|---|---|
| **Interfaces** | Controllers, DTOs, assemblers, facade interfaces | Application (ports) |
| **Application** | Command/Query service implementations, facades | Domain (ports) + Infrastructure (impls) |
| **Domain** | Entities, value objects, commands, queries, events, service interfaces, exceptions | Nothing |
| **Infrastructure** | Repository implementations, external API clients | Domain (ports) |