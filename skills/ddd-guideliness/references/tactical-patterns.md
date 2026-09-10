# Tactical Patterns

Read this when modeling *inside* a bounded context: choosing the building blocks that make up the domain model and the rules for combining them well. These are the tools that keep business logic in the model instead of leaking into controllers, services, or SQL — the cure for the **anemic domain model**.

Do strategic design first (`references/strategic-design.md` and `references/domain-modeling.md`); these patterns live inside the boundaries you found there. For how to express them in a specific stack, see the stack reference (e.g., `references/spring-boot.md`).

## Model-driven design, not Smart UI

The building blocks below only pay off under **model-driven design**: the domain is expressed as a rich model of objects with behavior, isolated by the layered architecture. The opposite choice is **Smart UI** — putting business logic directly in the user interface or in procedural scripts. The two are mutually exclusive. Commit to the model.

The failure mode to watch for is the **anemic domain model**: objects that are just bags of getters and setters while all the behavior lives in "service" classes around them. It looks object-oriented but is procedural.

### CargoRoute lesson: the Booking that could be canceled after loading

We shipped this. A `Booking` entity had `getStatus()` and `setStatus()`. An `ApplicationService` checked whether the booking was canceled before processing. The check passed because the status was set in a different service call. The cargo was loaded anyway.

Fix: move `cancel()` into the `Booking` aggregate. Now canceling after `CargoLoaded` is impossible by construction.

## How the building blocks fit together

- **Entities** and **value objects** cluster into **aggregates**; an entity usually **acts as the aggregate root**.
- **Aggregates** are stored and retrieved through **repositories** (one per aggregate root).
- Complex creation is encapsulated in **factories**.
- **Domain events** announce facts that other aggregates or contexts may react to.
- **Domain services** hold logic that doesn't belong to a single entity.

## Entity

An **entity** is an object with a distinct identity that persists through changes to its attributes. What makes a Booking the same Booking is its `BookingNumber`, not whether the vessel or destination has changed.

### Decision: Entity vs. Value Object (CargoRoute edition)

Ask three questions:

1. **Does this concept have an identity you track over time?** A `Booking` yes — it exists for months. A `CargoWeight` no — 42 tons is 42 tons.
2. **Do two instances with the same attributes need to be distinguishable?** Two bookings with identical cargo but different customers — yes, they're different bookings.
3. **Does this span a lifecycle with distinct states?** Bookings go Placed → Confirmed → Loaded → Delivered. Cargo weight doesn't.

If you answered yes to any, you have an Entity.

### CargoRoute invariant checklist for entities

- [ ] Equality is by identity, not attributes — two `Booking` objects with the same fields are equal only if they share `BookingNumber`
- [ ] Identity is assigned once and never reused — a canceled `BKG-123` never becomes a new booking
- [ ] Behavior lives on the entity — `Booking.confirm()`, not `BookingService.confirm()`

### Test: entity identity
Verify two entities with the same attributes but different IDs are distinct — equality is by identity, not by value. Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Value Object

A **value object** has no identity: it is defined entirely by its attributes. Two `CargoWeight(42, TONNES)` are interchangeable.

### CargoRoute value objects

- `CargoWeight` — amount + unit; rejects negative values, forbids adding tonnes to kilograms without conversion
- `RouteDistance` — nautical miles; always non-negative
- `PortCode` — type-safe identifier preventing mixing origin/destination
- `VesselCapacity` — slots remaining; immutable, computed, never set from outside

### Design rules

- **Immutable** — to "change" a value, create a new one
- **Compare by value** — `equals()` and `hashCode()` from attributes
- **Validate at construction** — invalid states are unrepresentable
- **Prefer over primitives** — `CargoWeight` over `int weight` + `String unit`

### CargoRoute lesson: the PortCode mixup

We used `String origin` and `String destination` everywhere. A developer passed `destination` where `origin` was expected in a route feasibility check. The route looked valid — it went from London to Southampton on a vessel leaving from Southampton. The cargo never moved.

Fix: `PortCode` as a typed value object. The compiler now catches the swap.

### Test: value object validation
Test that invalid values are rejected at construction and that behavioral rules (e.g., refusing to add different units) throw. Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Aggregate

An **aggregate** is a cluster of entities and value objects treated as a single consistency unit, accessed only through its root.

### The CargoRoute Booking aggregate

The `Booking` aggregate owns:
- `BookingNumber` (identity)
- `CargoDetails` (weight, dimensions — value objects)
- `RouteSpecification` (origin → destination — value object)
- `Status` (PLACED, CONFIRMED, LOADED, DELIVERED, CANCELED)
- `events` (domain events to be published)

External systems reference only `BookingNumber`. Port Operations gets a `LoadJob` derived from the confirmed booking — never the booking itself.

### Decision tree: do you need an aggregate here?

```
Is this a cluster of related things that must change together?
  → No: it's just an entity or value object
  → Yes: do these invariants need to hold in the same transaction?
      → No: use eventual consistency (domain event + separate aggregate)
      → Yes: you have an aggregate. Is the cluster small?
          → No: you're trying to do too much. Split it.
          → Yes: is the root the only entry point?
              → No: external code reaches into the internals. Fix it.
              → Yes: correct aggregate.
```

### CargoRoute invariant checklist for aggregates

- [ ] Invariants are enforced at the root — `Booking.confirm()` checks route feasibility before allowing CONFIRMED
- [ ] External code references by ID only — PortOps gets `BookingNumber`, not the `Booking` entity
- [ ] One aggregate per transaction — we don't update Booking + Route + Vessel in one transaction
- [ ] Aggregate is small — Booking holds only what must change together
- [ ] Events are registered, not published manually — `registerEvent()` inside aggregate methods

### CargoRoute lesson: the VGM weight trap

We let `Booking` reference the full `Cargo` entity from the PortOps context. When Cargo's weight was verified (VGM — verified gross mass), it triggered a cascade: Booking recalculates, Route checks capacity, Vessel checks load balance. All in one transaction. Deadlocking hell.

Fix: Booking references `CargoWeight` as a value object (copied at confirmation time). PortOps publishes `CargoWeightVerified`, and a separate saga updates downstream aggregates asynchronously.

### The Aggregate Design Canvas (ddr-crew)

When a boundary is non-obvious, work through:

- **Name** — `Booking` (not `CargoBooking` — too long, not `LoadJob` — that's Port Ops' name for it)
- **Description** — "A customer's request to transport cargo, with route preferences and status lifecycle"
- **State transitions** — PLACED → CONFIRMED → LOADED → DELIVERED; CANCELED from any pre-LOADED state
- **Invariants & corrective policies** — "confirmed booking has a feasible route" (enforced); "capacity is checked optimistically, overbooking reconciled by a nightly audit job" (corrective policy)
- **Handled commands & created events** — `ConfirmBooking` → `BookingConfirmed`; `CancelBooking` → `BookingCanceled`
- **Throughput** — hundreds of bookings per minute during peak
- **Size** — bounded; one booking + its cargo details + route spec

### Test: aggregate invariant
Test the full state lifecycle: build the aggregate into a state, attempt an invalid operation, assert it is rejected. Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Domain Event

A **domain event** records that something meaningful happened — a fact that other aggregates or contexts may care about.

### CargoRoute events

- `BookingConfirmed` — published with booking number, route, vessel
- `CargoLoadedOnVessel` — published with vessel name, loading port, time
- `BookingCanceled` — published with reason

### Design rules

- **Past tense** — they describe facts already occurred
- **Immutable payload** — final fields, no setters
- **Enough data** — subscribers shouldn't need an extra round-trip for common cases
- **Not a snapshot** — don't serialize the entire aggregate

### CargoRoute lesson: the Tracking update delay

Tracking was polling Booking every 30 seconds. During a vessel departure delay, three systems showed three different statuses. Customers called support.

Fix: Booking publishes `CargoLoadedOnVessel`. Tracking subscribes and updates immediately. No polling.

### Test: event is raised
Verify that the right domain event is raised with the right payload when a triggering action occurs. Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Domain Service

A **domain service** holds domain logic that doesn't belong to a single entity — logic that spans entities or has no natural home object.

### CargoRoute: Route Feasibility Service

Checking whether a cargo can fit on a vessel requires comparing weight, dimensions, route distance, and capacity — spanning `CargoWeight`, `RouteSpecification`, and `VesselCapacity`. No single entity owns this check, so it's a service: it lives in the domain layer, takes the relevant objects as parameters, and returns a verdict. Implementation details vary by stack — see `references/spring-boot.md` for the Java idiom.

### CargoRoute lesson: the pricing calculator that shouldn't have been a service

A `PricingService` started as a legitimate domain service (combining distance, weight, and fuel surcharge). Then someone added "apply customer's loyalty discount" and "apply seasonal promotion." Soon it took 12 parameters and was called from every context.

Fix: split into `DistanceRateCalculator` (domain service) and `DiscountPolicy` (domain service). Each has one reason to change.

### Decision: domain service vs. entity method

- **Entity method** — the logic belongs to one thing and modifies its state
- **Domain service** — the logic spans multiple entities, or has no obvious owner

### Test: domain service
Test as a stateless function: give it known inputs (e.g., a route spec and a vessel itinerary with insufficient capacity) and assert the boolean output. Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Repository

A **repository** gives collection-like access to aggregates by their root. You `save` an aggregate and `find` it later in the same state.

### CargoRoute repository

One repository per aggregate root. The interface lives in the domain; the implementation lives in outbound. The interface is named in the ubiquitous language (e.g., `findConfirmedForVoyage`), not in storage terms.

### Rules

- Interface in the domain (no framework leakage)
- Named in the ubiquitous language (`findConfirmedForVoyage`, not `findByStatus`)
- Deals in whole aggregates, not inner entities

### CargoRoute lesson: the query that broke transactional isolation

We let PortOps call `bookingRepository.findConfirmedForVoyage()` directly. They started adding filters: "only bookings with HazardousGoods," "only bookings from CustomerTier.PREMIUM." The repository interface ballooned with use-case-specific finders.

Fix: PortOps gets its own projection (read model) updated from `BookingConfirmed` events. Repository stays clean.

### Test: repository round-trip
Test that saving an aggregate and loading it back by its root ID returns an equivalent object. Use an in-memory implementation for unit tests. Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Factory

A **factory** encapsulates the creation of complex objects — aggregates or value objects whose construction involves real logic or invariants that a plain constructor would obscure.

### CargoRoute: BookingFactory

Creating a valid booking requires combining cargo details, route preferences, customer info, and generating a booking number — with validation. The factory encapsulates this logic in one named, domain-language method so the construction rules can't be bypassed or scattered.

### Use one when:
- Construction involves assembly + invariants (e.g., creating a booking requires a valid route and weight)
- You want creation logic named in domain terms (`BookingFactory.createFrom` vs. scattered `new Booking(...)` calls)

If a constructor is clear and sufficient, you don't need a factory.

### Test: factory produces valid aggregate
Test that the factory always produces a well-formed aggregate with the correct initial state (e.g., status is PLACED, ID is generated). Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Choosing the right block

When modeling and unsure which block to reach for:

| Question | Answer | Block |
|---|---|---|
| Has identity you track over time? | Yes | Entity |
| Defined purely by attributes, interchangeable? | Yes | Value Object |
| Several objects must stay consistent together? | Yes, in same txn | Aggregate |
| Logic spans multiple entities / no natural home? | Yes | Domain Service |
| Something happens others must react to? | Yes | Domain Event |
| Persisting/retrieving this aggregate? | Yes | Repository (one per root) |
| Construction involves real decisions/rules? | Yes | Factory |

## CQRS

Command Query Responsibility Segregation separates the model that **changes** state from the model that **reads** it.

### How it works:

- The write side accepts commands, changes aggregates, publishes domain events.
- The read side maintains **read models** (projections): denormalized views optimized for querying, updated from events.
- The read model is **eventually consistent** with the write side.

### CargoRoute uses CQRS in Tracking

- **Write side:** Booking aggregate handles `ConfirmBooking`, publishes `BookingConfirmed`
- **Read side:** TrackingProjection listens to `BookingConfirmed`, `CargoLoadedOnVessel`, etc., maintaining a denormalized table: `(bookingNumber, currentPort, status, eta)`

### Naming commands, queries, events (and handlers)

Name the intent after what it does, in the ubiquitous language:

- **Command** — `Action + Target + Command`: `PlaceBookingCommand`, `ConfirmBookingCommand`, `CancelBookingCommand`.
- **Query** — `Action + Target + Criteria + Query`: `GetBookingByIdQuery`, `FindBookingsForVoyageQuery`.
- **Domain event** — `Target + PastAction`: `BookingConfirmed`, `BookingPlaced`, `CargoLoadedOnVessel`. Append `Event` only on collision with a non-event of the same name (e.g., a `BookingConfirmed` aggregate or command already exists).
- **Event handler** — `<EventName> + EventHandler`: `BookingConfirmedEventHandler`. One handler per event per module; handlers live in the module that raised the event and call other contexts through the ACL, never by subscribing across modules.

### When to use it

- Read and write needs genuinely diverge
- Reports span multiple aggregates
- Read vs. write load scales independently

### Cost and caution

CQRS adds moving parts. Many bounded contexts are well served by a single model. Apply it **per bounded context**, where it earns its keep.

### Test: projection updates from event
Test that a read model (projection) correctly updates when it receives the domain events it subscribes to. Verify the resulting view reflects the event's data. Business rules live in the domain, so test them as plain unit tests without a database or web context.

## Testing the tactical model

Business rules live inside domain objects, so they should be directly, cheaply testable **without** a database, web server, or mocking framework for infrastructure:

- **Entities and value objects** — plain unit tests: construct one, call a behavior, assert.
- **Aggregates** — test invariants directly: build a state, attempt an invalid operation, assert rejection.
- **Domain services** — like any stateless function: given inputs, assert output.
- **Domain events** — assert that the right event was raised with the right payload.
- **Repositories/factories** — unit test the factory logic; use an in-memory implementation for the repository.

If testing a piece of domain logic requires a database or web context, the logic isn't in the domain layer yet.

## Common pitfalls

- **Anemic aggregates:** root exposing public setters, with rule enforcement in the service layer. *Fix: move checks into domain methods.*
- **Aggregates that are too large:** embedding the full `Customer` and `Product` catalogs because one query needs them. *Fix: reference by ID, use a read model for cross-aggregate queries.*
- **Modifying two aggregates in one transaction "just this once."** *Fix: publish an event, let a handler update the second aggregate separately.*
- **Value objects that aren't immutable** (class with setters "for convenience"). *Fix: make them records or final classes.*
- **Domain services as dumping grounds** for anything that doesn't fit elsewhere. *Fix: if the behavior belongs on an entity or VO, move it back.*
- **CQRS applied everywhere by default** rather than where the divergence justifies it.

## Quick reference

| DDD concept | When you need it | Key design rule |
|---|---|---|
| Entity | Concept tracked over time with lifecycle | Identity is stable, not attributes |
| Value Object | Magnitude with rules; no identity | Immutable, validate at construction |
| Aggregate | Group of objects that must stay consistent in one transaction | Small, root-only access, reference other aggregates by ID |
| Aggregate Root | Referenced by ID from outside the aggregate | Only the root enforces invariants |
| Domain Event | Other parts need to react to something that happened | Past tense (`Target + PastAction`), immutable, enough data to avoid round-trips; `Event` suffix only on collision |
| Domain Service | Logic spans multiple entities or has no natural home | Stateless, in the domain layer |
| Repository | Persisting and retrieving whole aggregates | One per aggregate root, interface in domain |
| Factory | Construction involves decisions and rules | Named method, always produces valid objects |
| Command | Write side of CQRS | Immutable intent, validated at construction, named `Action + Target + Command` |
| Query | Read side of CQRS | Read-only, no side effects, named `Action + Target + Criteria + Query` |
| Read Model | Optimized querying across aggregates | Rebuilt from events, eventually consistent |
