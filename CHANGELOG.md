# Changelog — Fragbyte Agent Skills

All notable changes to the repository will be documented in this file.

Format is based on [Keep a Changelog](https://keepachangelog.com/), and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Changed

- **ddd-guideliness** skill (v1.0.4): `spring-boot.md` now properly separates aggregates, entities, and value objects; adds `AuditableModel` for internal entities; new internal entity examples (Cargo, RouteStop) in CargoRoute domain
- **ddd-guideliness** skill (v1.0.3): added State, Template Method, and Composite patterns plus aggregate-root create-methods to `design-patterns-arch-patterns.md`, all rewritten as language-agnostic pseudo-code in the CargoRoute domain.

## [1.0.0] — 2026-08-12

### Added

- Repository scaffolding: `README.md`, `AGENTS.md`, `LICENSE`, `CHANGELOG.md`.
- **ddd-guideliness** skill (moved from `ddd-guideliness-v1`): a comprehensive Domain-Driven Design guide covering strategic design, tactical patterns, collaborative modeling, and stack-specific implementation for Spring Boot and Angular. Example domain: CargoRoute.
