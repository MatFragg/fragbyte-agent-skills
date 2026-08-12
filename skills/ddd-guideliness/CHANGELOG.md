# Changelog — DDD Guideliness

All notable changes to this skill are documented in this file.

Format is based on [Keep a Changelog](https://keepachangelog.com/), and this skill adheres to [Semantic Versioning](https://semver.org/).

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
- `angular.md`: Changed example domain from QuickBite to CargoRoute for consistency with the rest of the skill.
- Folder renamed from `ddd-guideliness-v1` to `ddd-guideliness`.

### Fixed
- `SKILL.md` "Map of this skill" table now includes a row for `design-patterns-arch-patterns.md`.
- `tactical-patterns.md` reference updated from `strategic-design.md` to `strategic-design.md` and `domain-modeling.md`.

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