# Application Architectural Patterns (MVC, MVVM, MVP, Layered, Hexagonal)

Every application has to decide **how the UI, business logic, and data access relate**. These patterns are the canonical answers. [Clean Architecture](07-clean-architecture.md) is one specific synthesis of the ideas; this file covers the surrounding family so you can place it in context.

## The common goal

All of these patterns try to:

1. **Separate concerns** — UI changes shouldn't force business-logic changes, and vice versa.
2. **Make code testable** — swap the UI or persistence for fakes without rewriting the domain.
3. **Constrain dependencies** — the domain should not know about HTTP, databases, or UI frameworks.

They differ in *how* they draw the lines and in which layer "owns" the coordination.

## Layered (n-tier) Architecture

The oldest and most common: code is organized as a stack of horizontal layers, each depending only on the one below.

```
┌──────────────────────┐
│  Presentation        │  ← Controllers, Views, API endpoints
├──────────────────────┤
│  Application         │  ← Use cases, orchestration
├──────────────────────┤
│  Domain              │  ← Entities, business rules
├──────────────────────┤
│  Infrastructure      │  ← EF Core, HTTP clients, message bus
└──────────────────────┘
```

**Rules:** top-down dependency only; each layer exposes interfaces/services consumed by the layer above.

**Strength:** easy to understand, familiar to every team.

**Weakness:** the Domain layer typically still depends on Infrastructure (because Entities are EF-mapped, Repositories return EF `DbSet`), which defeats the separation. That's what Hexagonal / Clean Architecture fix.

## MVC (Model-View-Controller)

From the Smalltalk tradition of the 1970s; popularized for the web by Rails and ported to .NET as ASP.NET MVC.

```
┌────────┐        user input        ┌────────────┐
│  View  │◄──────────────┬──────────│ Controller │
└────────┘               │          └──────┬─────┘
     ▲                   │                 │
     │ updates           │ invokes         │
     │                   │                 ▼
     └───────── Model ◄──┘             (domain / services)
```

- **Model**: the data and business logic.
- **View**: renders the model to the user.
- **Controller**: receives input, calls the model, selects the view to render.

> In classic MVC the View observes the Model directly. In **server-side MVC** (ASP.NET MVC, Spring MVC, Rails), the Controller assembles the model and passes it to the View — the View doesn't observe; it's a one-shot render. This is technically "Model2" MVC but the name stuck.

**When to reach for it:** traditional server-rendered web apps, REST-ish APIs where controllers map HTTP requests to commands/queries.

## MVVM (Model-View-ViewModel)

Designed for UI platforms with **data binding** — WPF, Xamarin, MAUI, and heavily influential in Angular's two-way binding pattern.

```
┌────────┐  two-way binding   ┌───────────┐        ┌───────┐
│  View  │◄──────────────────►│ ViewModel │───────►│ Model │
└────────┘                    └───────────┘        └───────┘
```

- **Model**: domain objects and services.
- **View**: XAML/HTML markup with declarative bindings; no code-behind logic.
- **ViewModel**: exposes observable properties and commands the View binds to. Contains view-state logic, not view rendering.

**Strength:** the View is pure markup; the ViewModel is plain .NET and trivially unit-testable.

**When:** desktop/mobile XAML apps, Angular (where components play a ViewModel-like role).

## MVP (Model-View-Presenter)

Similar to MVC, but the **View is passive** — it holds no logic and exposes an interface the Presenter drives directly.

```
┌────────────┐      calls       ┌───────────┐       ┌───────┐
│    View    │◄────────────────►│ Presenter │──────►│ Model │
│ (passive)  │   updates        └───────────┘       └───────┘
└────────────┘
```

- The View is dumb — just "show this string here, enable that button".
- The Presenter handles all UI decisions and talks to the Model.
- Testing the Presenter = inject a fake View that implements the View's interface.

**When:** Winforms, older Silverlight, legacy platforms without data binding. Mostly historical today but still shows up in interview questions.

## Hexagonal Architecture (Ports and Adapters)

Alistair Cockburn, 2005. Instead of a vertical stack, the application is a **central core** surrounded by **ports** (interfaces it defines) and **adapters** (implementations that connect ports to the outside world).

> *"aims at creating loosely coupled application components that can be easily connected to their software environment by means of ports and adapters."* (`en.wikipedia.org/wiki/Hexagonal_architecture_(software)`)

```
            ┌───────── Web adapter ─────────┐
            │      (REST controllers)       │
            └──────────────┬────────────────┘
                           │ drives
                           ▼
┌──────────────────┐   ┌─────────┐    ┌──────────────────┐
│ CLI adapter      │─► │  CORE   │ ◄──│ Test adapter     │
│ (console handler)│   │         │    │ (fake)           │
└──────────────────┘   │ Domain  │    └──────────────────┘
                       │ + ports │
                       └────┬────┘
                            │ driven
                            ▼
                   ┌────────────────────┐
                   │ EF Core adapter    │
                   │ (implements ports) │
                   └────────────────────┘
```

- **Primary (driving) adapters** call into the core: HTTP controllers, CLI, tests.
- **Secondary (driven) adapters** are called by the core: DB repository, message publisher.
- Ports are interfaces defined by the core; adapters implement them.

**Strength:** the core has **zero** knowledge of HTTP, EF, or Kafka. You swap persistence, transport, or framework without touching business logic. Testing = wire in fake adapters.

**When:** any domain where you expect to survive multiple UI frameworks or infrastructure changes, or where testing discipline matters.

## Clean Architecture (Uncle Bob, 2012)

A synthesis that borrows from Hexagonal, Onion (Jeffrey Palermo), and Screaming Architecture. Same core idea: **dependencies point inward**, the domain is at the center, everything else plugs in.

See [Clean Architecture](07-clean-architecture.md) for the full breakdown.

## How to choose

| Situation | Pattern that fits |
|---|---|
| Server-rendered web app, REST API | **MVC** (classic) or Clean + Web adapter |
| Desktop/mobile with rich data binding (WPF, MAUI) | **MVVM** |
| Legacy Winforms or platforms without binding | **MVP** |
| Small-to-medium business app, fast time-to-market | **Layered** (accept the EF-in-domain leak, ship faster) |
| Complex domain, long-lived system, multiple UIs/adapters | **Hexagonal / Clean** |
| You're starting a microservice with one small bounded context | Layered is usually enough — Clean is overkill per service |

## Pitfalls

- **Calling it "Clean Architecture" just because you have folders named Application/Domain/Infrastructure.** Clean Architecture is about dependency direction. If your Domain layer references EF Core types, it's not Clean, it's Layered with ambition.
- **MVVM but with logic in code-behind.** The entire point is that the View is logic-free. If the code-behind has `if (SomeBusinessCondition)`, you rebuilt MVP accidentally.
- **Fighting the framework.** ASP.NET Core is a Web adapter in a Clean/Hexagonal sense. You don't need a custom abstraction over `IActionResult` — let the framework be the adapter.
- **Over-architecting a CRUD app.** Layered + EF Core beats five-project Clean Architecture for a form-over-data internal tool. The cost of the ceremony only pays off when the domain is complex.

## Rule of thumb

> The canonical patterns are tools for managing **dependency direction** and **UI/domain coupling**. Pick the lightest one that separates the concerns your app actually has — and graduate to Hexagonal/Clean only when the domain's complexity earns it.

---

[← Previous: Event Sourcing](15-event-sourcing.md) | [Back to index](README.md) | [Next: Design Docs, C4 and ADRs →](17-design-docs-c4-adr.md)
