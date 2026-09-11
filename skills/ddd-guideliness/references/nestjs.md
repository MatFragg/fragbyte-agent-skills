# Implementing DDD in NestJS

Read this when writing DDD code in **NestJS / TypeScript**. It assumes you know *what* the building blocks are (see `SKILL.md` and `references/tactical-patterns.md`); here we show *how to express them idiomatically*, walking through what CargoRoute's Booking module looked like **before** we knew better and **after** the refactor.

Works on NestJS 10+ with TypeScript 5+ (`strict: true`). Persistence uses **TypeORM**. Validation uses `class-validator` + `class-transformer`. Domain events use `@nestjs/event-emitter`, not a command bus — see [A note on `@nestjs/cqrs`](#a-note-on-nestjscqrs) for why.

## Contents

- [Module structure](#module-structure)
- [The shared kernel](#the-shared-kernel)
- [Value objects](#value-objects)
- [Entities and aggregate roots](#entities-and-aggregate-roots)
- [Domain events](#domain-events)
- [Commands and queries](#commands-and-queries)
- [Command and query services](#command-and-query-services)
- [Repositories](#repositories)
- [Outbound services and dependency injection with tokens](#outbound-services-and-dependency-injection-with-tokens)
- [Anti-corruption layer](#anti-corruption-layer)
- [Interfaces (REST)](#interfaces-rest)
- [Domain exceptions and error handling](#domain-exceptions-and-error-handling)
- [Identity and persistence: preferences](#identity-and-persistence-preferences)
- [A note on `@nestjs/cqrs`](#a-note-on-nestjscqrs)
- [Testing each layer](#testing-each-layer)
- [Common pitfalls](#common-pitfalls)
- [Quick reference](#quick-reference)

## Module structure

Give each **bounded context** its own NestJS module, split into four layers, with dependencies pointing inward toward `domain`:

```
src/booking/
├── interfaces/                        // inbound adaptors — the outside drives the context
│   ├── rest/
│   │   ├── booking.controller.ts      // REST controllers
│   │   ├── resources/                 // request/response DTOs (class-validator)
│   │   └── transform/                 // assemblers: resource <-> command / entity
│   ├── acl/                           // facade interface this context exposes to other contexts
│   └── filters/                       // BC-scoped exception filter
├── application/
│   └── internal/
│       ├── commandservices/           // command service implementations
│       ├── queryservices/             // query service implementations
│       ├── eventhandlers/             // react to domain events (@OnEvent)
│       └── outboundservices/          // outbound ports (optional — only when context calls out)
│           ├── {concept}/             // technology port per concept (hashing/, tokens/, llm/)
│           │   └── {concept}.service.ts
│           └── acl/
│               └── external-{bc}.service.ts   // calls another context's facade
├── domain/                            // the domain model + its ports (depends on nothing)
│   ├── model/
│   │   ├── aggregates/                // aggregate roots only
│   │   ├── entities/                  // internal entities within aggregates
│   │   ├── valueobjects/
│   │   ├── commands/                  // command types (domain)
│   │   ├── queries/                   // query types (domain)
│   │   └── events/                    // domain events
│   ├── services/                      // command/query service interfaces (ports) + tokens
│   ├── repositories/                  // repository interfaces (ports) + tokens
│   └── exceptions/                    // domain-specific exceptions
├── infrastructure/                    // outbound adaptors — the context reaches out
│   ├── persistence/
│   │   └── typeorm/
│   │       ├── entities/              // *.orm-entity.ts (TypeORM @Entity)
│   │       ├── assemblers/            // orm-entity <-> domain entity
│   │       └── adapters/              // repository port implementations
│   └── {technology}/                  // technology-specific adapters
│       └── {implementation}/          // e.g., bcrypt/, jwt/, anthropic/
└── booking.module.ts
```

- **`interfaces`** — inbound adaptors. REST controllers, listeners, CLI. They translate external input into application calls. No business logic. The `acl/` subfolder holds the facade interface this context publishes for others.
- **`application`** — application services. They orchestrate use cases: load aggregates, invoke behavior, save through the repository port. Coordinate, hold no business rules. The `outboundservices/` subfolder holds technology-agnostic port interfaces — concept subfolders (`hashing/`, `tokens/`, `llm/`) for external dependencies, and `acl/` exclusively for `External{Bc}Service` classes that call other contexts' facades.
- **`domain`** — the model. Aggregates, entities, value objects, domain events, service and repository *interfaces* (ports) with their injection tokens, exceptions. Every business rule lives here. **Zero imports from `@nestjs/*`, `typeorm`, or `class-validator`.**
- **`infrastructure`** — outbound adaptors. TypeORM entities, repository adapters, external API clients. Follows the `infrastructure/{technology}/{implementation}/` pattern (e.g., `infrastructure/hashing/bcrypt/`).

### Why `repositories/` sits in `domain`, not `application`

Unlike the interfaces used only within `application` (command/query services), a repository port is a first-class domain concept — the aggregate's persistence contract. Keeping `repositories/` at the top of `domain/` (sibling to `model/` and `services/`) mirrors the boundary: the domain declares *what* it needs persisted, `infrastructure` decides *how*.

## The shared kernel

A `shared/` folder holds the small **shared kernel** every module reuses — base classes, a common response resource, cross-cutting config. Keep it small and stable; it must never hold business rules.

```typescript
// shared/domain/model/domain-event.ts
// Every domain event implements this — see "Domain events" for why this is
// mandatory rather than `unknown` or `any`.
export interface DomainEvent {
  readonly eventName: string;
  readonly occurredAt: Date;
}
```

```typescript
// shared/domain/model/aggregate-root.ts
// The domain counterpart of Spring's AbstractAggregateRoot: every aggregate root
// raises domain events into this buffer and drains them after the unit of work
// commits. No persistence or framework types — just the event drain.
export abstract class AggregateRoot {
  protected readonly _domainEvents: DomainEvent[] = [];

  protected registerEvent(event: DomainEvent): void {
    this._domainEvents.push(event);
  }

  pullDomainEvents(): DomainEvent[] {
    const events = [...this._domainEvents];
    this._domainEvents.length = 0;
    return events;
  }
}
```

```typescript
// shared/infrastructure/persistence/auditable.orm-entity.ts
export abstract class AuditableOrmEntity {
  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

```typescript
// shared/interfaces/rest/resources/message.resource.ts
export class MessageResource {
  constructor(public readonly message: string) {}
}
```

```typescript
// shared/domain/repositories/unit-of-work.ts
// The unit of work is a persistence concept (see "Repositories"), but its PORT
// lives here in the shared kernel so the application layer can ask for a
// transaction without knowing which ORM is behind it. The implementation is in
// infrastructure, exactly like any other port+adapter pair.
export const UNIT_OF_WORK = Symbol('UNIT_OF_WORK');

export interface IUnitOfWork {
  run<T>(fn: () => Promise<T>): Promise<T>;
}
```

```typescript
// shared/infrastructure/persistence/typeorm/repositories/transaction-context.ts
// TypeORM, unlike EF Core's SaveChangesAsync(), has no ambient deferred context:
// each repository call commits its own transaction. So the unit of work holds the
// transactional EntityManager in an AsyncLocalStorage, and repository adapters
// read it from here. This is the same mechanism typeorm-transactional uses.
import { AsyncLocalStorage } from 'node:async_hooks';
import { EntityManager } from 'typeorm';

const storage = new AsyncLocalStorage<EntityManager | null>();

export const TransactionContext = {
  run<T>(manager: EntityManager, fn: () => Promise<T>): Promise<T> {
    return storage.run(manager, fn);
  },
  // The transactional manager, or null when no UnitOfWork.run() is active.
  get current(): EntityManager | null {
    return storage.getStore() ?? null;
  },
  // The transactional manager, throwing when none is active — for writes that
  // must join the running transaction and shouldn't silently run standalone.
  get manager(): EntityManager {
    const manager = storage.getStore();
    if (!manager) throw new Error('No active transaction — run inside UnitOfWork.run()');
    return manager;
  },
};
```

```typescript
// shared/infrastructure/persistence/typeorm/repositories/unit-of-work.impl.ts
@Injectable()
export class TypeOrmUnitOfWork implements IUnitOfWork {
  constructor(@InjectDataSource() private readonly dataSource: DataSource) {}

  async run<T>(fn: () => Promise<T>): Promise<T> {
    const runner = this.dataSource.createQueryRunner();
    await runner.connect();
    await runner.startTransaction();
    try {
      const result = await TransactionContext.run(runner.manager, fn);
      await runner.commitTransaction();
      return result;
    } catch (error) {
      await runner.rollbackTransaction();
      throw error;
    } finally {
      await runner.release();
    }
  }
}
```

```typescript
// shared/interfaces/rest/filters/global-exception.filter.ts
@Catch(HttpException)
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    response.status(exception.getStatus()).json({
      message: exception.message,
      statusCode: exception.getStatus(),
    });
  }
}
```

Register it once, in the app root — never per-module:

```typescript
// main.ts
const app = await NestFactory.create(AppModule);
app.useGlobalFilters(new GlobalExceptionFilter());
app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
await app.listen(3000);
```

## Value objects

Model value objects as **immutable** classes: `private readonly` fields, validation in the constructor, an `equals()` method for value comparison, no setters. Value objects never carry TypeORM or `class-validator` decorators — those belong to the ORM entity and the resource, respectively.

**Apply this consistently to identifiers.** If an object crossing a method boundary refers to another aggregate, or has business-facing identity at all, wrap it — never pass a bare `string`/`number`. The only legitimate exception is an internal entity's id that never leaves its aggregate and has no meaning in the ubiquitous language (see [Identity and persistence](#identity-and-persistence-preferences)); even then, prefer wrapping once a second call site needs it, since "just this once" is how bare strings spread.

### CargoRoute: CargoWeight, PortCode, BookingNumber, CargoId, VoyageNumber, RouteId

```typescript
// domain/model/valueobjects/cargo-weight.value-object.ts
// Closed sets use an `as const` object + type union — the TS equivalent of
// Spring's `enum`. Do not use `export enum`: it is non-erasable, breaks under
// `isolatedModules` / `erasableSyntaxOnly`, and diverges from `angular.md`.
export const WeightUnit = { TONNES: 'TONNES', KILOGRAMS: 'KILOGRAMS' } as const;
export type WeightUnit = typeof WeightUnit[keyof typeof WeightUnit];

export class CargoWeight {
  constructor(
    public readonly amount: number,
    public readonly unit: WeightUnit,
  ) {
    if (amount < 0) throw new Error('weight cannot be negative');
  }

  add(other: CargoWeight): CargoWeight {
    if (this.unit !== other.unit) throw new Error('cannot add different units');
    return new CargoWeight(this.amount + other.amount, this.unit);
  }

  equals(other: CargoWeight): boolean {
    return this.amount === other.amount && this.unit === other.unit;
  }
}

// Parse untrusted input (persistence, external APIs) into the union set explicitly.
// Never `as WeightUnit` — a corrupt DB value would silently pass as valid.
export function parseWeightUnit(value: string): WeightUnit {
  if (!(Object.values(WeightUnit) as string[]).includes(value)) throw new Error(`invalid weight unit: ${value}`);
  return value as WeightUnit;
}
```

```typescript
// domain/model/valueobjects/port-code.value-object.ts
export class PortCode {
  private constructor(public readonly value: string) {}

  static of(value: string): PortCode {
    if (!value?.trim()) throw new Error('port code required');
    return new PortCode(value);
  }

  equals(other: PortCode): boolean { return this.value === other.value; }
}
```

```typescript
// domain/model/valueobjects/booking-number.value-object.ts
export class BookingNumber {
  private constructor(public readonly value: string) {}

  static generate(): BookingNumber {
    return new BookingNumber(`BKG-${randomUUID().slice(0, 8).toUpperCase()}`);
  }

  static of(value: string): BookingNumber {
    if (!/^BKG-[A-F0-9]+$/.test(value)) throw new Error('invalid booking number format');
    return new BookingNumber(value);
  }

  equals(other: BookingNumber): boolean { return this.value === other.value; }
}

// Use value objects for typed identifiers too, so a reference to another aggregate
// stays type-safe and meaningful — never a bare string or number.
export class CustomerId {
  private constructor(public readonly value: string) {}

  static of(value: string): CustomerId {
    if (!value) throw new Error('invalid customer id');
    return new CustomerId(value);
  }

  equals(other: CustomerId): boolean { return this.value === other.value; }
}
```

**VO construction conventions.** Quantity VOs — a measurement like `CargoWeight` — use a public constructor and validate in it. Identifier VOs — `BookingNumber`, `CustomerId`, `PortCode`, `VoyageNumber`, `RouteId`, `CargoId`, `ShipmentId` — use a `private` constructor plus a static `of()` (parse + validate) and `generate()` where identity is created. That keeps the coercion boundary explicit: a `CustomerId` is only ever built from a named source, not from a `new` scattered through an orchestrator. *Reference-data* carriers — `VesselInfo`, `VesselBookingInfo`, `VoyageContext` (see [Cross-context reference data](#cross-context-reference-data)) — are read-only DTOs at an ACL boundary: a public constructor is correct there, since they carry no identity or invariants.

```typescript
// domain/model/valueobjects/voyage-number.value-object.ts
// The voyage a booking gets confirmed onto — owned by Booking once assigned,
// even though Voyage Scheduling is a different bounded context. Booking only
// ever needs the identifier to query and expose, never the scheduling
// context's full model, so a VO (not an ACL round-trip) is the right size.
export class VoyageNumber {
  private constructor(public readonly value: string) {}

  static of(value: string): VoyageNumber {
    if (!value?.trim()) throw new Error('voyage number required');
    return new VoyageNumber(value);
  }

  equals(other: VoyageNumber): boolean { return this.value === other.value; }
}
```

```typescript
// domain/model/valueobjects/route-id.value-object.ts
// Identifies the route a booking was confirmed onto. Travels alongside
// VoyageNumber on BookingConfirmed — mirrors spring-boot.md's
// `BookingConfirmed(BookingNumber, RouteId, VoyageNumber)`, where both
// cross-context references are typed VOs, never bare strings.
export class RouteId {
  private constructor(public readonly value: string) {}

  static of(value: string): RouteId {
    if (!value?.trim()) throw new Error('route id required');
    return new RouteId(value);
  }

  equals(other: RouteId): boolean { return this.value === other.value; }
}
```

```typescript
// domain/model/valueobjects/cargo-id.value-object.ts
// Cargo is an internal entity, but a CargoId still gets a VO: it appears in
// events, logs, and (later) a "resend manifest for cargo X" finder — the same
// rule as CustomerId/BookingNumber above, applied without exceptions.
export class CargoId {
  private constructor(public readonly value: string) {}

  static generate(): CargoId {
    return new CargoId(randomUUID());
  }

  static of(value: string): CargoId {
    if (!value) throw new Error('invalid cargo id');
    return new CargoId(value);
  }

  equals(other: CargoId): boolean { return this.value === other.value; }
}

// domain/model/valueobjects/shipment-id.value-object.ts
export class ShipmentId {
  private constructor(public readonly value: string) {}

  static generate(): ShipmentId {
    return new ShipmentId(randomUUID());
  }

  static of(value: string): ShipmentId {
    if (!value) throw new Error('invalid shipment id');
    return new ShipmentId(value);
  }

  equals(other: ShipmentId): boolean { return this.value === other.value; }
}
```

### What this saved us

Using `origin: string` and `destination: string` everywhere, a developer passed `destination` where `origin` was expected in a route check. The route looked valid — London to Southampton on a vessel leaving from Southampton. The cargo never moved. Typed `PortCode` value objects make this a compile error: `checkRoute(destination, origin)` still compiles in plain TS, but once callers only ever construct a `PortCode` from a named source (`PortCode.origin(...)`, or simply distinct parameter types when the domain calls for it), the swap becomes visible at the call site instead of buried in a passing test.

### Test: VO validation

```typescript
describe('CargoWeight', () => {
  it('rejects negative weight', () => {
    expect(() => new CargoWeight(-1, WeightUnit.TONNES)).toThrow();
  });

  it('rejects mixing units', () => {
    const tonnes = new CargoWeight(10, WeightUnit.TONNES);
    const kg = new CargoWeight(500, WeightUnit.KILOGRAMS);
    expect(() => tonnes.add(kg)).toThrow();
  });
});
```

## Entities and aggregate roots

An **entity** is an object with a distinct identity that persists through changes to its attributes. In DDD, entities come in two flavors, and in NestJS **neither one is the TypeORM entity**:

- **Aggregate root** — the entry point to an aggregate, a plain TypeScript class in `domain/model/aggregates/` with real behavior. It has no decorators at all.
- **Internal entity** — an entity *inside* an aggregate, reachable only through the root, in `domain/model/entities/` (flat folder). Also a plain class.

The TypeORM class in `infrastructure/persistence/typeorm/entities/` is a **separate, third thing**: the persistence shape. A `BookingOrmEntity` and the domain `Booking` are never the same class.

### Why not decorate the domain class directly

Spring can annotate the aggregate root with `@Entity` because JPA is metadata-only. TypeORM's decorators pull in column types, relations, and lazy-loading semantics that leak persistence concerns into the domain; keeping the split is the safer default in Nest. Treat "decorate the domain class" as a shortcut for trivial, CRUD-only modules only.

### CargoRoute: the naive Booking (before)

```typescript
// what we shipped — an anemic aggregate decorated straight onto the ORM shape
@Entity('bookings')
export class Booking {
  @PrimaryGeneratedColumn('uuid') id: string;
  @Column() customerEmail: string;      // no validation
  @Column() originPort: string;         // raw string — typo-prone
  @Column() destinationPort: string;    // easy to mix
  @Column() status: string;             // "CONFIRMED"? "CONFIRMD"?

  // No behavior. A BookingService checks: if (booking.status !== 'CONFIRMED') throw ...
}
```

Every rule lives in a service layer. Every typo in a status string is a production bug. Rules are optional — if you forget to call the service, the invariant is gone.

### After: the idiomatic Booking aggregate

The constructor below is for **creating a new booking** — it always starts `PLACED` and always registers `BookingPlaced`. Reloading an existing booking from storage is a *different* operation with different rules (no new event, arbitrary starting status), so it gets its own named path: the static `rehydrate()` factory, paired with a `private` full-state constructor so the two can never be confused.

```typescript
// domain/model/aggregates/booking.aggregate.ts
export const BookingStatus = { PLACED: 'PLACED', CONFIRMED: 'CONFIRMED', LOADED: 'LOADED', CANCELED: 'CANCELED' } as const;
export type BookingStatus = typeof BookingStatus[keyof typeof BookingStatus];

export class Booking extends AggregateRoot {
  // Private: only reachable through the two named factories below, so a
  // caller can never construct a Booking in an ambiguous "is this new or
  // reloaded?" state.
  private constructor(
    private readonly _bookingNumber: BookingNumber,
    private readonly _customerId: CustomerId,
    private readonly _origin: PortCode,
    private readonly _destination: PortCode,
    private _status: BookingStatus,
    private readonly _cargo: Cargo,
    // Assigned once the booking is confirmed. Lives on the aggregate itself —
    // not just on the ORM shape — because "a booking only carries a voyage
    // after CONFIRMED" is an invariant the domain owns, and because a
    // repository finder needs a real field to query, not a value patched in
    // from outside after the fact. Mirrors `spring-boot.md`'s Booking, whose
    // `findConfirmedForVoyage(VoyageNumber voyageNumber)` likewise assumes
    // the entity itself carries the voyage it was confirmed onto.
    private _voyageNumber: VoyageNumber | null = null,
  ) {}

  /** Creates a brand-new booking. Always PLACED; always raises BookingPlaced. */
  static place(input: {
    customerId: CustomerId; origin: PortCode; destination: PortCode; cargo: Cargo;
  }): Booking {
    const booking = new Booking(
      BookingNumber.generate(),
      input.customerId,
      input.origin,
      input.destination,
      BookingStatus.PLACED,
      input.cargo,
      null, // no voyage until confirmed
    );
    booking.registerEvent(new BookingPlaced(booking._bookingNumber, booking._customerId));
    return booking;
  }

  /** Reconstructs a booking already known to storage. No events, no status assumptions. */
  static rehydrate(input: {
    bookingNumber: BookingNumber; customerId: CustomerId; origin: PortCode;
    destination: PortCode; status: BookingStatus; cargo: Cargo; voyageNumber: VoyageNumber | null;
  }): Booking {
    return new Booking(
      input.bookingNumber, input.customerId, input.origin,
      input.destination, input.status, input.cargo, input.voyageNumber,
    );
  }

  confirm(feasibility: RouteFeasibilityService, proposal: RouteProposal): void {
    if (this._status !== BookingStatus.PLACED)
      throw new IllegalBookingStateError('can only confirm a placed booking');
    if (!feasibility.isFeasible(proposal, this._cargo.weight))
      throw new RouteNotFeasibleError('proposed route cannot handle cargo weight');
    this._status = BookingStatus.CONFIRMED;
    // proposal.vessel already arrives as a VoyageNumber from the ACL boundary
    // (see "Anti-corruption layer") — assign it as-is, never re-wrap a value
    // that's already the right type.
    this._voyageNumber = proposal.vessel;
    this.registerEvent(
      new BookingConfirmed(this._bookingNumber, RouteId.of(proposal.routeId), this._voyageNumber),
    );
  }

  loadCargo(): void {
    if (this._status !== BookingStatus.CONFIRMED)
      throw new IllegalBookingStateError('cannot load an unconfirmed booking');
    this._status = BookingStatus.LOADED;
    this.registerEvent(new CargoLoadedOnVessel(this._bookingNumber));
  }

  cancel(reason: string): void {
    if (this._status === BookingStatus.LOADED)
      throw new IllegalBookingStateError('cannot cancel a loaded booking');
    this._status = BookingStatus.CANCELED;
    this.registerEvent(new BookingCanceled(this._bookingNumber, reason));
  }

  get bookingNumber(): BookingNumber { return this._bookingNumber; }
  get customerId(): CustomerId { return this._customerId; }
  get origin(): PortCode { return this._origin; }
  get destination(): PortCode { return this._destination; }
  get status(): BookingStatus { return this._status; }
  get voyageNumber(): VoyageNumber | null { return this._voyageNumber; }
  get cargo(): Cargo { return this._cargo; }
}
```

`pullDomainEvents()` returns a typed `DomainEvent[]`, not `unknown[]` — see [Domain events](#domain-events) for why that typing isn't optional in strict TypeScript.

### Internal entity: Cargo

```typescript
// domain/model/entities/cargo.entity.ts
export class Cargo {
  constructor(
    public readonly id: CargoId,
    public readonly weight: CargoWeight,
    public readonly description: string,
    public readonly hazardous: boolean,
  ) {}

  validateForVessel(capacity: VesselCapacity): void {
    if (this.hazardous && !capacity.allowsHazardous)
      throw new Error('vessel does not allow hazardous cargo');
    if (this.weight.amount > capacity.maxWeightTonnes)
      throw new Error('cargo exceeds vessel weight capacity');
  }
}
```

### What this fixed

| Before | After | Saved us from |
|---|---|---|
| `id: string` from `@PrimaryGeneratedColumn` — mixable with any other string | `BookingNumber` — typed, validated | The $4M ship-to-wrong-port incident (see `strategic-design.md`) |
| `status: string` — typo = broken flow | `BookingStatus` union (`as const`) — compiler-enforced | Three failed deployments |
| Rules in a service layer — bypassable | Rules in the aggregate — unbreakable | The booking canceled after loading |
| Domain class decorated with `@Entity` — TypeORM leaks into behavior | Domain class plain, ORM entity separate | Lazy-loading exceptions in unit tests |
| Single public constructor doubling as "create" and "reload" | `place()` vs `rehydrate()` — distinct, unambiguous factories | A reload path accidentally re-raising `BookingPlaced` |

### Test: the invariant that used to be a service-level if-check

```typescript
describe('Booking', () => {
  it('cannot cancel after cargo is loaded', () => {
    const booking = Booking.place(placedBookingInput());
    booking.confirm(feasibilityServiceStub(), routeProposal());
    booking.loadCargo();

    expect(() => booking.cancel('customer request')).toThrow(IllegalBookingStateError);
  });
});
```

### Composite: a root that owns a collection and derives its state from the parts

`Booking` owns a single, detail-like `Cargo` — the deciding rule is: *is the child an entity with its own identity and lifecycle, or effectively a value object of the root?* When the root owns **many** children, each with identity and a status the root must aggregate over, use the **Composite** flavor: the root owns the collection, builds children through create-methods (dedup + validity at the point of creation), and derives whole-state from the children instead of carrying a parallel flag (see `references/design-patterns-arch-patterns.md`).

#### CargoRoute: `Shipment` holding many `CargoItem`s

```typescript
// domain/model/entities/cargo-item.entity.ts
export const CargoStatus = { DECLARED: 'DECLARED', CONFIRMED: 'CONFIRMED', LOADED: 'LOADED' } as const;
export type CargoStatus = typeof CargoStatus[keyof typeof CargoStatus];

export class CargoItem {
  constructor(
    public readonly id: CargoId,
    public readonly reference: string,
    public readonly weight: CargoWeight,
    public readonly temperature: number | null,
    public readonly kind: 'container' | 'bulk' | 'reefer',
    private _status: CargoStatus = CargoStatus.DECLARED,
  ) {}

  get status(): CargoStatus { return this._status; }
  isTrackable(): boolean { return this.reference !== null; }

  confirm(): void {
    if (this._status !== CargoStatus.DECLARED)
      throw new IllegalCargoStateError('only a declared cargo can be confirmed');
    this._status = CargoStatus.CONFIRMED;
  }
}
```

```typescript
// domain/model/aggregates/shipment.aggregate.ts
export const ShipmentStatus = { PLACED: 'PLACED', CONFIRMED: 'CONFIRMED', LOADED: 'LOADED' } as const;
export type ShipmentStatus = typeof ShipmentStatus[keyof typeof ShipmentStatus];

export class Shipment extends AggregateRoot {
  private readonly _cargo: CargoItem[] = [];
  private _status: ShipmentStatus = ShipmentStatus.PLACED;

  constructor(private readonly _id: ShipmentId) {}

  // Create-methods on the root. Both flavors of the Factory pattern from
  // design-patterns-arch-patterns.md live here: the root builds children and
  // hides `new`, so dedup and validity can't be bypassed from a service.
  addContainer(reference: string, weight: CargoWeight): void {
    if (this.exists(reference)) return;                       // dedup — same ref is a no-op
    this._cargo.push(new CargoItem(CargoId.generate(), reference, weight, null, 'container'));
  }

  addBulkCargo(tonnage: number): void {
    if (tonnage <= 0) throw new InvalidTonnageError('tonnage must be positive');   // invariant at creation
    this._cargo.push(new CargoItem(CargoId.generate(), `bulk-${this._cargo.length + 1}`, new CargoWeight(tonnage, WeightUnit.TONNES), null, 'bulk'));
  }

  addReeferCargo(reference: string, temperature: number, weight: CargoWeight): void {
    if (this.exists(reference)) return;
    this._cargo.push(new CargoItem(CargoId.generate(), reference, weight, temperature, 'reefer'));
  }

  // Child transitions also go through the root — the only entry point. The
  // getter below returns a copy of the collection, so confirming happens here.
  confirmCargo(cargoId: CargoId): void {
    const cargo = this._cargo.find((c) => c.id.equals(cargoId));
    if (!cargo) throw new UnknownCargoError('cargo not found in this shipment');
    cargo.confirm();
  }

  // Composite rule: the whole's invariant is *defined by* the parts, not a flag
  // someone must keep in sync. isReadyForLoading() is a derived getter...
  isReadyForLoading(): boolean {
    return this._cargo.length > 0 && this._cargo.every((c) => c.status === CargoStatus.DECLARED);
  }

  // ...and confirm() is a guarded transition that reads the children's state,
  // then sets the root's own status and raises the event.
  confirm(): void {
    if (this._cargo.length === 0 || !this._cargo.every((c) => c.status === CargoStatus.CONFIRMED))
      throw new IncompleteShipmentError('all cargo must be confirmed before the shipment can be confirmed');
    this._status = ShipmentStatus.CONFIRMED;
    this.registerEvent(new ShipmentConfirmed(this._id));
  }

  hasTrackableCargo(): boolean { return this._cargo.some((c) => c.isTrackable()); }
  get cargo(): CargoItem[] { return [...this._cargo]; }
  get status(): ShipmentStatus { return this._status; }

  private exists(reference: string): boolean {
    return this._cargo.some((c) => c.reference === reference);
  }
}

// domain/model/events/shipment-confirmed.event.ts
export class ShipmentConfirmed implements DomainEvent {
  static readonly eventName = 'ShipmentConfirmed';
  readonly eventName = ShipmentConfirmed.eventName;
  readonly occurredAt = new Date();

  constructor(public readonly shipmentId: ShipmentId) {}
}
}
```

A service calling `new CargoItem(...)` or `cargo.confirm()` directly can't exist — creation and child transitions go through the root (`addContainer`, `confirmCargo`), so a duplicate reference, negative tonnage, or an out-of-order confirmation is unrepresentable. Compare with `spring-boot.md`'s `Route` + `List<RouteStop>`: the same "root owns a collection" shape.

## Domain events

Model each event as a plain class, named in past tense, carrying only the data subscribers need. Every event implements `DomainEvent` and carries a static `eventName` (`constructor.name` is not stable under minification, and an `unknown` array would not type-check under `strict: true`).

**Naming:** `Event` is `Target + PastAction` (`BookingConfirmed`, `CargoLoadedOnVessel`); append `Event` only on collision with a non-event of the same name. `Event handlers` are `<EventName> + EventHandler` (`BookingConfirmedEventHandler` in `application/internal/eventhandlers/`).

Payloads carry the same typed VOs as everywhere else in the domain — never bare strings for business-facing identity (mirroring `spring-boot.md`).

```typescript
// domain/model/events/booking-confirmed.event.ts
export class BookingConfirmed implements DomainEvent {
  static readonly eventName = 'BookingConfirmed';
  readonly eventName = BookingConfirmed.eventName;
  readonly occurredAt = new Date();

  constructor(
    public readonly bookingNumber: BookingNumber,
    public readonly routeId: RouteId,
    public readonly vessel: VoyageNumber,
  ) {}
}
```

The aggregate collects events via `pullDomainEvents()` (see [Entities and aggregate roots](#entities-and-aggregate-roots)); the command service drains and publishes them **after the unit of work commits** (i.e. after `unitOfWork.run(...)` resolves), using `EventEmitter2` from `@nestjs/event-emitter`:

```typescript
// application/internal/commandservices/booking-command.service.impl.ts (excerpt)
private publishDomainEvents(booking: Booking): void {
  for (const event of booking.pullDomainEvents()) {
    this.eventEmitter.emit(event.eventName, event);
  }
}
```

```typescript
// application/internal/eventhandlers/booking-confirmed.event-handler.ts
@Injectable()
export class BookingConfirmedEventHandler {
  @OnEvent(BookingConfirmed.eventName)
  async handle(event: BookingConfirmed): Promise<void> {
    // trigger downstream: notify Tracking, update Route assignment
  }
}
```

Register `EventEmitterModule.forRoot()` **exactly once, in the application root** (`app.module.ts`) — it configures a global dynamic module, and calling it again from a feature module doesn't extend it, it just re-registers global state. Feature modules only need to inject `EventEmitter2`; they don't import or call `forRoot()`.

```typescript
// app.module.ts — the ONLY place forRoot() is called
@Module({
  imports: [EventEmitterModule.forRoot(), BookingModule /* ... */],
})
export class AppModule {}
```

### Why "after commit" matters, and how Nest makes you enforce it manually

Spring's `AbstractAggregateRoot` publishes registered events automatically when the aggregate is saved through Spring Data, tying publication to the transaction outcome by construction. `EventEmitter2` has no such hook into TypeORM's save lifecycle — so the unit of work owns the guarantee: **`publishDomainEvents()` must be called only after `unitOfWork.run(...)` resolves**, since `run()` commits (or rolls back) the transaction inside its callback before resolving. Never publish inside the `run()` callback, never before it, and never inside a `try` block whose `catch` still runs. If the save throws, `run()` rolls back and the loop publishing events never executes. Get the ordering wrong and other contexts react to bookings that don't exist yet — the exact bug `tactical-patterns.md` documents for the Unit of Work pattern.

For handlers with real external side effects (sending an email, calling another service), consider `{ async: true }` in `@OnEvent(BookingConfirmed.eventName, { async: true })` so a slow handler can't block the request, and wrap the handler body in its own error handling — an unhandled rejection in an event handler does not roll back the save that already committed.

### Domain events never subscribed to from another module

`@nestjs/event-emitter` has no module isolation: `EventEmitterModule.forRoot()` registers one `EventEmitter2` for the **entire app**, and any `@OnEvent(name)` anywhere receives it, regardless of which module emitted it. That makes it tempting for another bounded context's module to subscribe directly:

```typescript
// ✗ tracking/application/internal/eventhandlers/booking-confirmed.event-handler.ts
@Injectable()
export class TrackingBookingConfirmedEventHandler {
  @OnEvent(BookingConfirmed.eventName)
  async handle(event: BookingConfirmed): Promise<void> { /* ... */ } // imports Booking's own event + RouteId + VoyageNumber
}
```

Don't. This is the same reversed dependency the [Anti-corruption layer](#anti-corruption-layer) section exists to prevent, just reached through the event bus instead of a facade call: Tracking would have to import Booking's event class and its VOs just to type a handler parameter. `strategic-design.md`'s own context map already settles this for CargoRoute — Routing and Billing consume `BookingConfirmed` "through their own ACL," not by subscribing to Booking's raw domain event.

**A `DomainEvent` is only ever handled by a `@OnEvent` listener inside the module that raised it.** When another context genuinely needs to know — per the context map, not per convenience — that in-module handler is the one that reacts, and it reacts by calling the other context through the **same ACL machinery already described**: an `External{Bc}Service` injecting that context's facade token, exactly as [Anti-corruption layer](#anti-corruption-layer) shows. No second event type, no new interface — this is a plain in-process call, because in a modular monolith the "other bounded context" is just another module in the same process, not a separate deployable talking over a broker:

```typescript
// application/internal/eventhandlers/booking-confirmed.event-handler.ts
@Injectable()
export class BookingConfirmedEventHandler {
  // Same ExternalRoutingService shape as ExternalVesselService in the ACL
  // section — a facade token this handler injects, nothing event-specific.
  constructor(private readonly routing: ExternalRoutingService) {}

  @OnEvent(BookingConfirmed.eventName)
  async handle(event: BookingConfirmed): Promise<void> {
    // Booking's own VOs are fine here — this code never leaves the module.
    // Notifying Routing is an ordinary ACL call, primitives at the boundary,
    // exactly like ExternalVesselService.proposeRoute in the ACL section.
    await this.routing.notifyBookingConfirmed(event.bookingNumber.value, event.routeId.value);
  }
}
```

If a real multi-service deployment eventually splits Routing or Billing into their own process, the facade's implementation moves from an in-process call to an HTTP/gRPC client — same pattern already described for the ACL in general — without this handler changing at all.

This is a narrower rule than it might look: the Shared Kernel threshold (a value object independently needed by more than two contexts — see `references/domain-modeling.md`) is about *value objects*, not about how events get consumed. Whether one context reacts to another's event or three do, the mechanism is the same in-process ACL call shown above — it's not a separate decision to make per event.

## Commands and queries

Make **commands** and **queries** first-class types in the domain, as plain classes that validate their own input at construction. No `implements ICommand` from `@nestjs/cqrs` — see [A note on `@nestjs/cqrs`](#a-note-on-nestjscqrs).

**Naming:** `Command` is `Action + Target + Command` (`PlaceBookingCommand`, `CancelBookingCommand`); `Query` is `Action + Target + Criteria + Query` (`GetBookingByIdQuery`, `FindBookingsForVoyageQuery`); the file mirrors the class (`place-booking.command.ts`, `get-booking-by-id.query.ts`). See `tactical-patterns.md` for the stack-agnostic rule.

```typescript
// domain/model/commands/place-booking.command.ts
export class PlaceBookingCommand {
  constructor(
    public readonly customerId: CustomerId,
    public readonly origin: PortCode,
    public readonly destination: PortCode,
    public readonly cargoWeight: CargoWeight,
    public readonly cargoDescription: string,
    public readonly hazardous: boolean,
  ) {
    if (!customerId) throw new Error('customerId required');
    if (!origin || !destination) throw new Error('route required');
  }
}

// domain/model/commands/cancel-booking.command.ts
export class CancelBookingCommand {
  constructor(
    public readonly bookingNumber: BookingNumber,
    public readonly reason: string,
  ) {
    if (!bookingNumber) throw new Error('bookingNumber required');
    if (!reason?.trim()) throw new Error('reason required');
  }
}
```

```typescript
// domain/model/queries/get-booking-by-id.query.ts
export class GetBookingByIdQuery {
  constructor(public readonly bookingNumber: BookingNumber) {}
}

// domain/model/queries/find-bookings-for-voyage.query.ts
export class FindBookingsForVoyageQuery {
  constructor(public readonly voyageNumber: VoyageNumber) {}
}
```

## Command and query services

Split the application layer along the command/query line (CQRS as a *design principle*, not the `@nestjs/cqrs` package — see `tactical-patterns.md`). Declare the **service interfaces in the domain** (`domain/services`) as ports, each with its **injection token**, and **implement them in application**.

### Why every port needs a token

TypeScript interfaces are erased at compile time, so nothing survives for Nest's DI to resolve — every port needs an explicit token.

```typescript
// domain/services/booking-command.service.ts
export const BOOKING_COMMAND_SERVICE = Symbol('BOOKING_COMMAND_SERVICE');

export interface BookingCommandService {
  handle(command: PlaceBookingCommand): Promise<BookingNumber>;
  handle(command: CancelBookingCommand): Promise<void>;
}
```

### Overloaded handle (one name, like Spring)

Both services expose only `handle`, overloaded per command/query — the same contract as `spring-boot.md`. The implementing class repeats every overload signature, then provides a single implementation signature with narrowing. That repetition is the accepted cost of the uniform contract: callers only ever see the precise overloads, never the union implementation signature.

```typescript
// domain/services/booking-query.service.ts
export const BOOKING_QUERY_SERVICE = Symbol('BOOKING_QUERY_SERVICE');

export interface BookingQueryService {
  handle(query: GetBookingByIdQuery): Promise<Booking | null>;
  handle(query: FindBookingsForVoyageQuery): Promise<Booking[]>;
}
```

### Command service — orchestrates, holds no rules

```typescript
// application/internal/commandservices/booking-command.service.impl.ts
@Injectable()
export class BookingCommandServiceImpl implements BookingCommandService {
  constructor(
    @Inject(BOOKING_REPOSITORY) private readonly bookings: BookingRepository,
    @Inject(UNIT_OF_WORK) private readonly unitOfWork: IUnitOfWork,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async handle(command: PlaceBookingCommand): Promise<BookingNumber>;
  async handle(command: CancelBookingCommand): Promise<void>;
  async handle(command: PlaceBookingCommand | CancelBookingCommand): Promise<BookingNumber | void> {
    if (command instanceof CancelBookingCommand) {
      const booking = await this.unitOfWork.run(async () => {
        const b = await this.bookings.findByBookingNumber(command.bookingNumber);
        if (!b) throw new BookingNotFoundError(command.bookingNumber);
        b.cancel(command.reason);
        await this.bookings.save(b);
        return b;
      });
      this.publishDomainEvents(booking);
      return;
    }
    // The whole write runs inside one transaction, so the save and (later) the
    // event publish are tied to a single commit — the same guarantee Spring's
    // @Transactional + AbstractAggregateRoot gives you for free.
    const booking = await this.unitOfWork.run(async () => {
      const b = Booking.place({
        customerId: command.customerId,
        origin: command.origin,
        destination: command.destination,
        cargo: new Cargo(CargoId.generate(), command.cargoWeight, command.cargoDescription, command.hazardous),
      });
      await this.bookings.save(b);
      return b;
    });
    // run() resolves only after commit, so this truly is "publish after commit".
    this.publishDomainEvents(booking);
    return booking.bookingNumber;
  }

  private publishDomainEvents(booking: Booking): void {
    for (const event of booking.pullDomainEvents()) {
      this.eventEmitter.emit(event.eventName, event);
    }
  }
}
```

**Why `run()` instead of `start()` / `complete()`.** ASP.NET's `CompleteAsync()` relies on EF Core deferring writes to `SaveChangesAsync()`; TypeORM has no deferred context, so `save()` commits immediately. The AsyncLocalStorage-scoped `run()` is the TypeORM equivalent — everything inside shares one `EntityManager`, and `run()` commits when the callback resolves or rolls back on throw. The `typeorm-transactional` package (or `@nestjs-cls`) offers the two-phase `start()`/`complete()` style if you prefer it.

### Query service — reads, no mutations

```typescript
// application/internal/queryservices/booking-query.service.impl.ts
@Injectable()
export class BookingQueryServiceImpl implements BookingQueryService {
  constructor(@Inject(BOOKING_REPOSITORY) private readonly bookings: BookingRepository) {}

  async handle(query: GetBookingByIdQuery): Promise<Booking | null>;
  async handle(query: FindBookingsForVoyageQuery): Promise<Booking[]>;
  async handle(query: GetBookingByIdQuery | FindBookingsForVoyageQuery): Promise<Booking | null | Booking[]> {
    if (query instanceof FindBookingsForVoyageQuery) {
      return this.bookings.findConfirmedForVoyage(query.voyageNumber);
    }
    return this.bookings.findByBookingNumber(query.bookingNumber);
  }
}
```

Wire both to their tokens in the module:

```typescript
// booking.module.ts
providers: [
  { provide: BOOKING_COMMAND_SERVICE, useClass: BookingCommandServiceImpl },
  { provide: BOOKING_QUERY_SERVICE, useClass: BookingQueryServiceImpl },
]
```

### Trade-off: the controller re-reads after every write

Splitting command/query services means a write endpoint that returns the created resource pays for an extra round-trip: `handle()` (write) followed by a query to rebuild the response (see [Interfaces (REST)](#interfaces-rest)). That's an accepted cost of the split for most CRUD-shaped endpoints. When it genuinely matters — a hot write path, or a command that already has every field the response needs — have the command service return a small response-shaped value straight from the aggregate it just saved, and skip the extra query for that one endpoint. Don't do this by default; it re-blurs the command/query line the split exists to keep clean.

## Repositories

One repository port per aggregate root, declared in `domain/repositories/` with its token; implemented in `infrastructure/persistence/typeorm/adapters/`.

```typescript
// domain/repositories/booking.repository.ts
export const BOOKING_REPOSITORY = Symbol('BOOKING_REPOSITORY');

export interface BookingRepository {
  findByBookingNumber(bookingNumber: BookingNumber): Promise<Booking | null>;
  findConfirmedForVoyage(voyageNumber: VoyageNumber): Promise<Booking[]>;
  save(booking: Booking): Promise<void>;
}
```

### The ORM entity — persistence shape only

`voyageNumber` needs its own column: a finder can only filter on what's actually persisted as a queryable field, not on data buried inside a JSON blob.

```typescript
// infrastructure/persistence/typeorm/entities/booking.orm-entity.ts
@Entity('bookings')
@Index(['voyageNumber'])
export class BookingOrmEntity extends AuditableOrmEntity {
  @PrimaryColumn() bookingNumber: string;
  @Column() customerId: string;
  @Column() origin: string;
  @Column() destination: string;
  @Column() status: string;
  @Column({ nullable: true }) voyageNumber: string | null;
  @Column('jsonb') cargo: { id: string; amount: number; unit: string; description: string; hazardous: boolean };
}
```

### The assembler — orm-entity ⇄ domain aggregate

Parse persisted strings into union sets through an explicit function (`parseWeightUnit`, and the equivalent `parseBookingStatus` below) rather than an `as` cast — a cast lets a corrupted or manually-edited DB row silently pass through as a valid member; a parser throws where the corruption actually happened, not three call frames later.

```typescript
// domain/model/aggregates/booking.aggregate.ts (addition, alongside BookingStatus)
export function parseBookingStatus(value: string): BookingStatus {
  if (!(Object.values(BookingStatus) as string[]).includes(value)) throw new Error(`invalid booking status: ${value}`);
  return value as BookingStatus;
}
```

```typescript
// infrastructure/persistence/typeorm/assemblers/booking-persistence.assembler.ts
@Injectable()
export class BookingPersistenceAssembler {
  toDomain(orm: BookingOrmEntity): Booking {
    return Booking.rehydrate({
      bookingNumber: BookingNumber.of(orm.bookingNumber),
      customerId: CustomerId.of(orm.customerId),
      origin: PortCode.of(orm.origin),
      destination: PortCode.of(orm.destination),
      status: parseBookingStatus(orm.status),
      voyageNumber: orm.voyageNumber ? VoyageNumber.of(orm.voyageNumber) : null,
      cargo: new Cargo(
        CargoId.of(orm.cargo.id),
        new CargoWeight(orm.cargo.amount, parseWeightUnit(orm.cargo.unit)),
        orm.cargo.description,
        orm.cargo.hazardous,
      ),
    });
  }

  toOrmEntity(domain: Booking): BookingOrmEntity {
    const orm = new BookingOrmEntity();
    orm.bookingNumber = domain.bookingNumber.value;
    orm.customerId = domain.customerId.value;
    orm.origin = domain.origin.value;
    orm.destination = domain.destination.value;
    orm.status = domain.status;
    orm.voyageNumber = domain.voyageNumber?.value ?? null;
    orm.cargo = {
      id: domain.cargo.id.value,
      amount: domain.cargo.weight.amount,
      unit: domain.cargo.weight.unit,
      description: domain.cargo.description,
      hazardous: domain.cargo.hazardous,
    };
    return orm;
  }
}
```

Note that `toOrmEntity` no longer takes a `voyageNumber` parameter from the caller: since `voyageNumber` now lives on `Booking` itself (set by `confirm()`), the assembler reads it straight off the aggregate, exactly like every other field. This is what "the ORM entity is the persistence shape of the aggregate" is supposed to mean — nothing enters the row that didn't come from the domain object being saved.

### The adapter — implements the port

Because `voyageNumber` is sourced from the aggregate rather than reconstructed by the repository, `save()` is a straight translate-and-persist — no read-before-write, no risk of clobbering a concurrent update to a field the aggregate itself doesn't know about.

```typescript
// infrastructure/persistence/typeorm/adapters/booking-repository.impl.ts
@Injectable()
export class BookingRepositoryImpl implements BookingRepository {
  constructor(
    @InjectRepository(BookingOrmEntity) private readonly orm: Repository<BookingOrmEntity>,
    private readonly assembler: BookingPersistenceAssembler,
  ) {}

  // Reads that must be transactionally consistent with a write run inside the
  // use-case should use the ambient manager; outside a transaction the injected
  // repository is fine. This getter keeps the adapter behind the port.
  private get repo(): Repository<BookingOrmEntity> {
    return TransactionContext.current ?? this.orm;
  }

  async findByBookingNumber(bookingNumber: BookingNumber): Promise<Booking | null> {
    const found = await this.repo.findOneBy({ bookingNumber: bookingNumber.value });
    return found ? this.assembler.toDomain(found) : null;
  }

  async findConfirmedForVoyage(voyageNumber: VoyageNumber): Promise<Booking[]> {
    const found = await this.repo.findBy({ voyageNumber: voyageNumber.value, status: BookingStatus.CONFIRMED });
    return found.map((f) => this.assembler.toDomain(f));
  }

  async save(booking: Booking): Promise<void> {
    await this.repo.save(this.assembler.toOrmEntity(booking));
  }
}
```

### Why there's no extra "persistence repository" layer

Spring Boot's `JpaRepository<Booking, Long>` is an interface Spring Data auto-implements at runtime — that generated implementation is what the adapter delegates to. TypeORM has no equivalent auto-implementation step: `@InjectRepository(BookingOrmEntity)` already gives you a working, generic `Repository<BookingOrmEntity>` directly. The adapter above *is* both the "repository implementation" and the "domain-port satisfier" in one class — there's no separate generated-interface layer to wire in between.

```typescript
// booking.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([BookingOrmEntity])],
  providers: [
    BookingPersistenceAssembler,
    { provide: BOOKING_REPOSITORY, useClass: BookingRepositoryImpl },
    { provide: UNIT_OF_WORK, useClass: TypeOrmUnitOfWork },
  ],
})
export class BookingModule {}
```

### Persisting a composite aggregate (children in their own table)

The Composite `Shipment` from [Entities and aggregate roots](#entities-and-aggregate-roots) owns many children with identity and behavior, so its persistence shape is a **parent table + a child table** — the analog of Spring's `@ElementCollection` + `@CollectionTable`. Give the child its own ORM entity and a back-reference to the parent:

```typescript
// infrastructure/persistence/typeorm/entities/shipment.orm-entity.ts
@Entity('shipments')
export class ShipmentOrmEntity extends AuditableOrmEntity {
  @PrimaryColumn() id: string;
  @Column() status: string;
  @OneToMany(() => CargoItemOrmEntity, (cargo) => cargo.shipment, { cascade: true })
  cargo: CargoItemOrmEntity[];
}
```

```typescript
// infrastructure/persistence/typeorm/entities/cargo-item.orm-entity.ts
@Entity('cargo_items')
export class CargoItemOrmEntity extends AuditableOrmEntity {
  @PrimaryColumn() id: string;
  @Column() reference: string;
  @Column('float') weight: number;
  @Column() weightUnit: string;
  @Column('float', { nullable: true }) temperature: number | null;
  @Column() kind: 'container' | 'bulk' | 'reefer';
  @Column() status: string;

  @ManyToOne(() => ShipmentOrmEntity, (shipment) => shipment.cargo, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'shipment_id' })
  shipment: ShipmentOrmEntity;
}
```

The persistence assembler translates `Shipment.cargo` → `CargoItemOrmEntity[]` and back — same pattern as `BookingPersistenceAssembler`, with the root's `status` and each child's `status` parsed through `parseShipmentStatus()` / `parseCargoStatus()` (never `as`-cast). Both follow the same `Object.values()` guard as `parseBookingStatus()`:

```typescript
// domain/model/aggregates/shipment.aggregate.ts (addition, alongside ShipmentStatus)
export function parseShipmentStatus(value: string): ShipmentStatus {
  if (!(Object.values(ShipmentStatus) as string[]).includes(value)) throw new Error(`invalid shipment status: ${value}`);
  return value as ShipmentStatus;
}
```

```typescript
// domain/model/entities/cargo-item.entity.ts (addition, alongside CargoStatus)
export function parseCargoStatus(value: string): CargoStatus {
  if (!(Object.values(CargoStatus) as string[]).includes(value)) throw new Error(`invalid cargo status: ${value}`);
  return value as CargoStatus;
}
```

**JSONB is the exception, not the default.** `Booking` stores its single `Cargo` as a `jsonb` column because it's effectively a value-object-shaped detail: one per booking, no per-child identity, no separate querying need. That's fine. The moment children have identity, behavior, or need to be filtered on (`findConfirmedForVoyage` filters on real columns), a `jsonb` blob stops working: it isn't queryable or indexable per-child, and it hides the aggregate's shape from the database. The trade-off plain and simple:

| Children need... | Use |
|---|---|
| No identity, always loaded with the root, never queried individually | `jsonb` (or an embedded/`@Column` shape) |
| Own identity + lifecycle, queried/aggregated over | Separate child table (`@OneToMany`/`@ManyToOne`) |

### When to skip the port

**Why the default differs from Spring.** Spring can keep JPA on the aggregate root because JPA is metadata-only, so `@Entity` IS the domain model and a Spring Data interface is a first-class abstraction — direct injection is genuinely the clean baseline. TypeORM's decorators instead carry runtime relation and lazy-loading behavior, so Nest keeps the domain aggregate as a plain class and the ORM entity as a separate class; something must translate between them, and that translation belongs behind a port. Hence the port + assembler + adapter is the Nest default, not an optional layer.

Given that, skipping the port is a genuine simplification, not the baseline. It's reasonable when:

1. **No business rules in the repository.** It only offers CRUD finders and a straight save — no domain logic to protect.
2. **No swappability or test boundary.** One persistence technology, and the test double would add no coverage beyond the real adapter.
3. **The module is small and low-invariant** — the domain aggregate is little more than data.

Introduce the port the moment the module has finders named in the ubiquitous language, or once a second implementation (e.g., an in-memory test double) earns its keep.

### Test: repository round-trip

```typescript
describe('BookingRepositoryImpl', () => {
  it('saves and reloads an equivalent aggregate', async () => {
    const booking = Booking.place(placedBookingInput());
    await repository.save(booking);

    const reloaded = await repository.findByBookingNumber(booking.bookingNumber);

    expect(reloaded?.bookingNumber.equals(booking.bookingNumber)).toBe(true);
  });

  it('finds only confirmed bookings for the given voyage', async () => {
    // confirmedBookingFor(voyage) = Booking.place(...) followed by .confirm(...)
    // with a RouteProposal whose vessel is VoyageNumber.of(voyage) — voyageNumber comes from
    // the aggregate's own behavior, never injected by the test at the repo layer.
    await repository.save(confirmedBookingFor('VOY-1'));
    await repository.save(placedBookingFor('VOY-1'));   // not confirmed — excluded
    await repository.save(confirmedBookingFor('VOY-2')); // wrong voyage — excluded

    const found = await repository.findConfirmedForVoyage(VoyageNumber.of('VOY-1'));

    expect(found).toHaveLength(1);
    expect(found[0].voyageNumber?.value).toBe('VOY-1');
  });
});
```

## Outbound services and dependency injection with tokens

When a bounded context calls an external technology (hashing, tokens, payments, LLMs, email), define a **technology-agnostic port interface** in `application/internal/outboundservices/{concept}/`, with its token, and implement it in `infrastructure/{technology}/{implementation}/`.

Because every interface needs a token regardless of whether there's DI ambiguity, NestJS has **no equivalent to Spring's Marker Interface pattern** — that pattern exists specifically to resolve ambiguity when a framework can auto-wire by type and two contracts collide. Nest never auto-wires by type, so there's nothing to disambiguate: the token *is* the mechanism, always, with zero extra ceremony when a class happens to also satisfy some third-party interface.

### CargoRoute example: hashing service

```typescript
// application/internal/outboundservices/hashing/hashing.service.ts
export const HASHING_SERVICE = Symbol('HASHING_SERVICE');

export interface HashingService {
  hash(rawValue: string): Promise<string>;
  matches(rawValue: string, hashed: string): Promise<boolean>;
}
```

```typescript
// infrastructure/hashing/bcrypt/bcrypt-hashing.service.ts
@Injectable()
export class BcryptHashingService implements HashingService {
  async hash(rawValue: string): Promise<string> {
    return bcrypt.hash(rawValue, 10);
  }
  async matches(rawValue: string, hashed: string): Promise<boolean> {
    return bcrypt.compare(rawValue, hashed);
  }
}
```

```typescript
// booking.module.ts
providers: [{ provide: HASHING_SERVICE, useClass: BcryptHashingService }]

// consumer
constructor(@Inject(HASHING_SERVICE) private readonly hashing: HashingService) {}
```

The application layer depends on `HashingService` (the port) and the `HASHING_SERVICE` token — never on `bcrypt` directly. Swapping to Argon2 means adding `infrastructure/hashing/argon2/argon2-hashing.service.ts` and changing one line in the module's `providers` array — no application code changes.

**When to skip:** if the external dependency is a thin, unlikely-to-change utility, injecting the concrete class directly (still via its own class as the "token", since Nest can inject concrete classes without a `provide` key) is fine. The port earns its cost when the technology is likely to change or when testing needs a mock boundary.

## Anti-corruption layer

When this context needs something from **another** bounded context, do not import its model. The provider exposes a **facade interface**; the consumer translates the result into its own value objects.

Keep the facade signature itself in primitives, not in the consumer's VOs. `VesselSchedulingContextFacade` below is *owned and exposed by Vessel Scheduling* (see its `exports: [VESSEL_SCHEDULING_FACADE]` in [Bidirectional ACL in a monolith](#bidirectional-acl-in-a-monolith)) — if its signature were typed with `PortCode`/`CargoWeight`/`VoyageNumber`, Vessel Scheduling would have to import Booking's domain VOs just to declare its own interface, which is exactly the reversed dependency the ACL exists to prevent. Translation into VOs is the **consumer's** job — that's what "the consumer translates the result into its own value objects" means literally — so it happens once, inside `ExternalVesselService`, not at the facade boundary:

```typescript
// vessel-scheduling/interfaces/acl/vessel-scheduling-context.facade.ts
export const VESSEL_SCHEDULING_FACADE = Symbol('VESSEL_SCHEDULING_FACADE');

export interface VesselSchedulingContextFacade {
  findViableVoyage(origin: string, destination: string, weightTonnes: number): Promise<string | null>;
}
```

```typescript
// booking/application/internal/outboundservices/acl/external-vessel.service.ts
@Injectable()
export class ExternalVesselService {
  constructor(@Inject(VESSEL_SCHEDULING_FACADE) private readonly scheduling: VesselSchedulingContextFacade) {}

  async proposeRoute(origin: PortCode, destination: PortCode, weight: CargoWeight): Promise<RouteProposal | null> {
    const voyage = await this.scheduling.findViableVoyage(origin.value, destination.value, weight.amount);
    // This is the translation the ACL exists for: the facade returns a plain
    // string with no knowledge of Booking's domain; ExternalVesselService,
    // as the consumer, is the only place that wraps it into VoyageNumber.
    return voyage ? new RouteProposal(VoyageNumber.of(voyage), this.computeEta(origin, destination)) : null;
  }
}
```

`RouteProposal` itself (its `vessel: VoyageNumber` and `routeId: string` fields, plus `computeEta`) is illustrative rather than fully specified here — the same simplification `spring-boot.md` makes with its own `RouteProposal`. `vessel` is the `VoyageNumber` `ExternalVesselService` just constructed from the facade's raw string; `routeId` is left as a plain string this reference wraps into `RouteId` inside `Booking.confirm()` (see [Entities and aggregate roots](#entities-and-aggregate-roots)), since neither stack reference shows where a routing computation would actually produce a typed `RouteId`.

If a project genuinely wants Booking and Vessel Scheduling to share `PortCode`/`CargoWeight`/`VoyageNumber` on purpose — not by accident of typing a facade signature — that's the **Shared Kernel** context-mapping pattern (`references/domain-modeling.md`): move those VOs to a `shared/domain/model/valueobjects/` folder both modules import, and say so explicitly. Don't back into a shared kernel silently by typing an ACL boundary with one side's VOs.

**When a facade returns a VO instead of primitives.** Primitives are the default — most facades return one or two values, and wrapping a single id/amount is the consumer's job. Use a VO (or a small DTO) at the boundary only when the payload is genuinely large or compound — roughly **6+ fields** the consumer would otherwise reassemble field-by-field — mirroring `spring-boot.md`'s "Cross-context reference data" pattern. When a VO is independently needed by 3+ bounded contexts, promote it to the Shared Kernel (`shared/domain/model/valueobjects/`) rather than typing each boundary with it by accident.

### Outbound vs inbound ACL

- **Outbound (this context consumes):** The consumer defines the facade interface + token in `interfaces/acl/`. The provider implements it in `application/acl/`. The consumer's `External{Bc}Service` (in `application/internal/outboundservices/acl/`) is the only class that injects the facade token. All other application code delegates to it.
- **Inbound (this context exposes):** Define the facade interface + token in `interfaces/acl/`. Implement it in `application/acl/`. Other modules depend on this token — never on your repositories, aggregates, or internal services.

### Bidirectional ACL in a monolith

Two bounded contexts can define facades that the other consumes:

```text
Booking → ExternalVesselService → VESSEL_SCHEDULING_FACADE
VesselScheduling → ExternalBookingService → BOOKING_CONTEXT_FACADE
```

This shape is a genuine circular module dependency — `BookingModule` imports `VesselSchedulingModule` and vice versa — and Nest needs to be told about it explicitly on **both sides**, or the app fails to bootstrap with a circular-dependency error. Use `forwardRef()` in both `imports` and, if the facade is constructor-injected directly rather than only through a provider token, in the `@Inject(forwardRef(() => ...))` at the injection site too:

```typescript
// booking.module.ts
@Module({
  imports: [forwardRef(() => VesselSchedulingModule)],
  exports: [BOOKING_CONTEXT_FACADE],
})
export class BookingModule {}

// vessel-scheduling.module.ts
@Module({
  imports: [forwardRef(() => BookingModule)],
  exports: [VESSEL_SCHEDULING_FACADE],
})
export class VesselSchedulingModule {}
```

Acceptable in a monolith when:

- Dependencies are to interfaces + tokens, not implementations.
- Each module controls its own facade, and `exports` **only** the facade provider — never the whole module's internals.
- Communication is synchronous and in-process.
- The circularity is between exactly two modules and stays that way; a third module joining the cycle is a strong signal to introduce an event-based decoupling instead of a third `forwardRef()`.

In microservices, these facade interfaces become HTTP clients (`HttpModule`) or gRPC clients — the interface and token stay, the adapter changes from in-process to network call, and the circular-module problem disappears with it (each service only imports its own facade implementation).

### Cross-context reference data

Some contexts need **read-only data** owned by another context to enrich their responses (e.g., Booking shows the vessel name; Tracking shows the port address). Apply this pattern:

1. **Provider VO:** The provider defines a simple class/record in `domain/model/valueobjects/` (e.g., `VesselInfo`). If the VO is used by 3+ bounded contexts, place it in `shared/domain/model/valueobjects/`.
2. **Facade returns the VO:** The provider's `XxxContextFacade` returns the VO directly, or returns primitives when only a single field is needed.
3. **Consumer mapping:** The consumer's `ExternalXxxService` (in `application/internal/outboundservices/acl/`) calls the facade. If the consumer needs only a subset of fields, it maps to its own minimal VO in `domain/model/valueobjects/`.
4. **No domain services:** Non-aggregate reference data does NOT get its own service interface in `domain/services/`. The `ExternalXxxService` is an application-layer service, not a domain service.

**Primitives (single field needed):**

```typescript
// Provider: VesselSchedulingFacade
fetchVesselName(voyageNumber: string): Promise<string | null>;

// Consumer: ExternalVesselService
async fetchVesselName(voyageNumber: string): Promise<string> {
  return (await this.scheduling.fetchVesselName(voyageNumber)) ?? '';
}
```

**Provider VO → Consumer minimal VO (multiple fields needed):**

```typescript
// Provider: domain/model/valueobjects/vessel-info.value-object.ts
export class VesselInfo {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly imoNumber: string,
    public readonly active: boolean,
  ) {}
}

// Provider: VesselSchedulingFacade
fetchVesselInfo(vesselId: string): Promise<VesselInfo>;

// Consumer: domain/model/valueobjects/vessel-booking-info.value-object.ts
export class VesselBookingInfo {
  constructor(
    public readonly vesselId: string,
    public readonly name: string,
    public readonly imoNumber: string,
  ) {}
}

// Consumer: ExternalVesselService
async fetchVesselBookingInfo(vesselId: string): Promise<VesselBookingInfo | null> {
  const info = await this.scheduling.fetchVesselInfo(vesselId);
  return info ? new VesselBookingInfo(info.id, info.name, info.imoNumber) : null;
}
```

**Shared VO (used by 3+ contexts):**

```typescript
// shared/domain/model/valueobjects/voyage-context.value-object.ts
export class VoyageContext {
  constructor(
    public readonly voyageNumber: string,
    public readonly vesselId: string,
    public readonly routeId: string,
  ) {}
}

// Provider: VesselSchedulingFacade
fetchVoyageContext(voyageNumber: string): Promise<VoyageContext>;

// Consumer: ExternalVesselService
fetchVoyageContext(voyageNumber: string): Promise<VoyageContext> {
  return this.scheduling.fetchVoyageContext(voyageNumber);
}
```

This keeps a single point of access per external context. When moving to microservices, the `ExternalXxxService` becomes an HTTP client while the VOs stay unchanged.

## Interfaces (REST)

Three parts: **`resources`** (DTOs), **`transform`** (assemblers), and the **controller**.

### Resources — request/response DTOs, validated with `class-validator`

```typescript
// interfaces/rest/resources/place-booking.resource.ts
export class PlaceBookingResource {
  @IsString() @IsNotEmpty() customerId: string;
  @IsString() @IsNotEmpty() origin: string;
  @IsString() @IsNotEmpty() destination: string;
  @IsNumber() cargoWeight: number;
  @IsIn(Object.values(WeightUnit)) weightUnit: WeightUnit;
  @IsString() cargoDescription: string;
  @IsBoolean() hazardous: boolean;
}

// interfaces/rest/resources/booking.resource.ts
export class BookingResource {
  constructor(
    public readonly bookingNumber: string,
    public readonly status: string,
    public readonly origin: string,
    public readonly destination: string,
  ) {}
}
```

`class-validator` decorators never appear on domain value objects or commands — only here. The global pipe (`app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))`, registered once in `main.ts` — see [The shared kernel](#the-shared-kernel)) runs these checks before the controller method executes.

### Transformers — static assemblers, resource ⇄ command/entity

```typescript
// interfaces/rest/transform/place-booking-command-from-resource.assembler.ts
export class PlaceBookingCommandFromResourceAssembler {
  static toCommand(resource: PlaceBookingResource): PlaceBookingCommand {
    return new PlaceBookingCommand(
      CustomerId.of(resource.customerId),
      PortCode.of(resource.origin),
      PortCode.of(resource.destination),
      new CargoWeight(resource.cargoWeight, resource.weightUnit),
      resource.cargoDescription,
      resource.hazardous,
    );
  }
}

// interfaces/rest/transform/booking-resource-from-entity.assembler.ts
export class BookingResourceFromEntityAssembler {
  static toResource(booking: Booking): BookingResource {
    return new BookingResource(booking.bookingNumber.value, booking.status, booking.origin.value, booking.destination.value);
  }
}
```

### The controller — thin orchestrator

Resource → assembler → command → service (by token) → id → query → entity → assembler → resource. (See [the trade-off note above](#trade-off-the-controller-re-reads-after-every-write) on the extra query this pattern costs per write.)

```typescript
// interfaces/rest/booking.controller.ts
@UseFilters(BookingExceptionFilter)
@Controller('bookings')
export class BookingController {
  constructor(
    @Inject(BOOKING_COMMAND_SERVICE) private readonly commands: BookingCommandService,
    @Inject(BOOKING_QUERY_SERVICE) private readonly queries: BookingQueryService,
  ) {}

  @Post()
  async placeBooking(@Body() resource: PlaceBookingResource): Promise<BookingResource> {
    const command = PlaceBookingCommandFromResourceAssembler.toCommand(resource);
    const bookingNumber = await this.commands.handle(command);
    const booking = await this.queries.handle(new GetBookingByIdQuery(bookingNumber));
    if (!booking) throw new NotFoundException();
    return BookingResourceFromEntityAssembler.toResource(booking);
  }

  @Get(':number')
  async getBooking(@Param('number') number: string): Promise<BookingResource> {
    const booking = await this.queries.handle(new GetBookingByIdQuery(BookingNumber.of(number)));
    if (!booking) throw new NotFoundException();
    return BookingResourceFromEntityAssembler.toResource(booking);
  }
}
```

## Domain exceptions and error handling

Express failures in the **ubiquitous language**. Define domain-specific exceptions in `domain/exceptions` as plain `Error` subclasses — no HTTP status codes inside them, keeping the domain pure. When compiling to an ES5 target, restore the prototype chain explicitly, or `instanceof` checks against these classes (which the exception filter below relies on) can silently fail:

```typescript
// domain/exceptions/booking-not-found.error.ts
export class BookingNotFoundError extends Error {
  constructor(bookingNumber: BookingNumber) {
    super(`Booking ${bookingNumber.value} not found`);
    this.name = 'BookingNotFoundError';
    Object.setPrototypeOf(this, BookingNotFoundError.prototype); // safe no-op on ES2015+ targets
  }
}

// domain/exceptions/illegal-booking-state.error.ts
export class IllegalBookingStateError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'IllegalBookingStateError';
    Object.setPrototypeOf(this, IllegalBookingStateError.prototype);
  }
}

// domain/exceptions/route-not-feasible.error.ts
export class RouteNotFeasibleError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'RouteNotFeasibleError';
    Object.setPrototypeOf(this, RouteNotFeasibleError.prototype);
  }
}
```

If `tsconfig.json` already targets `ES2015` or later, `Object.setPrototypeOf` is a redundant safety net rather than a fix — keep it anyway, since it costs nothing and protects the code against a future target downgrade.

Translate at the edge with an `ExceptionFilter`. Every domain exception the aggregate can throw for this context needs an entry here — an exception raised in the domain but missing from `@Catch(...)` falls through to the global filter as a generic, uninformative 500:

```typescript
// interfaces/rest/filters/booking-exception.filter.ts
@Catch(BookingNotFoundError, IllegalBookingStateError, RouteNotFeasibleError)
export class BookingExceptionFilter implements ExceptionFilter {
  catch(exception: Error, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse<Response>();
    const status = this.statusFor(exception);
    response.status(status).json({ message: exception.message, statusCode: status });
  }

  private statusFor(exception: Error): number {
    if (exception instanceof BookingNotFoundError) return HttpStatus.NOT_FOUND;
    if (exception instanceof RouteNotFeasibleError) return HttpStatus.UNPROCESSABLE_ENTITY;
    return HttpStatus.CONFLICT; // IllegalBookingStateError
  }
}
```

### BC-scoped exception handling

A **global** filter in `shared/` (see [The shared kernel](#the-shared-kernel)) maps common cases (`NotFoundException`, `BadRequestException`, unhandled errors). A **BC-specific** filter, applied with `@UseFilters()` on the controller (as above) or registered per-module, maps exceptions unique to that context.

**Rules:**

- Never redefine shared-kernel exceptions in a bounded context.
- BC-scoped filters only catch that context's own `domain/exceptions` classes.
- Only create BC-specific exceptions when the semantic meaning differs from an existing shared one.
- Every exception type an aggregate can raise must appear in the matching `@Catch(...)` list — treat a new domain exception and its filter entry as one change, not two.

## Identity and persistence: preferences

- **Identity (aggregate roots).** Two approaches, in order of recommendation:
  1. **Typed id VO with application-side generation** — `BookingNumber.generate()` produces `"BKG-" + UUID` at construction time in the aggregate's factory, mapped as `@PrimaryColumn()` (not `@PrimaryGeneratedColumn()`) on the ORM entity. Keeps identity type-safe and avoids a round-trip to read back a database-generated key.
  2. **Database-generated `uuid`** — `@PrimaryGeneratedColumn('uuid')` on the ORM entity, read back after `save()` and wrapped in the typed VO before returning from the command service. Simpler, costs one extra field on the returned aggregate.
  Prefer (1) for identifiers that are meaningful in the ubiquitous language (a booking number customers reference); (2) is fine for internal entities with no business-facing identity.

- **Identity (internal entities).** The aggregate root assigns a generated id via the entity's own VO factory (`CargoId.generate()`), same reasoning as Spring's "Explicit ID" pattern — keeps identity domain-driven, no DB round-trip, and consistent with wrapping every identifier in a VO (see [Value objects](#value-objects)).

- **Repository.** `TypeOrmRepository<OrmEntity>` injected inside the adapter is simplest; add the domain port + assembler once the module has real ubiquitous-language finders or needs a swappable/in-memory implementation for tests.

- **TypeORM in the domain.** Not recommended — and for the same reason that makes the repository port the default here (see [Repositories](#repositories)). TypeORM decorators carry runtime relation and lazy-loading behavior, so the domain aggregate stays a plain class and the ORM entity a separate class; the split is the default, not the "maximum isolation" option.

- **Parsing persisted union sets.** Always through an explicit `parseX()` function (see [Repositories](#repositories)), never an `as` cast — the assembler is the one place untrusted storage data re-enters the domain, so it's the one place validation can't be skipped.

Non-negotiable: business rules and invariants stay in the domain, and the domain never depends on `interfaces`, `application`, or `infrastructure`.

## A note on `@nestjs/cqrs`

`@nestjs/cqrs` provides `CommandBus`, `QueryBus`, `EventBus`, and an `AggregateRoot` base with `apply()`/`mergeObjectContext()`. It is **not part of Nest's core** and is not a de facto standard — plenty of DDD codebases in Nest skip it and use the explicit-service pattern shown throughout this file.

This reference does **not** use it by default: the bus hides which handler answers a command behind `@CommandHandler(X)` + a runtime dispatch, and its `AggregateRoot`/`IEvent`/`ICommand` types pull `@nestjs/cqrs` into `domain/`, which the rest of this skill avoids.

If a project already commits to it — often because it also wants Event Sourcing, where it genuinely reduces boilerplate — the mapping is:

| This reference | `@nestjs/cqrs` |
|---|---|
| `BookingCommandService.handle(command)` (explicit interface + token) | `CommandBus.execute(command)` + `@CommandHandler(PlaceBookingCommand)` |
| `BookingQueryService.handle(query)` | `QueryBus.execute(query)` + `@QueryHandler(GetBookingByIdQuery)` |
| `booking.pullDomainEvents()` + manual `EventEmitter2.emit()` after the unit of work commits | `Booking extends AggregateRoot`, `this.apply(event)`, `publisher.mergeObjectContext(booking)`, `booking.commit()` |
| `@OnEvent(BookingConfirmed.eventName)` | `@EventsHandler(BookingConfirmed)` implementing `IEventHandler` |

Adopt it deliberately, as a team decision with the coupling trade-off stated explicitly — not as the default idiom for "doing CQRS in Nest."

## Testing each layer

Match the test style to what the layer actually does:

- **`domain`** — plain Jest unit tests, no `Test.createTestingModule`. Construct the aggregate via `Booking.place(...)`, call behavior, assert on state/exceptions/pulled events. Should run in milliseconds.
- **`application`** — plain Jest tests with a hand-built mock satisfying the repository/port interface (no DI container needed — just pass the mock into the constructor), or `Test.createTestingModule` with `overrideProvider(BOOKING_REPOSITORY)` when you want Nest's DI wiring exercised too.
- **`infrastructure`** — integration tests against a real Postgres via Testcontainers (preferred over sqlite for TypeORM, since column types and constraints diverge): confirm mappings, typed-id round-trips, union-set parsing on corrupt/legacy data, and finders (including ones that filter on real columns, like `findConfirmedForVoyage`).
- **`interfaces`** — `Test.createTestingModule` + `supertest` against the compiled Nest app, or a sliced test that mocks the command/query service tokens and calls the controller method directly for faster, non-HTTP tests.

A domain test that needs `Test.createTestingModule` to pass is usually a sign business logic leaked into an `@Injectable()`.

## Common pitfalls

- **Injecting an interface without a token.** `constructor(private repo: BookingRepository)` compiles, then throws `Nest can't resolve dependencies` at boot, because the interface doesn't exist at runtime. Every port needs its `Symbol`/token and an explicit `@Inject()`.
- **Decorating the aggregate root with TypeORM decorators "to save a class."** Reintroduces the anemic-model risk this reference exists to avoid — see [Entities and aggregate roots](#entities-and-aggregate-roots).
- **Publishing domain events before the unit of work commits**, or in a `try` block that still runs on the `catch` path. Bypasses the "publish only after commit" guarantee — `publishDomainEvents()` goes after `unitOfWork.run(...)` resolves, never inside it.
- **A write that bypasses the unit of work.** Ordering `save()` then `complete()` only gives you one transaction if every statement shares the transaction's `EntityManager`. TypeORM has no deferred context, so repository adapters must read `TransactionContext.current` (or accept the manager) inside `run()`; otherwise `complete()` has nothing to commit and cross-aggregate atomicity silently disappears.
- **Calling `EventEmitterModule.forRoot()` from more than one module.** It configures global state; call it once, in the app root, and let feature modules just inject `EventEmitter2`.
- **Emitting/subscribing to events by `event.constructor.name`.** Not guaranteed stable under minification, and won't type-check against a `DomainEvent[]` under `strict: true`. Use a static `eventName` on the event class instead.
- **A repository finder that ignores its own filter parameter.** If `findConfirmedForVoyage(voyageNumber)` doesn't actually filter on `voyageNumber` in both the ORM entity and the query, it's a silent correctness bug waiting for production data volume to expose it.
- **An ORM column with no backing field on the aggregate.** If `save()` needs a read-before-write (or any other side-channel) just to avoid losing a column's value, that's a sign the column represents domain state that belongs on the aggregate, not persistence-only metadata. Add the field to the aggregate — as `Booking._voyageNumber` shows — instead of patching around its absence in the repository.
- **Using a TS `enum` for a closed set.** Prefer an `as const` object + type union (see [Value objects](#value-objects)) — `enum` is non-erasable and breaks under `isolatedModules` / `erasableSyntaxOnly`.
- **`as` casting a persisted string into a domain union set.** Skips validation exactly where untrusted data re-enters the domain. Parse explicitly and throw on an unrecognized value.
- **Holding a composite's children in a `jsonb` blob by default.** Children with identity, lifecycle, or queryable fields belong in a child table (`@OneToMany`/`@ManyToOne`). JSONB isn't queryable or indexable per child and hides the aggregate's shape; reserve it for a genuinely value-object-shaped single detail, like `Booking`'s one `Cargo`.
- **Two-way `forwardRef()`-free circular module imports.** Two modules importing each other directly (for a bidirectional ACL or otherwise) fails at bootstrap without `forwardRef()` on both sides.
- **A module importing another module's `TypeOrmModule.forFeature([...])` or repository token directly** instead of going through its facade. Exactly what the ACL exists to prevent.
- **Typing a facade interface with the consumer's value objects instead of primitives.** If the provider's own module exports the facade token (see [Bidirectional ACL](#bidirectional-acl-in-a-monolith)), a signature like `findViableVoyage(origin: PortCode, ...)` forces the provider to import the consumer's domain just to declare its own interface — the reversed dependency the ACL exists to prevent, and a silent, undocumented Shared Kernel. Keep the facade in primitives; let the consumer's `External{Bc}Service` do the translation into VOs, exactly as the section's own opening sentence says.
- **Another bounded context's module subscribing directly to a `DomainEvent` via `@OnEvent`.** `EventEmitter2` is a single, app-wide bus with no module isolation, so nothing stops it at the framework level — but it forces the subscribing module to import the emitting context's event class and VOs. Keep `@OnEvent(SomeDomainEvent.eventName)` handlers inside the emitting module only; if another context needs to react, that in-module handler calls the other context's facade — see [Domain events never subscribed to from another module](#domain-events-never-subscribed-to-from-another-module).
- **Business rules inside a `*ServiceImpl` class.** If a command service contains an `if` deciding whether an operation is *allowed*, it belongs inside the aggregate.
- **Fat controllers** that build responses by hand from multiple service calls with conditional logic, instead of delegating orchestration to the command/query service.
- **A domain exception with no matching `@Catch(...)` entry.** Falls through to the global filter as an undifferentiated 500 instead of the meaningful HTTP status the ubiquitous language implies.
- **Reaching for `@nestjs/cqrs` by default** because it "is the NestJS way to do CQRS" — it's one option, with a real coupling cost; see the note above.

## Quick reference

| DDD concept | NestJS idiom |
| --- | --- |
| Entity (internal) | Plain class in `domain/model/entities/`, id assigned via its own VO factory (`CargoId.generate()`), behavior methods |
| Aggregate Root | Plain class extending `AggregateRoot` (in `domain/model/aggregates/`), no decorators, private constructor with `place()`/`rehydrate()` static factories, event drain inherited from the base, every field a behavior method can set (e.g. `voyageNumber` set by `confirm()`) lives on the aggregate — never only on the ORM entity |
| Composite aggregate collection | Root owns a collection of children with identity/behavior, builds them via create-methods (`addContainer` with dedup), derives whole-state (`isReadyForLoading()`, `confirm()`) from the children — see `design-patterns-arch-patterns.md`; persisted as a parent + child table (`@OneToMany`/`@ManyToOne`), not a `jsonb` blob |
| Value Object | Plain class, `private readonly`/`readonly` fields, validated in constructor, `equals()` |
| Typed identifier | Plain class wrapping the raw value, validated in constructor — applied to every identifier, including internal-entity ids |
| Domain Event | Plain class in `domain/model/events/` implementing `DomainEvent`, named `Target + PastAction` (`BookingConfirmed`; `Event` suffix only on collision), with a static `eventName`, payload fields typed as VOs (never bare strings for identifiers), published via `EventEmitter2` after the unit of work commits — never subscribed to from another bounded context's module |
| Event handler | `@Injectable()` `<EventName>EventHandler` with `@OnEvent(EventClass.eventName)` in `application/internal/eventhandlers/` |
| Command / Query | Plain class in `domain/model/commands` or `queries`, validated in constructor, named `Action + Target + Command` / `Action + Target + Criteria + Query` (`PlaceBookingCommand`, `GetBookingByIdQuery`) |
| Command/Query Service | Interface + token in `domain/services`, overloaded `handle` per command/query (Spring-style), impl in `application/internal/...` |
| Repository (port) | Interface + `Symbol` token in `domain/repositories/`, finders backed by real, filterable columns |
| Repository (adapter) | `@Injectable()` class in `infrastructure/persistence/typeorm/adapters/`, injects `Repository<OrmEntity>` via `@InjectRepository()` |
| Unit of work | `IUnitOfWork` port + `UNIT_OF_WORK` token in `shared/domain/repositories/`, `TypeOrmUnitOfWork` in `infrastructure/.../repositories/`; writes run inside `unitOfWork.run()`, events publish after it resolves |
| ORM entity | `@Entity()` class in `infrastructure/persistence/typeorm/entities/`, persistence shape only |
| Assembler | Class in `infrastructure/persistence/typeorm/assemblers/`, orm-entity ⇄ domain aggregate, union-set fields parsed via `parseX()`, never `as`-cast |
| Anti-corruption layer | Facade interface + token (`interfaces/acl`), signature in primitives — the *provider's* language, never the consumer's VOs; the consumer's `External{Bc}Service` (`outboundservices/acl/`) is the only place that translates into its own VOs; bidirectional ACL needs `forwardRef()` on both module imports |
| Outbound service (tech port) | Interface + token in `outboundservices/{concept}/`, adapter in `infrastructure/{tech}/{impl}/` |
| Marker interface (Spring concept) | **Not needed** — Nest always resolves by token, never by type, so there's no ambiguity to disambiguate |
| Domain exception | `Error` subclass in `domain/exceptions` with `Object.setPrototypeOf` restored, mapped by an `ExceptionFilter` covering every exception the aggregate can raise |
| BC-scoped exception handling | `@UseFilters(BookingExceptionFilter)` on the controller, filter in `{context}/interfaces/rest/filters/` |
| Resource / DTO | Class with `class-validator` decorators in `interfaces/rest/resources/` |
| Global input validation / exception handling | `app.useGlobalPipes(...)` and `app.useGlobalFilters(...)`, registered once in `main.ts` |
