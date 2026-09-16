# Changelog — DDD Guideliness

All notable changes to this skill are documented in this file.

Format is based on [Keep a Changelog](https://keepachangelog.com/), and this skill adheres to [Semantic Versioning](https://semver.org/).

## [1.0.12] — 2026-09-16

### Changed

- `nestjs.md` / `spring-boot.md`: Interface assemblers now use directional names (`toCommandFromResource` / `toResourceFromEntity`) instead of generic `toCommand` / `toResource`, matching the class names and the existing `angular.md` convention (`toEntityFromResource` / `toResourceFromEntity`). Definitions and controller call sites updated.
- `SKILL.md`: Version bumped to `1.0.12`.

## [1.0.11] — 2026-09-16

### Changed

- `nestjs.md`: Service method names now derive from the command/query class (`lowerCamel` without the `Command`/`Query` suffix): `getByBookingNumber` → `getBookingById`, `findForVoyage` → `findBookingsForVoyage` (`placeBooking` / `cancelBooking` already complied). Interface, impl, controller, `@nestjs/cqrs` mapping table, and quick reference updated.
- `SKILL.md`: Version bumped to `1.0.11`.

## [1.0.10] — 2026-09-15

### Changed

- `nestjs.md`: Command and query services now use one named method per case (`placeBooking` / `cancelBooking`, `getByBookingNumber` / `findForVoyage`, each taking its command/query object) instead of overloaded `handle`. TypeScript has no runtime overloading, so a single `handle` forced a `CommandA | CommandB` union plus an `instanceof` chain — ambiguous for codegen and inconsistent at scale. `spring-boot.md` keeps native overloaded `handle`; controller, trade-off note, `@nestjs/cqrs` mapping table, and quick reference updated accordingly.
- `SKILL.md`: Version bumped to `1.0.10`.

## [1.0.9] — 2026-09-13

### Fixed

- `nestjs.md`: `BookingRepositoryImpl` getter now translates the ambient `EntityManager` via `manager.getRepository(BookingOrmEntity)` instead of returning `TransactionContext.current ?? this.orm` directly — the old form is `EntityManager | Repository | null` vs `Repository` and fails under `strict: true` (TS2322).

## [1.0.8] — 2026-09-11

### Changed

- `nestjs.md`: Command and query services now expose only overloaded `handle` (Spring-style) — `BookingCommandService` handles `PlaceBookingCommand` / `CancelBookingCommand`, `BookingQueryService` handles `GetBookingByIdQuery` / `FindBookingsForVoyageQuery` with narrowing implementations. Removed `handleCancel` primitives and `getByBookingNumber` / `findForVoyage` names; controller, `@nestjs/cqrs` mapping table, and quick reference updated accordingly.
- `SKILL.md`: Version bumped to `1.0.8`.

## [1.0.7] — 2026-09-10

### Added

- `tactical-patterns.md`: New "Naming commands, queries, events (and handlers)" rule under CQRS — `Action + Target + Command`, `Action + Target + Criteria + Query`, `Target + PastAction` (`Event` suffix only on collision), handlers as `<EventName>EventHandler`.
- `nestjs.md`: New `CancelBookingCommand` example; `Commands and queries` and `Domain events` sections now state the naming rule with file-name mirrors.

### Changed

- `nestjs.md`: Closed sets (`WeightUnit`, `BookingStatus`, `CargoStatus`, `ShipmentStatus`) now use `as const` object + type union instead of `export enum` — the TS equivalent of Spring's `enum`, safe under `isolatedModules` / `erasableSyntaxOnly`. Parsers use `Object.values()` guards; `@IsEnum` replaced with `@IsIn(Object.values(...))`; added `parseShipmentStatus()` / `parseCargoStatus()`.
- `nestjs.md`: `GetBookingQuery` renamed to `GetBookingByIdQuery` (`get-booking-by-id.query.ts`); `BookingConfirmedHandler` renamed to `BookingConfirmedEventHandler` (`booking-confirmed.event-handler.ts`).
- `SKILL.md`: Version bumped to `1.0.7`.

## [1.0.6] — 2026-09-02

### Added

- `nestjs.md`: New "Cross-context reference data" subsection under the anti-corruption layer — provider VO, consumer minimal VO, and Shared-Kernel VO (3+ contexts) variants, matching `spring-boot.md`.
- `nestjs.md`: Shared kernel gains an `AggregateRoot` domain base (hosts the domain-event buffer and `pullDomainEvents()`); `Booking` and `Shipment` extend it. The unused `BaseEntity` interface was removed.
- `nestjs.md`: Added `ShipmentId` value object; new "VO construction conventions" rule — quantity VOs use a public constructor, identifier VOs a `private` constructor with `of()`/`generate()` (converted `CustomerId`/`PortCode` accordingly).

### Changed

- `nestjs.md`: Repository timing and default now explain *why* it differs from Spring — JPA-on-entity (metadata-only) makes direct injection the Spring baseline, while TypeORM's runtime decorators force the plain-domain + separate-ORM split, so the port + assembler is the Nest default. `When to skip the port` reframed to Spring-style criteria; "TypeORM in the domain" softened accordingly.
- `nestjs.md`: Hardened the Composite `Shipment` — child transitions go through the root (`confirmCargo()`), the root carries a real `ShipmentStatus` set by `confirm()` (raising `ShipmentConfirmed`), and `ShipmentOrmEntity` persists the status.
- `nestjs.md`: Compressed rationale asides (why every port needs a token, query handler naming, why `run()` not `start()`/`complete()`, why not decorate the domain class, `@nestjs/cqrs` note) to one line + a pointer to the parent reference, bringing the tone in line with `spring-boot.md`.
- `SKILL.md`: Version bumped to `1.0.6`.

## [1.0.5] — 2026-09-01

### Added

- `nestjs.md`: New NestJS framework reference covering DDD tactical patterns, package structure, aggregates, entities, value objects, domain services, repositories, application services, commands/queries, domain events, outbound services, ACLs, REST interfaces, error handling, dependency injection, and testing.
- `nestjs.md`: New "Unit of work" guidance — `IUnitOfWork` port + `UNIT_OF_WORK` token in the shared kernel, `TypeOrmUnitOfWork` adapter + AsyncLocalStorage `TransactionContext` in infrastructure, writes run inside `unitOfWork.run()`, domain events published after the work commits.
- `nestjs.md`: New Composite aggregate example — `Shipment` owning a collection of `CargoItem`s, built through root create-methods (dedup + validity) with derived whole-state, persisted as a parent + child table (`@OneToMany`/`@ManyToOne`); JSONB framed as a value-object-shaped exception, not the default.
- `SKILL.md`: Registered `nestjs.md` in the "Map of this skill" table and "Implementation references (by stack)"; description now mentions NestJS/TypeScript.

### Changed

- `nestjs.md`: Read path (`FindBookingsForVoyageQuery`, repository port, adapter, tests) now types `voyageNumber` as `VoyageNumber` end-to-end — unwrapped to `.value` only at the ORM boundary.
- `spring-boot.md` + `nestjs.md`: ACL facade signature reconciled to one rule — primitives by default (the provider's own language); a VO/DTO payload only for large ~6+-field or compound payloads; a shared VO only via the Shared Kernel threshold (3+ contexts). Spring Boot's `VesselSchedulingFacade` example moved to primitives.

## [1.0.4] — 2026-08-26

### Changed

- `spring-boot.md`: Restructured to explicitly separate **aggregates**, **entities**, and **value objects** per DDD tactical patterns
- `spring-boot.md`: Package structure now includes `domain/model/entities/` (flat) alongside `aggregates/` and `valueobjects/`
- `spring-boot.md`: Added `AuditableModel` in shared kernel for internal entities needing auto-generated surrogate ID + audit timestamps
- `spring-boot.md`: New "Internal entities within an aggregate" subsection — two patterns: (1) explicit ID assigned by aggregate root, (2) auto-generated ID via `AuditableModel`; CargoRoute examples `Cargo` and `RouteStop`
- `spring-boot.md`: Section renamed from "Aggregate root and entities" to "Entities and aggregate roots"
- `spring-boot.md`: Quick reference table splits internal entities (two rows) from aggregate roots

### Fixed

- `spring-boot.md`: Internal entities were previously conflated with aggregate roots; now correctly modeled as `@Embeddable` classes with identity (explicit or generated) within the aggregate boundary

## [1.0.3] — 2026-08-15

### Added

- `design-patterns-arch-patterns.md`: New "State" behavioral pattern — simplified enum + guarded-transition flavor, CargoRoute `Booking` lifecycle example.
- `design-patterns-arch-patterns.md`: New "Template Method" behavioral pattern — `Cargo.renderManifest()` skeleton with `ContainerCargo`, `BulkCargo`, `RefrigeratedCargo` hooks.
- `design-patterns-arch-patterns.md`: New "Composite" structural pattern — `Shipment` deriving whole-state from its cargo children.
- `design-patterns-arch-patterns.md`: "Factory Method" extended with a "Create methods on the aggregate root" subsection — `Shipment.addContainer()`, `addBulkCargo()`, `addReeferCargo()`.
- `design-patterns-arch-patterns.md`: Added language-conventions disclaimer (pseudo-code intent vs. per-stack idioms) and updated Contents table, pattern decision tree, and pattern-to-concept cross-reference with the four new patterns.

## [1.0.2] — 2026-08-14

### Changed

- `angular.md`: Added guidance for keeping API base URLs in environment configuration and moving repeated literals into exported constants or enums.
- `angular.md`: Updated the endpoint/context API examples to use central constants for endpoint fragments instead of inline magic strings.

## [1.0.1] — 2026-08-12

### Added

- `spring-boot.md`: Expanded package structure diagram with `outboundservices/` (concept subpackages + `acl/`), `infrastructure/{technology}/{implementation}/` pattern, and `interfaces/acl/` for published facades.
- `spring-boot.md`: New "Outbound services" section — technology ports in `outboundservices/{concept}/`, adapters in `infrastructure/{technology}/{implementation}/`, when to add vs skip.
- `spring-boot.md`: New "Marker interfaces for Spring DI" section — problem/solution/why alternatives fail, CargoRoute `BCryptHashingService` example.
- `spring-boot.md`: "Repositories" section extended with "When to inject the repository directly" — YAGNI, no business rules in repo, `JpaRepository` IS the abstraction.
- `spring-boot.md`: Enhanced "Anti-corruption layer" — outbound vs inbound ACL, bidirectional ACL in monoliths, cross-context reference data pattern (Provider VO, consumer mapping, shared VOs).
- `spring-boot.md`: New "BC-scoped exception handling" section — global vs BC-specific `@RestControllerAdvice`, rules.
- `spring-boot.md`: Quick Reference table updated with outbound service, marker interface, and BC-scoped handler rows.
- `design-patterns-arch-patterns.md`: New "Ports & Adapters" architectural pattern — when/benefit/failure, tactical naming convention, when to skip.
- `design-patterns-arch-patterns.md`: New "Marker Interface" structural pattern — when/benefit/failure, CargoRoute example, decision rule.
- `design-patterns-arch-patterns.md`: Enhanced "Facade" pattern with bidirectional facades in monoliths and microservice migration note.
- `design-patterns-arch-patterns.md`: Pattern decision tree and cross-reference table updated with Ports & Adapters and Marker Interface.

## [1.0.0] — 2026-08-12

### Added

- Initial release of the DDD guideliness skill.
- Strategic design reference: bounded contexts, subdomains, context mapping, CargoRoute worked example.
- Tactical patterns reference: entity, value object, aggregate, domain event, domain service, repository, factory, CQRS — with invariant checklists, decision trees, and CargoRoute lessons.
- Domain modeling reference: EventStorming, domain message flow, bounded context canvas, context mapping patterns, modeling process.
- Design patterns & architectural patterns reference: Factory Method, Command, Strategy, Observer, Facade, Service Layer, Repository, Resource/DTO, Mapper/Assembler, Unit of Work, CQRS, Layered Architecture — with a pattern-to-code cross-reference.
- Spring Boot implementation reference: package structure, shared kernel, value objects, aggregates, commands/queries, services, repositories, domain events, ACL, REST interfaces, error handling, identity choices, testing, common pitfalls.
- Angular implementation reference: four-layer structure, shared kernel, entities/commands, DTOs/assemblers/endpoints, signal store, views/components, routing, reactive forms, strategic design on the frontend.

### Changed

- `tactical-patterns.md`: Removed all embedded code examples; replaced with conceptual summaries. Testing guidance kept inline. Updated quick-reference table from stack-specific idioms to design rules.
- Folder renamed from `ddd-guideliness-v1` to `ddd-guideliness`.

### Fixed

- `SKILL.md` "Map of this skill" table now includes a row for `design-patterns-arch-patterns.md`.
- `tactical-patterns.md` reference updated from `strategic-design.md` to `strategic-design.md` and `domain-modeling.md`.
