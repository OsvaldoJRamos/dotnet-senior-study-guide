# Anti-Patterns

An **anti-pattern** is a common solution to a recurring problem that *looks* reasonable but ends up being more harmful than helpful. Recognizing them is a senior-level skill: juniors write them by accident, seniors spot them in PRs and propose incremental refactoring.

This file groups the ones you will actually be asked about, split into **code-level** (inside a class/module) and **architecture-level** (across services/databases).

## Code-level anti-patterns

### God Class / God Object

A single class that knows and does too much. It pulls in unrelated responsibilities until changing anything in the codebase means touching it.

**Smell:** hundreds/thousands of lines, mixed concerns (persistence, validation, business rules, formatting), and every feature ends up depending on it.

**Fix:** extract cohesive collaborators (SRP). Pull out policies, repositories, formatters, validators into separate classes. Often signals missing domain concepts.

### Anemic Domain Model

A domain object with fields and getters/setters but no real behavior — all the logic lives in "service" classes around it.

> Martin Fowler: *"The fundamental horror of this anti-pattern is that it's so contrary to the basic idea of object-oriented design; which is to combine data and process together."* And: *"If all your logic is in services, you've robbed yourself blind."* (`martinfowler.com/bliki/AnemicDomainModel.html`)

**Fix:** push invariants and behavior back into the domain objects. An `Order` should know how to `Confirm()`, `Cancel()`, and refuse invalid transitions — not expose setters for status.

See also: [Domain-Driven Design](13-ddd.md).

### Feature Envy

A method that uses data from another class more than its own — usually a string of `other.X`, `other.Y`, `other.Z` calls.

**Fix:** move the method to the class whose data it envies. Classic Refactoring (Fowler) example.

### Primitive Obsession

Using primitive types (`string`, `int`, `decimal`) where a meaningful value object would be clearer and safer.

```csharp
// Smell
public void Transfer(string fromAccount, string toAccount, decimal amount, string currency) { ... }

// Better
public void Transfer(AccountId from, AccountId to, Money amount) { ... }
```

Value objects catch bugs at compile time (can't pass an `Email` where a `Name` is expected), centralize validation, and make the intent obvious.

### Shotgun Surgery

One logical change forces edits scattered across many classes. The *opposite* of SRP violations: here the concern is spread too thin.

**Fix:** consolidate the scattered pieces into one cohesive module. If "adding a new payment method" means touching 12 files, the payment concept is fragmented.

### Premature Optimization

Optimizing before profiling. Paying the cost of complexity (less readable code, caches with invalidation bugs, unsafe async tricks) for a speed-up you never measured.

> Donald Knuth (1974): *"Premature optimization is the root of all evil."*

**Fix:** write the clearest code first, profile under realistic load, then optimize the actual hotspot. Most systems are slow in one or two places, not everywhere.

## Architecture-level anti-patterns

### Distributed Monolith

Microservices in name only — services share a database, call each other synchronously, deploy together, and must be released in lockstep. You paid the operational cost of microservices with none of the independence benefits.

**Signals:**
- Any single user action produces a chain of synchronous HTTP calls across 4+ services.
- A feature change requires coordinated PRs in multiple repos.
- Services share a database or tables.
- "We can't deploy service A until service B is updated."

**Fix:** identify true bounded contexts, replace synchronous fan-out with events (pub/sub), give each service its own data store. See [Microservices](09-microservices.md) and [Messaging](10-messaging.md).

### Database-as-Integration (Shared Database)

Two or more services write to the same tables. The database becomes the integration contract — schema changes break every consumer invisibly.

**Fix:** each service owns its schema. Integrate via APIs, events, or a replicated read model (e.g., CDC from one service's DB into another's read store). See [Polyglot Persistence](../09-data-access/09-polyglot-persistence.md).

### Golden Hammer

"When all you have is a hammer, everything looks like a nail." Using the same favorite technology (microservices, Kafka, GraphQL, Kubernetes, MongoDB, Rust, …) regardless of whether the problem needs it.

**Fix:** match the tool to the problem. See [Technology Selection](../16-soft-skills/07-technology-selection.md) for an evaluation framework.

### Big Ball of Mud

A system with no discernible architecture — ad hoc modifications accreted over years, cross-cutting dependencies, no clear layering.

> Brian Foote and Joseph Yoder (1997): *"A BIG BALL OF MUD is haphazardly structured, sprawling, sloppy, duct-tape-and-baling-wire, spaghetti code jungle."* (`laputan.org/mud/`)

**Fix:** usually incremental — introduce seams (interfaces, facades, anti-corruption layers), carve out bounded contexts, strangle the legacy. Full rewrites rarely pay off.

### God Service / Mega-Service

The microservices version of the God Class. One service owns most of the business logic; the others are CRUD wrappers.

**Fix:** re-evaluate the domain boundaries using event storming. A service owning "Orders + Payments + Shipping + Inventory" is almost never right.

### Chatty I/O / N+1 across services

Equivalent of EF's N+1 but across the network: loading a list of orders, then calling a service for each order's customer. Turns a 50 ms query into a 2 s page load.

**Fix:** bulk APIs, GraphQL or BFF aggregation, or denormalize the needed fields into the caller's read store.

### Over-Abstraction (speculative generality)

Layers of generic interfaces, base classes, and extension points designed for flexibility that never materializes. You pay the readability cost forever for an optionality you'll never exercise.

> YAGNI: *"You Aren't Gonna Need It."* Build for today's requirements plus the known near-term ones; not for the imaginary future.

## Why this matters

Most production fires trace back to an anti-pattern someone introduced months or years earlier — not to a single bad commit. The interview signal is whether a candidate can:

1. **Name** the anti-pattern from a short code or architecture snippet.
2. **Explain** why it's harmful in concrete terms (deploy friction, latency, bug classes).
3. **Propose incremental refactoring** — not a full rewrite.

## Rule of thumb

> Anti-patterns are almost never introduced on purpose. They accrete. The cure is also accretive: one seam, one extracted class, one bounded context at a time — not a big-bang rewrite.

---

[← Previous: Idempotency and Race Conditions](11-idempotency-and-race-conditions.md) | [Back to index](README.md) | [Next: Domain-Driven Design →](13-ddd.md)
