# DDD in Angular (frontend)

Read this when structuring an **Angular** app around a domain. Start with the honest part below — DDD on the frontend is *adapted*, not the same as on the backend — and then apply the structure and idioms that follow. Examples use the **CargoRoute** freight domain (a `tracking` feature). Idioms follow Angular 20+ (standalone components, signals, `inject()`) and apply on Angular 21, 22, and 23.

> **Angular 22+ note.** Everything here stays supported; newer versions only *add* modern alternatives you can opt into: **`@Service()`** (shorter root-provided), **Signal Forms**, and **`resource()` / `httpResource()`** replacing manual `subscribe` in stores. Signals pair naturally with OnPush — the default change detection from Angular 22+.

## Contents

- [The honest truth about frontend DDD](#the-honest-truth-about-frontend-ddd)
- [Folder structure: bounded contexts and four layers](#folder-structure-bounded-contexts-and-four-layers)
- [The shared kernel](#the-shared-kernel)
- [The domain layer: entities (and commands)](#the-domain-layer-entities-and-commands)
- [Infrastructure: DTOs, assemblers, endpoints, and the context API](#infrastructure-dtos-assemblers-endpoints-and-the-context-api)
- [The application layer: a signal store](#the-application-layer-a-signal-store)
- [The presentation layer: views and components](#the-presentation-layer-views-and-components)
- [Routing](#routing)
- [Reactive forms and writes](#reactive-forms-and-writes)
- [Strategic design on the frontend](#strategic-design-on-the-frontend)
- [What carries over, loosens, or doesn't apply](#what-carries-over-loosens-or-doesnt-apply)

## The honest truth about frontend DDD

The backend is the **system of record**: it owns the business rules, the invariants, and the transactional consistency. The frontend cannot enforce those — a determined user bypasses any client-side check — so it should not pretend to.

**CargoRoute reality check:** When a vessel is at capacity, the backend rejects the rebooking. Our first tracking UI showed the booking as "confirmed" for 3 seconds before the backend error arrived. Customers called support in that window. We moved the optimistic update into the store with rollback-on-error, so the UI now reflects backend truth with at most a 1-frame flicker.

What the frontend *does* gain from DDD is **structure**: organizing the app by the domain (not by technical type), speaking the same **ubiquitous language** as the backend, keeping a client-side model of the domain separate from the UI, and pushing logic out of components. Treat what follows as **DDD-inspired organization**, not as a place to re-enforce business rules.

### The naive tracking store (before)

We shipped this. One store, 200 lines, every component reaching into it:

```typescript
// everything-tracking.store.ts — a flat dump of everything
@Injectable({ providedIn: 'root' })
export class EverythingTrackingStore {
  shipments = signal<Shipment[]>([]);     // Shipment = raw API response, used directly
  loading = signal(false);
  error = signal<string | null>(null);
  
  loadAll() {                             // loads every shipment for every context
    this.loading.set(true);
    fetch('/api/shipments')               // raw JSON, no typing
      .then(r => r.json())
      .then(data => this.shipments.set(data));  // DTOs leak into components
  }
}
// Components: <div>{{ shipment.customer_name }}</div> — snake_case everywhere
// Components: <div *ngIf="store.shipments().status === 'DELIVERED'"> — string comparisons
// Components call fetch() directly for "quick" updates — bypassing the store
```

This is the frontend anemic model: no domain types, logic in components, state scattered.

### What this cost us
- `shipment.customer_name` — typo in a template. Missed in dev, shipped to prod. 500 customer-support calls.
- A component called `fetch('/api/cancel')` directly, bypassing the store. Cancelled bookings reappeared after the store refresh.
- A "quick filter" component mutated `store.shipments()` directly. Three components broke because they held stale references.

### After: layered, domain-driven

- `domain/` → `Shipment` class with typed getters, `BookingNumber` VO, `ShipmentStatus` enum
- `application/` → `TrackingStore` — single source of truth, read-only signals
- `infrastructure/` → `TrackingApi` (facade), `ShipmentResource` (DTO), `ShipmentAssembler` (ACL)
- `presentation/` → views (smart) + components (dumb)

Now components read `shipment.bookingNumber` (typed), and cancellation goes through the store: `store.cancelBooking(bookingNumber)`.

## Folder structure: bounded contexts and four layers

Organize `src/app/` by **bounded context** (a feature area), and split each context into four layers — `domain`, `application`, `infrastructure`, and `presentation` (the frontend's name for the interfaces/inbound layer):

```
src/app/
├── tracking/                          // the Tracking bounded context
│   ├── domain/
│   │   └── model/
│   │       └── shipment.entity.ts     // class: private fields + getters
│   ├── application/
│   │   └── tracking.store.ts          // signal store (state + orchestration)
│   ├── infrastructure/
│   │   ├── tracking-api.ts            // context API facade — the store uses this
│   │   ├── shipments-api-endpoint.ts  // the repository (extends BaseApiEndpoint)
│   │   ├── shipments-response.ts      // ShipmentResource + ShipmentsResponse (DTOs)
│   │   └── shipment-assembler.ts      // resource <-> entity (ACL)
│   └── presentation/
│       ├── views/                     // routed "smart" components (inject the store)
│       ├── components/                // reusable "dumb" components (input()/output())
│       └── tracking.routes.ts         // context's own lazy-loaded routes
├── shared/                            // the shared kernel (see below)
└── app.routes.ts                      // root router composes the contexts
```

Dependencies point inward toward `domain`: `presentation` and `infrastructure` depend on `domain`; `domain` depends on nothing. Each context owning its own routes keeps the boundary visible at the routing level.

## The shared kernel

The `shared/` folder is the **shared kernel** — what genuinely belongs to every context. Unlike on the backend, it spans all four layers, including UI:

```
shared/
├── domain/model/
│   └── base-entity.ts                 // BaseEntity: the { id } every entity carries
├── infrastructure/
│   ├── base-response.ts               // BaseResource / BaseResponse (DTO markers)
│   ├── base-assembler.ts              // BaseAssembler<Entity, Resource, Response>
│   ├── base-api-endpoint.ts           // generic CRUD endpoint
│   └── base-api.ts                    // base for a context's API facade
└── presentation/
    ├── components/                    // Layout (app shell), footer, BaseForm
    └── views/                         // app-wide views: home, page-not-found
```

- **`domain/model`** — `BaseEntity` is an **interface** (`{ id: number }`); entities `implements BaseEntity`.
- **`infrastructure`** — the base classes the infra layer builds on: `BaseResource`/`BaseResponse` (interfaces), `BaseAssembler` (the mapping contract), `BaseApiEndpoint` (CRUD with error handling), and `BaseApi` (a marker the context APIs extend).
- **`presentation`** — this is real UI, and it legitimately belongs to the kernel — a **`Layout`** shell, app-wide **views**, reusable cross-cutting components, and **`BaseForm`**. It's UI rather than domain, but it's *shared* UI.

Keep the kernel small: base classes, the app shell, and a few app-wide views.

## The domain layer: entities (and commands)

Model the domain as **classes** named in the ubiquitous language. Each entity lives in a `*.entity.ts` file with **private fields**, **getters**, and a constructor taking a single options object. An entity `implements BaseEntity`.

### CargoRoute: Shipment entity

```typescript
// tracking/domain/model/shipment.entity.ts
import { BaseEntity } from '../../../shared/domain/model/base-entity';

export type ShipmentStatus = 'BOOKED' | 'CONFIRMED' | 'IN_TRANSIT' | 'DELIVERED' | 'CANCELED';

export class Shipment implements BaseEntity {
  private _id: number;
  private _bookingNumber: string;        // references Booking aggregate by id
  private _origin: string;
  private _destination: string;
  private _status: ShipmentStatus;
  private _currentPort: string | null;
  private _eta: Date | null;

  constructor(input: {
    id: number; bookingNumber: string; origin: string; destination: string;
    status: ShipmentStatus; currentPort?: string | null; eta?: Date | null;
  }) {
    this._id = input.id;
    this._bookingNumber = input.bookingNumber;
    this._origin = input.origin;
    this._destination = input.destination;
    this._status = input.status;
    this._currentPort = input.currentPort ?? null;
    this._eta = input.eta ?? null;
  }

  get id(): number { return this._id; }
  get bookingNumber(): string { return this._bookingNumber; }
  get origin(): string { return this._origin; }
  get destination(): string { return this._destination; }
  get status(): ShipmentStatus { return this._status; }
  get currentPort(): string | null { return this._currentPort; }
  get eta(): Date | null { return this._eta; }
}
```

On the frontend, entities stay thin — the backend owns the invariants. An entity *may* hold a **resolved related object** (e.g. a `Shipment` carrying its `Vessel` details) that the store stitches in after loading; the raw id stays the source of truth.

**Commands** are for non-CRUD intents — multi-step actions, anything where the input isn't just "save this entity." A command is a class too (`*.command.ts`):

```typescript
// tracking/domain/model/cancel-shipment.command.ts
export class CancelTrackingCommand {
  private _bookingNumber: string;
  private _reason: string;

  constructor(input: { bookingNumber: string; reason: string }) {
    this._bookingNumber = input.bookingNumber;
    this._reason = input.reason;
  }

  get bookingNumber(): string { return this._bookingNumber; }
  get reason(): string { return this._reason; }
}
```

## Infrastructure: DTOs, assemblers, endpoints, and the context API

The infrastructure layer rests on the shared base classes from the kernel: `BaseResource`/`BaseResponse` (DTO markers), `BaseAssembler`, the generic `BaseApiEndpoint` that implements CRUD, and `BaseApi` for a context's API facade.

### DTOs — never leave this layer

They live in `*-response.ts`: a `Resource` (one item, extends `BaseResource`) and a `Response` (the envelope, extends `BaseResponse`). They mirror the API's wire shape:

```typescript
// tracking/infrastructure/shipments-response.ts
import { BaseResource, BaseResponse } from '../../shared/infrastructure/base-response';

export interface ShipmentResource extends BaseResource {
  id: number;
  booking_number: string;           // snake_case — never reaches the domain
  origin_port: string;
  destination_port: string;
  status: ShipmentStatus;
  current_port: string | null;
  eta: string | null;              // ISO date string on the wire
}

export interface ShipmentsResponse extends BaseResponse {
  shipments: ShipmentResource[];
}
```

### The assembler — the anti-corruption layer

It maps both ways — building entities with `new`, and turning an entity back into a resource for writes:

```typescript
// tracking/infrastructure/shipment-assembler.ts
export class ShipmentAssembler implements BaseAssembler<Shipment, ShipmentResource, ShipmentsResponse> {
  toEntityFromResource(resource: ShipmentResource): Shipment {
    return new Shipment({
      id: resource.id,
      bookingNumber: resource.booking_number,
      origin: resource.origin_port,
      destination: resource.destination_port,
      status: resource.status,
      currentPort: resource.current_port,
      eta: resource.eta ? new Date(resource.eta) : null,
    });
  }

  toEntitiesFromResponse(response: ShipmentsResponse): Shipment[] {
    return response.shipments.map(r => this.toEntityFromResource(r));
  }

  toResourceFromEntity(entity: Shipment): ShipmentResource {
    return {
      id: entity.id,
      booking_number: entity.bookingNumber,
      origin_port: entity.origin,
      destination_port: entity.destination,
      status: entity.status,
      current_port: entity.currentPort,
      eta: entity.eta?.toISOString() ?? null,
    } as ShipmentResource;
  }
}
```

The assembler is the **anti-corruption layer**: the API's snake_case and quirks stop here. They never reach the domain or the views.

### The endpoint — the repository

It declares only its URL and assembler; CRUD comes from the base class:

```typescript
// tracking/infrastructure/shipments-api-endpoint.ts
export class ShipmentsApiEndpoint extends BaseApiEndpoint<
  Shipment, ShipmentResource, ShipmentsResponse, ShipmentAssembler
> {
  constructor(http: HttpClient) {
    super(http, `${environment.apiBaseUrl}/tracking/shipments`, new ShipmentAssembler());
  }
}
```

### The context API — the store's only gateway

```typescript
// tracking/infrastructure/tracking-api.ts
@Injectable({ providedIn: 'root' })
export class TrackingApi extends BaseApi {
  private readonly shipments: ShipmentsApiEndpoint;

  constructor(http: HttpClient) {
    super();
    this.shipments = new ShipmentsApiEndpoint(http);
  }

  getShipments(): Observable<Shipment[]> { return this.shipments.getAll(); }
  getShipment(id: number): Observable<Shipment> { return this.shipments.getById(id); }
  cancelShipment(bookingNumber: string, reason: string): Observable<Shipment> {
    return this.shipments.custom<CancelTrackingCommand, ShipmentResource>(
      `${bookingNumber}/cancel`, 'POST', new CancelShipmentRequest(bookingNumber, reason)
    ).pipe(map(r => new ShipmentAssembler().toEntityFromResource(r)));
  }
}
```

For CRUD there's no separate request DTO — the resource is the write payload. For command-style writes, add a dedicated `*.request.ts` and an assembler mapping command → request.

## The application layer: a signal store

The application layer holds **state** and orchestrates **use cases**. A signal-based store: talks to the context API, exposes state through read-only signals, keeps coordination out of views.

```typescript
// tracking/application/tracking.store.ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Injectable({ providedIn: 'root' })
export class TrackingStore {
  private api = inject(TrackingApi);

  private readonly shipmentsSignal = signal<Shipment[]>([]);
  private readonly loadingSignal = signal(false);
  private readonly errorSignal = signal<string | null>(null);

  readonly shipments = this.shipmentsSignal.asReadonly();
  readonly loading = this.loadingSignal.asReadonly();
  readonly error = this.errorSignal.asReadonly();
  readonly inTransit = computed(() =>
    this.shipmentsSignal().filter(s => s.status === 'IN_TRANSIT'));

  constructor() { this.loadShipments(); }

  shipmentByBookingNumber(bookingNumber: string): Signal<Shipment | undefined> {
    return computed(() =>
      this.shipmentsSignal().find(s => s.bookingNumber === bookingNumber));
  }

  loadShipments(): void {
    this.loadingSignal.set(true);
    this.errorSignal.set(null);
    this.api.getShipments().pipe(takeUntilDestroyed()).subscribe({
      next: shipments => {
        this.shipmentsSignal.set(shipments);
        this.loadingSignal.set(false);
      },
      error: () => {
        this.errorSignal.set('Could not load shipments');
        this.loadingSignal.set(false);
      },
    });
  }

  cancelShipment(bookingNumber: string, reason: string): void {
    this.loadingSignal.set(true);
    this.errorSignal.set(null);
    this.api.cancelShipment(bookingNumber, reason).pipe(takeUntilDestroyed()).subscribe({
      next: canceled => {
        this.shipmentsSignal.update(ss =>
          ss.map(s => s.bookingNumber === bookingNumber ? canceled : s));
        this.loadingSignal.set(false);
      },
      error: () => {
        this.errorSignal.set('Could not cancel shipment');
        this.loadingSignal.set(false);
      },
    });
  }
}
```

State is private; the outside reads it through `asReadonly()` signals and `computed()` derivations. The store is also where **cross-entity coordination** lives — keeping that out of views.

> **Angular 22+ alternative:** `httpResource()` can replace the manual `signal` + `subscribe` wiring for the query side. The pattern above works unchanged as an explicit fallback when you need the retry/error coordination shown here.

## The presentation layer: views and components

Two kinds of component. A **view** is a routed, "smart" component: injects the store, reads its signals, dispatches actions. A **component** is reusable, "dumb": takes data via `input()`, reports intent via `output()`, knows nothing about the store or the API.

```typescript
// tracking/presentation/views/shipment-list/shipment-list.ts
@Component({
  selector: 'app-shipment-list',
  standalone: true,
  imports: [ShipmentItem],
  templateUrl: './shipment-list.html',
})
export class ShipmentList {
  private store = inject(TrackingStore);

  protected readonly shipments = this.store.inTransit;     // a signal, read in the template
  protected readonly loading = this.store.loading;
  protected readonly error = this.store.error;
}
```

```typescript
// tracking/presentation/components/shipment-item/shipment-item.ts
@Component({
  selector: 'app-shipment-item',
  standalone: true,
  templateUrl: './shipment-item.html',
})
export class ShipmentItem {
  shipment = input.required<Shipment>();
  cancel = output<string>();              // emits the booking number to cancel
}
```

Business logic in a component is the frontend fat-controller smell — push it into the store. The view stays thin too: it wires the store to the components and handles navigation, nothing more.

## Routing

Each context owns a `*.routes.ts` **inside its `presentation/` folder**, exporting a `Routes` array that lazy-loads its views with `loadComponent`:

```typescript
// tracking/presentation/tracking.routes.ts
import { Routes } from '@angular/router';

const shipmentList = () => import('./views/shipment-list/shipment-list')
  .then(m => m.ShipmentList);
const shipmentDetail = () => import('./views/shipment-detail/shipment-detail')
  .then(m => m.ShipmentDetail);

export const trackingRoutes: Routes = [
  { path: 'shipments',       loadComponent: shipmentList },
  { path: 'shipments/:id',   loadComponent: shipmentDetail },
];
```

The root `app.routes.ts` composes the contexts with `loadChildren`:

```typescript
// app.routes.ts
const trackingRoutes = () => import('./tracking/presentation/tracking.routes')
  .then(m => m.trackingRoutes);

export const routes: Routes = [
  { path: 'home',      component: Home },
  { path: 'tracking',  loadChildren: trackingRoutes },
  { path: '',          redirectTo: '/home', pathMatch: 'full' },
];
```

Lazy-loading each context keeps the bounded-context boundary visible at the routing level, and the bundles split along it.

## Reactive forms and writes

A reactive form gathers and validates input as **UX** — fast feedback, while the server validates again. For CRUD, the form builds the **entity** and hands it to the store; a form view can `extend BaseForm`:

```typescript
// tracking/presentation/views/shipment-cancel/shipment-cancel.ts
export class ShipmentCancel extends BaseForm {
  private fb = inject(FormBuilder);
  private store = inject(TrackingStore);
  private router = inject(Router);

  protected form = this.fb.group({
    reason: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
  });

  submit(): void {
    if (this.form.invalid) return;
    const command = new CancelShipmentCommand({
      bookingNumber: this.route.snapshot.params['bookingNumber'],
      reason: this.form.value.reason!,
    });
    this.store.cancelShipment(command.bookingNumber, command.reason);
    this.router.navigate(['tracking/shipments']);
  }
}
```

## Strategic design on the frontend

- **Bounded contexts** become feature folders (or Nx libraries). A real app has several — `booking`, `routing`, `tracking`, `billing`. When one context needs another — say `tracking` needs the signed-in user from `identity` — it consumes that context's store, not its internals.
- **Ubiquitous language** runs through the names: `Shipment`, `TrackingStore`, `TrackingApi`, `CancelShipmentCommand`.
- **The shared kernel** holds what every context reuses — base classes, the app shell, app-wide views.
- **Assemblers** are the anti-corruption layer: they protect the domain model from the backend's wire format.

A real app also leans on plumbing that isn't domain modeling — a component library in the views, an i18n pipe, cross-cutting interceptors in infrastructure. Keep it where it belongs: out of the domain and the stores.

## What carries over, loosens, or doesn't apply

- **Carries over:** the ubiquitous language; bounded contexts; the four-layer split with an isolated domain; anti-corruption via assemblers; keeping logic out of the UI.
- **Loosens:** repositories are API endpoints rather than aggregate stores; aggregates and value objects are lighter (the UI rarely needs them); "domain events" are usually signal/observable updates.
- **Doesn't apply:** authoritative invariants and transactional consistency — those belong to the backend. Client-side checks are UX, and the server validates again.

## Testing each layer

- **`domain`** — plain Jest tests on entities: construct one, check getters. No `TestBed`.
- **`infrastructure`** — test assemblers as pure functions. Test API/endpoint classes with `HttpTestingController`, asserting the right URL and payload.
- **`application`** — test the store by providing a mock API (spy on `TrackingApi`), calling a method, and asserting on signals. Cover error paths too.
- **`presentation`** — "dumb" components mount directly and assert on `input()`/`output()`. "Smart" views are best tested by providing a fake store via DI.

## Common pitfalls

- **Business rules re-implemented client-side "just in case."** Keeps checks to UX level; let the backend be authoritative.
- **Components injecting the API service directly**, skipping the store. Route everything through the store.
- **One global store for the whole app** instead of one per bounded context. One per context.
- **DTOs (resources) used directly in templates** instead of being assembled into entities first. Leaks snake_case and backend quirks into the UI.
- **Forgetting `takeUntilDestroyed()`** on store subscriptions. State updates after teardown.
- **Dumping feature-specific code into `shared/`** because it's convenient. The kernel becomes a second uncontrolled context.

## Quick reference

| DDD concept | Angular idiom |
|---|---|
| Bounded Context | Feature folder, four internal layers |
| Entity | `*.entity.ts` class, private fields + getters, `implements BaseEntity` |
| Command (non-CRUD intent) | `*.command.ts` class, same encapsulation |
| Repository | `*-api-endpoint.ts` extending `BaseApiEndpoint` |
| Anti-corruption layer | `*-assembler.ts` implementing `BaseAssembler`, mapping resource ⇄ entity |
| Application service | Signal store (`*.store.ts`), `@Injectable({ providedIn: 'root' })` |
| Domain event | A signal update / observable emission from the store |
| Shared kernel | `shared/` folder spanning all four layers |
