---
name: ddd-guideliness
description: Apply Domain-Driven Design when modeling a business domain or structuring code around business logic, on the backend or the frontend. Use whenever the user is designing or refactoring a domain model, pulling business rules out of controllers or UI components, organizing a backend service or an Angular app by domain or bounded context, or mentions DDD, bounded contexts, ubiquitous language, aggregates, entities, value objects, domain events, repositories, domain or application services, anti-corruption layers, or CQRS — even if they never say "DDD". Covers strategic design and collaborative modeling, tactical patterns, and idiomatic implementation in Spring Boot/Java, NestJS/TypeScript, or Angular. Prefer it over ad-hoc modeling whenever non-trivial business rules or invariants are involved.
license: MIT
metadata:
  author: Ethan Matias Aliaga Aguirre
  version: "1.0.12"
---

# Domain-Driven Design (DDD)

Domain-Driven Design exists because someone once put a ship on the wrong vessel.

That's not a hypothetical — it's the story of CargoRoute's fifth major production incident. The system let a customer *book* cargo on a vessel, but never checked whether that vessel's *route* actually went to the destination. The cargo arrived somewhere the customer wasn't. A $4M recovery operation later, we had DDD.

DDD tackles complex software by putting the **business domain** — its language, rules, and invariants — at the center of the design, instead of letting the database schema, the framework, or the UI drive the model.

Apply this whenever you model a domain or write code around business logic. Follow the decision rules in this file; open a reference file when you need depth on a specific area. The rules matter more than any single example.

## Map of this skill

| Start here when... | Read |
|---|---|
| You're designing from scratch and need to pick a starting point | This file |
| You need to discover the domain with business experts | `strategic-design.md` and `domain-modeling.md` |
| You're modeling entities, aggregates, or invariants inside a single context | `tactical-patterns.md` |
| You're writing Java/Spring Boot code | `spring-boot.md` |
| You're writing NestJS/TypeScript code | `nestjs.md` |
| You're writing Angular/TypeScript code | `angular.md` |
| You're looking up design patterns and their DDD equivalents | `design-patterns-arch-patterns.md` |

Typical reading order for a new task: this file → `strategic-design.md` → `domain-modeling.md` → `tactical-patterns.md` → the stack-specific reference.

## Prime directives

These four rules are the ones every other piece of guidance in this skill exists to serve. When in doubt about a design decision, come back to these.

1. **Put business rules inside the domain model.** Rules and invariants live in entities, value objects, and aggregates — not scattered across controllers, application services, or SQL. A model that only holds data while the logic lives elsewhere is an **anemic domain model**, the single most common DDD failure.
2. **Speak the ubiquitous language.** Use the exact terms domain experts use, in class/method/variable names and in conversation with the user. If the business says "booking confirmed", the code says `BookingConfirmed`, not `OrderRecordActivated`.
3. **Keep the domain pure.** The domain layer expresses business concepts only, and must not depend on frameworks, persistence, web, or messaging concerns.
4. **Do strategic design before tactical design.** Decide the boundaries (bounded contexts) and the language before picking entities and aggregates inside them.

### Recognizing an anemic domain model

Because rule 1 is the failure mode you'll hit most often, watch for these tells and treat any of them as a prompt to move logic inward:

- Entities/classes with only getters and setters, no behavior.
- "Service" or "Manager" classes that contain `if` chains implementing business rules over data pulled from otherwise-passive objects.
- The same invariant re-checked in multiple call sites instead of being unbreakable by construction.

## Layered architecture (4 layers)

Organize code into four layers. Dependencies point **inward**, toward the domain; the domain depends on nothing outside itself.

1. **Interfaces** — the **inbound adaptors**: entry points where the outside world drives the context (REST controllers, CLI, message listeners, schedulers).
2. **Application** — orchestrates use cases: loads aggregates, invokes domain behavior, manages transactions and security. Holds **no business rules** itself.
3. **Domain** — the heart: entities, value objects, aggregates, domain events, domain services, and the repository and service *interfaces* (**ports**).
4. **Infrastructure** — the **outbound adaptors**: the technical implementations the context uses to reach external systems (persistence/ORM and repository implementations, messaging, external API clients).

The inner layers declare **ports** (interfaces) and the adaptors implement them.

### A quick self-check per layer

When placing new code, ask which of these it is doing — the answer tells you the layer:

| If the code... | It belongs in... |
|---|---|
| Decides whether an operation is allowed, or what a valid state looks like | Domain |
| Loads an aggregate, calls its method, saves it, manages the transaction | Application |
| Parses an HTTP request, maps a DTO, returns a status code | Interfaces |
| Talks to a database, queue, or external API | Infrastructure |

## Tactical building blocks — how to decide

When modeling, choose the right block deliberately:

- **Value Object** — no identity; defined entirely by its attributes; immutable. Use liberally: cargo weight, route distance, booking capacity.
- **Entity** — has a distinct identity that persists through changes. A `Booking` stays the same booking even if the vessel is changed.
- **Aggregate** — a cluster of entities and value objects treated as one consistency unit, accessed only through its root.
- **Domain Event** — a statement that something meaningful happened (e.g., `BookingConfirmed`).
- **Domain Service** — stateless domain logic that doesn't belong to a single entity (e.g., checking route feasibility across bookings).
- **Repository** — collection-like access to aggregates by their root.
- **Factory** — encapsulates complex creation of an aggregate or value object.

→ For the full catalog with detailed rules, aggregate-boundary trade-offs, and worked examples, read `references/tactical-patterns.md`.

## Strategic design — essentials

Before tactical modeling, get the big picture right:

- **Bounded Context** — an explicit boundary within which a model and its ubiquitous language stay consistent. "Booking" in the Customer portal ≠ "Booking" in the Port Operations system.
- **Subdomains** — distinguish the **core** (your competitive advantage), **supporting**, and **generic** (buy/reuse) subdomains.
- **Context Mapping** — define the relationships between bounded contexts (e.g., an **Anti-Corruption Layer** to protect your model from an external one).

→ For the strategic concepts in depth, read `references/strategic-design.md`. For the collaborative modeling process and tools, read `references/domain-modeling.md`.

## CQRS

Command Query Responsibility Segregation separates the model that **changes** state from the model that **reads** it. Reach for it when read and write needs genuinely diverge — complex queries, very different read/write load, or read models that span aggregates.

→ For depth, read `references/tactical-patterns.md`.

## How to approach a DDD task

Use this checklist for any non-trivial modeling or refactoring request:

1. **Establish the language.** Clarify the domain terms with the user; use them verbatim.
2. **Locate the bounded context.** Which context are we in, and what is its model?
3. **Find the aggregates and their invariants.** What must always be true?
4. **Model tactically.** Choose value objects, entities, and aggregate roots; push rules into them.
5. **Place each piece in the right layer.**
6. **Implement for the stack.** Read the matching implementation reference.

## Implementation references (by stack)

The principles above are stack-agnostic, but the idioms differ. When writing code, read the file for the project's stack:

- **Spring Boot / Java** → `references/spring-boot.md`
- **NestJS / TypeScript** → `references/nestjs.md`
- **Angular (frontend, DDD-adapted)** → `references/angular.md`
- **Design patterns & architectural patterns** → `references/design-patterns-arch-patterns.md`
