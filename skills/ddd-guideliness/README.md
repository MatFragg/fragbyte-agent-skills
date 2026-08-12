# DDD Guideliness

Apply Domain-Driven Design (DDD) when modeling a business domain or structuring code around business logic — on the backend (Spring Boot/Java) or the frontend (Angular).

## When to use this skill

- Designing or refactoring a domain model
- Pulling business rules out of controllers or UI components
- Organizing a backend service or Angular app by bounded context
- Mentions of DDD, bounded contexts, ubiquitous language, aggregates, entities, value objects, domain events, repositories, domain/application services, anti-corlation layers, or CQRS

## Example domain

All code examples and worked cases in this skill use **CargoRoute** — an ocean-freight booking system. The same patterns apply to any domain; CargoRoute provides a single consistent vocabulary and context throughout.

## Folder structure

```
skills/ddd-guideliness/
├── README.md                     # This file
├── CHANGELOG.md                  # Skill version history
├── SKILL.md                      # Main skill file
├── references/
│   ├── tactical-patterns.md      # Building blocks (entities, aggregates, VOs, etc.)
│   ├── strategic-design.md       # Bounded contexts, subdomains, context mapping
│   ├── domain-modeling.md        # EventStorming, canvas, modeling process
│   ├── design-patterns-arch-patterns.md  # Pattern mapping (Factory, Observer, etc.)
│   ├── spring-boot.md            # Java/Spring Boot implementation idioms
│   └── angular.md                # Angular/TypeScript implementation idioms
└── AGENTS.md                     # Skill-specific guidelines (optional)
```

## Reading order

1. **SKILL.md** — start here for the overview and decision rules
2. **strategic-design.md** — get boundaries and language right first
3. **domain-modeling.md** — collaborative modeling techniques
4. **tactical-patterns.md** — building blocks inside a context
5. **spring-boot.md** or **angular.md** — implementation idioms
6. **design-patterns-arch-patterns.md** — pattern cross-references

## License

MIT — see [../../LICENSE](LICENSE).