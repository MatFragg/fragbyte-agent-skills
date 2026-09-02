# Strategic Design

Read this when doing the **strategic** part of DDD: shaping the ubiquitous language, discovering subdomains, and drawing and mapping bounded contexts. This is where the architecture of a domain gets decided — do it before tactical modeling.

For the collaborative process and tools (EventStorming, Bounded Context Canvas), see `references/domain-modeling.md`. For the tactical building blocks (entities, value objects, aggregates), see `references/tactical-patterns.md`. For stack-specific implementation, see `references/spring-boot.md`.

## Why strategic design comes first

DDD has two halves: **strategic** (the big picture — language, subdomains, boundaries) and **tactical** (the building blocks inside a model). A common mistake is to jump straight into tactical work — creating entities, repositories, and value objects — before the boundaries are clear.

Strategic design is where you decide *how the system is divided and how the parts relate*. Getting it wrong is far more expensive to fix later than any single class: a misplaced aggregate boundary is a refactor, a misplaced *context* boundary is usually a multi-team, multi-service migration.

## Domain and domain model

- The **domain** is the subject area the software is about — CargoRoute's business of moving cargo across oceans.
- The **domain model** is the code that represents that business: concepts like Booking, Route, Vessel, Cargo, and the rules governing how they behave.
- What many developers call **business logic** is exactly this: the higher-level rules for how domain concepts behave and interact. In DDD it belongs *in the domain model*, not scattered across controllers or queries.

Keep the mental equation: **domain = your business; domain model = the code that represents it.**

## Ubiquitous language

The single most important strategic practice. The **ubiquitous language** is one shared language, built around the domain model, that the *whole* team uses everywhere — in conversation with domain experts, in documentation, and in the code itself.

Why it matters: most defects come from misunderstanding, not from typos. When a port supervisor says "the cargo is slotted" and the code says `BookingStatus.RESERVED`, every translation is a chance to get the rule wrong.

### The language is scoped to a bounded context

The ubiquitous language is not one global glossary. It is **local to each bounded context**. "Booking" means something concrete in CargoRoute's customer-facing system and something different in PortOps:

- In **Customer Booking**: a *Booking* is a customer's request with preferred routes, cargo details, and a status tracked through confirmation.
- In **Port Operations**: the same real-world thing is a *Load Job* — a container, a vessel, and a time window. It doesn't care about preferred routes or customer status.

Because the language differs, each context keeps its **own** model. They do not share one `Booking` class.

### Keep technical detail out of the language

The language should describe *what happens in the business*, not how the software does it. Technical words — flags, tables, queues, gateways — hide the domain.

**Before** (leaks implementation):
> "When the booking form submits, set `status = 1` in the bookings table and push a row onto the queue for the routing service."

**After** (reveals intention):
> "When a customer places a booking and a viable route is found, the booking is confirmed. A confirmed booking becomes available for port scheduling."

## Domain storytelling

Domain storytelling is a lightweight, **pictographic** technique for learning a domain *with* the domain experts. You draw a short story as actors, the work objects they act on, and the activities between them, in sequence.

Use it to:
- Build and validate the ubiquitous language before writing code.
- Surface the real steps, actors, and edge cases of a process.
- Derive user stories directly.

It is a conversation tool, not a deliverable: the value is the shared understanding it produces.

## Subdomains

CargoRoute's domain divides into coherent areas:

- **Booking** and **Routing** are the **core** — placing bookings and finding optimal routes are what the business lives or dies on.
- **Handling** and **Tracking** are **supporting** — necessary, competently built, but not a competitive lever.
- **Billing**, **Notifications**, and **Customer Management** are **generic** — buy or reuse rather than build.

### A quick classification test

When unsure how to classify a piece of the domain, ask two questions:

1. **Would customers notice or care if this were done differently?** If yes and it's a source of advantage → core. If yes but it's table stakes → supporting. If no → generic.
2. **Is there already a mature product/library/vendor that solves this well?** If yes and using it costs nothing strategic → generic, regardless of how central it feels.

## Bounded contexts

A **bounded context** is an explicit boundary within which one model and its ubiquitous language are consistent and valid. Inside the boundary, every term means exactly one thing.

Each bounded context has:
- its **own ubiquitous language**,
- its **own model**, and
- its **own, independent implementation**.

A "Cargo" in the Booking context (a customer's shipment request with preferred routing) is not the same as a "Cargo" in Port Operations (a container on the ground with a crane assignment and a deadline). Forcing one shared model across the whole system produces a tangle.

### Benefits
- The team and the business speak one language with less risk of misunderstanding.
- Smaller models are easier to maintain and test.
- Contexts can evolve independently.
- Side effects stop being surprises.

### Considerations
- More architectural complexity.
- More up-front effort mapping the domain.
- A mindset shift: everyone must agree on vocabulary and ownership.

### How to find the boundaries

In order of reliability:

1. **Follow the language.** Where the same word starts needing qualifiers ("a Booking, but I mean the *port ops* one"), that's a seam.
2. **Follow the pivotal points in an event timeline.** A moment where the process shifts phase (Booking Confirmed → Port Scheduling begins) is a strong candidate boundary.
3. **Follow team/ownership lines**, but treat them as a hypothesis to check against the language and event flow.

## Context mapping

Real systems have several bounded contexts, and they must cooperate. A **context map** is the picture of how the contexts relate: which ones depend on which, and how they communicate across their boundaries.

A context never reaches into another's model directly. Each context exposes an **interface** to the outside, and when two contexts interact you **translate** between their languages at the boundary.

The CargoRoute map, simplified:

```
[ Customer Booking ] --books--> [ Routing ] --confirms--> [ Port Operations ]
         |                           |                         |
         |                          v                        v
         |                   [ Vessel Scheduling ]      [ Tracking ]
         |                           |
         v                           v
    [ Notifications ]          [ Billing ]
```

- Customer Booking publishes `BookingConfirmed` → Routing consumes it via an anti-corruption layer (their model speaks `LoadJob`, not `Booking`).
- Port Operations publishes `CargoLoadedOnVessel` → Tracking updates the customer-facing view.
- Billing listens to both `BookingConfirmed` and `CargoLoadedOnVessel` to generate charges, through its own ACL.

The full catalog of context-mapping patterns (anti-corruption layer, open host service, conformist, shared kernel, customer/supplier, partnership, published language, separate ways) is in `references/domain-modeling.md`. The essential rule: **map the relationships explicitly, and translate at the boundary.**

## Characteristics of a strong domain model

Aim for a model that is:

- **Aligned** with the business's real model, strategy, and processes.
- **Isolated** from other domains and from the technical layers around it.
- **Loosely coupled** — it does not depend on the layers on either side.
- **Reusable**, so concepts are not duplicated across the system.
- **Cleanly separated** as its own layer.
- **Minimal in its dependencies** on frameworks.

This is the strategic justification for the **domain purity** rule in the main `SKILL.md`.

## Worked example: CargoRoute

CargoRoute manages cargo bookings across ocean freight. Three bounded contexts, each with its own model:

**Subdomains:**
- **Booking** and **Routing** are **core** — customer experience and route optimization are the business.
- **Port Operations** is **supporting** — necessary infrastructure, not differentiating.
- **Notifications** is **generic** — off-the-shelf messaging.

**Language boundary — "Booking" means different things in different contexts:**
- In **Customer Booking**, a *Booking* is the customer's request with preferred routes and vessel.
- In **Port Operations**, the same real-world cargo movement is a *Load Job*: a container, a vessel slot, and a time window.

Because the language differs, each context keeps its **own** model.

**A small context map:**

```
[ Booking Context ] --BookingConfirmed--> [ Routing Context ]
                                               |
                                          route is proposed
                                               v
                                       [ Port Ops Context ]
```

- Booking publishes `BookingConfirmed`; Routing translates it into its own `Cargo` model through an ACL.
- Port Ops consumes the finalized route; Booking does not reach into Port Ops' model.

This separation is what lets the Port Ops team change crane scheduling without touching Booking.

## A practical workflow

1. **Talk to the domain experts.** The language comes from here.
2. **Build the ubiquitous language** as you go. Use domain storytelling and BDD scenarios.
3. **Identify the subdomains** and label each core, supporting, or generic.
4. **Draw the bounded contexts** — one model and language per context.
5. **Map the contexts** — make relationships explicit, translate at boundaries.
6. **Only then go tactical** — model entities, value objects, and aggregates inside each context.