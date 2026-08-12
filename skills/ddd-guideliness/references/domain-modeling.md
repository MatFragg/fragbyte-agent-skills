# Collaborative Domain Modeling Process

Read this when you need to **run or structure the modeling work** of DDD: discovering the domain with experts, finding bounded contexts, designing each context, and mapping the relationships between them.

Where `references/strategic-design.md` defines the strategic *concepts*, this file describes the *process and tools* used to produce them. These are collaborative, workshop-style techniques. As an agent you will rarely run a live workshop, but use these to **structure your own analysis**, to ask the right questions, and to produce the same artifacts when helping design a system.

## Start by understanding the business

Modeling is a *team* activity, not a solo technical exercise. The biggest gains come from understanding how the system people work in and how they interact — far more than from any individual's technical skill. So the process starts with deeply understanding the business *with* its experts, and only then turns to software.

Treat the techniques below as ways to make that shared understanding explicit and checkable.

## EventStorming

EventStorming is the entry point: a bottom-up way to build a shared picture of a business process before any software design begins. You build a picture of the domain's events on a timeline, working left to right, asking what triggers each event and who cares about it.

### Who's in the room (or who you should be asking about)

Keep it cross-functional: engineers, domain experts, product, QA, UX, support/ops. Diversity of perspective is the entire point. Keep the group under ~10 people.

### The color-coded elements (sticky-note types)

Even in text, keep these categories visually distinct so the reader can scan for a type at a glance:

| Element | Color (convention) | Grammar | CargoRoute example |
|---|---|---|---|
| Domain Event | Orange | past tense | `Booking Confirmed` |
| Command | Blue | imperative | `Confirm Booking` |
| Actor | Small yellow | noun (role) | `Customer` |
| Aggregate | Yellow (large) | noun | `Booking` |
| Policy | Lilac/purple | "Whenever X, then Y" | `Whenever Booking Confirmed, then Propose Route` |
| Read Model | Green | noun (a view) | `Available Routes Screen` |
| External System | Pink | noun | `Vessel Scheduling System` |
| Pain point | Red note | free text | `Manual cargo weight verification, takes 45 min` |
| Question / knowledge gap | Red note with `?` | free text | `Who overrides vessel capacity in emergencies?` |

### The 10-step process

Run these roughly in order, but treat it as iterative — later steps routinely surface gaps that send you back to step 1 or 2.

1. **Unstructured exploration.** Brainstorm every domain event you or the experts can think of, in **past tense**: `Booking Placed`, `Booking Confirmed`, `Cargo Loaded On Vessel`. Don't filter or sequence yet — the goal is volume and honesty.
2. **Timelines.** Arrange events left to right in the order they occur. Model the **happy path** first as a clean spine, then branch off alternative paths (e.g., `Booking Canceled`) and exceptional paths (e.g., `Cargo Discharged to Wrong Vessel`).
3. **Pain points.** Walk the completed timeline looking for trouble: bottlenecks, manual steps, missing documentation, or missing domain knowledge. A step nobody can explain is itself a critical finding — flag it.
4. **Pivotal points.** Find events that mark a phase change — `Booking Confirmed` → port scheduling begins; `Cargo Loaded` → transit begins. Mark each with a vertical divider. These are your first signal of where **bounded context boundaries** might fall.
5. **Commands.** For each event, ask "what triggered this?" and add the command: `Place Booking` → `Booking Placed`. Note who issues the command.
6. **Policies.** Look for commands that have no specific human actor — these are **automation policies**: "whenever Booking Confirmed, automatically issue Propose Route."
7. **Read models.** For each command issued by a human, ask what information they looked at — a screen, report, or notification.
8. **External systems.** Add anything outside the domain that issues commands (e.g., a `Vessel Scheduling System` webhook) or gets notified of events.
9. **Aggregates.** Group related commands/events into aggregates: an aggregate is the thing that **receives a command and produces the resulting event(s)**. Name each as a noun (`Booking`, `Cargo`, `Route`).
10. **Bounded contexts.** Cluster aggregates that are closely related — by shared functionality or by being tightly coupled through policies found in step 6. Cross-check against pivotal points from step 4.

### Producing the artifact as text

When reconstructing an EventStorming session, a good default output shape is a table per phase (happy path, then each alternative/exceptional path), with columns for `Trigger (Actor/Policy/External System)`, `Command`, `Event`, `Aggregate`, and notes for pain points or open questions. Call out pivotal points and candidate bounded contexts explicitly at the end.

---

## Domain Message Flow Modelling

Once you have candidate bounded contexts, you need to see how they **interact** for a specific scenario. A Domain Message Flow Diagram shows the commands, events, and queries flowing between actors, bounded contexts, and external systems, for exactly **one scenario at a time**.

A bounded context here can be a microservice, a module in a monolith, or any other deployable/ownable unit aligned to part of the domain.

### The three message types

- **Command** — sender tells the recipient to *do something*. Directed, one recipient, obligation to act.
- **Domain Event** — something that *happened*. Broadcast, any number of subscribers, **no obligation** on if/when/how they react.
- **Query** — sender asks the recipient for *information*, no side effect expected.

### Process

1. Pick **one scenario** (not the whole domain) — e.g., "customer places a booking for a container ship." List out the scenarios you eventually want, then diagram them one at a time.
2. Start with the actor, context, or external system that initiates the scenario.
3. Create the first message it sends.
4. Add the receiver and draw the connector.
5. Place the message's name (and contents) near the connector.
6. Move to the receiver as the new "current" node and repeat, until complete — including failure paths worth documenting.

Each message needs three things: its **name**, its **significant data/contents**, and its **order** in the flow.

### Producing the artifact

A numbered list of steps (`1. Customer → Booking Context: Place Booking {cargo, origin, destination}`) is often sufficient in text form.

---

## Bounded Context Canvas

The Bounded Context Canvas (ddd-crew) is the tool for designing and documenting a **single** bounded context in depth, once EventStorming has proposed it as a candidate. Fill in every section — an incomplete canvas usually means fuzzy thinking that hasn't been resolved yet.

### Sections

**Name.** Spend real effort here. Agreeing on a name forces the team to agree on what the context *is*.

**Description.** A few sentences, in **business language**, no technical detail: why does this context exist, what does it do.

**Strategic Classification.** Three independent lenses:

- *Domain:* **core** (competitive advantage), **supporting** (necessary, not differentiating), or **generic** (solved problem, prefer buying).
- *Business model:* **revenue generator** (customers pay directly), **engagement creator** (drives usage), or **compliance enforcer** (protects the business).
- *Evolution:* **genesis** (novel), **custom build** (in-house), **product** (off-the-shelf), or **commodity** (standardized utility).

**Domain Roles.** Note the behavioral traits this context plays.

**Inbound Communication.** Collaborations *initiated by others* toward this context.

**Outbound Communication.** Same structure, but for collaborations this context *initiates*.

**Ubiquitous Language.** A short glossary of key domain terms *as used inside this context*.

**Business Decisions.** The key business rules and policies this context owns.

### The three message types

- **Command** — sender tells recipient to *do something*.
- **Domain Event** — something that *happened*. Others may react, but there is **no obligation**.
- **Query** — one context asks another for information.

### Information and services provided

The public interface of a context breaks into:

- **Queryable Information** — what others can ask about.
- **Invokable Commands** — what others can ask this to do.
- **Published Events** — what this context broadcasts.
- **Reactive Jobs** — work this context starts on its own (scheduled or in response to another context's event).

### Producing the artifact

Render the canvas as a structured document with clear headers. A well-organized markdown document is usually enough for iterating on content quickly.

---

## Context mapping

Begin with the **core subdomains** — the highest-differentiation parts — and let the map grow outward.

### Context map patterns

Every integration between two contexts should get a named pattern — leaving it unnamed usually means the team hasn't actually agreed on the terms of the relationship.

| Pattern | What it means | When it fits |
|---|---|---|
| **Open Host Service** | Publishes a defined, open protocol/API for any number of consumers | You expect many consumers and want to standardize |
| **Conformist** | Downstream adopts upstream's model as-is, no translation | Fine when upstream is stable and translation isn't worth the cost |
| **Anti-Corruption Layer (ACL)** | Downstream builds a translation layer to protect its own model | The safe default whenever you must integrate with something noisy |
| **Shared Kernel** | Two contexts deliberately share a small common model/code | Rare — only when teams coordinate tightly |
| **Customer/Supplier** | Upstream supplier plans with downstream customer's needs in mind | Ongoing relationship where downstream has influence |
| **Partnership** | Two contexts/teams succeed or fail together | Genuinely coupled delivery |
| **Published Language** | A well-documented shared format/language | Often paired with OHS |
| **Separate Ways** | No integration at all | When the cost of integrating exceeds the benefit |
| **Big Ball of Mud** | A messy area with no clear boundaries | Recognize it, wall it off with an ACL, don't let it spread |

### Team relationships

The *organizational* relationship shapes what's technically realistic:

- **Mutually Dependent** — the two contexts must ship together to work.
- **Free** — genuinely independent.
- **Upstream/Downstream (U/D)** — upstream's decisions affect downstream — not just in code, but in schedule and responsiveness.

A context map is inherently visual — use a diagram when there are more than 2-3 contexts.

---

## Putting it together: a modeling process

A practical order for combining the techniques, based on the ddd-crew Starter Modelling Process. Treat it as a **loop**, not a one-way pipeline:

1. **Big Picture EventStorming** — explore the whole domain as a flow of events.
2. **Candidate Context Modelling** — group aggregates/events into named candidate bounded contexts.
3. **Domain Message Flow Modelling** — model how candidate contexts talk to each other per scenario.
4. **Bounded Context Canvas** — design each candidate context in full detail.
5. **Refined Context Exploration** — revisit and refine boundaries as understanding improves.

### Practical guidance for running this as an agent

- Don't skip straight to bounded contexts because they feel like "the real deliverable." Boundaries drawn without first walking the event timeline tend to mirror org charts rather than the actual domain.
- When the user's description is thin, that's diagnostic: missing events, unclear triggers, or "I'm not sure who decides that" are exactly the gaps steps 1 and 8 surface. Ask about them directly.
- If the user already has a system and wants you to *evaluate* their boundaries, run the same loop in reverse: reconstruct the event timeline and message flows from what they describe, then check whether existing service boundaries align with the pivotal points and aggregate clusters you find.
- Keep core-subdomain effort proportional: spend the most design depth on core contexts, and be comfortable being terse about generic ones.

---

**Sources:** EventStorming (Alberto Brandolini). The Bounded Context Canvas and the DDD Starter Modelling Process are from the ddd-crew (github.com/ddd-crew). Use them as canonical references when more depth is needed.
