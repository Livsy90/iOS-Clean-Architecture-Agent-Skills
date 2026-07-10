# iOS Clean Architecture Agent Skills

<p align="center">
<img width="320" src="https://github.com/user-attachments/assets/a71688d2-54a3-4ea5-8362-e4659b3d46c6"/>
</p>

An Agent Skill for designing, reviewing, and simplifying Clean Architecture composition in Swift and iOS applications.

The skill focuses on dependency direction, composition roots, Use Cases, Repositories, model boundaries, feature assembly, and Swift module design.

Its default position is intentionally pragmatic:

> Use concrete types by default. Introduce protocols only when a current and explicit requirement justifies them.

The goal is not to reproduce textbook layers mechanically. The goal is to build the simplest architecture that preserves meaningful boundaries.

## What this skill helps with

Use this skill when working on:

- Clean Architecture composition for an iOS feature or application;
- dependency direction between Presentation, Application, Domain, Data, and Infrastructure;
- composition roots, application containers, and feature factories;
- dependency injection without protocol-first design;
- deciding whether a Use Case is useful;
- deciding whether a Repository or Data Source is needed;
- reviewing thin pass-through layers;
- DTO, persistence, domain, and presentation model boundaries;
- external SDK isolation;
- dependency lifetimes;
- session-scoped dependency graphs;
- SwiftUI preview and test composition;
- logical boundaries and Swift Package modularization;
- removing unnecessary protocols, wrappers, mappers, and modules.

## Core philosophy

### Concrete types by default

Dependency injection does not require protocols.

Start with concrete structs, classes, actors, values, functions, and closures. Introduce a protocol only when there is a visible need such as:

- multiple implementations that already exist;
- a known second implementation based on current requirements;
- a real module boundary that requires dependency inversion;
- a stable public or integration contract;
- explicit live, offline, cached, platform, or test implementations;
- an external capability that cannot be isolated more simply.

Do not create protocols only because:

- another implementation may exist someday;
- every dependency is expected to have a mock;
- the project uses dependency injection;
- Clean Architecture is being applied;
- a naming convention expects `Protocol` and `Default`.

### Optional layers

Use Cases, Repositories, services, Data Sources, mappers, factories, and modules are optional tools.

A simple feature may use:

```text
View
  ↓
ViewModel
  ↓
Concrete service or client
````

A feature with meaningful data-access policy may use:

```text
View
  ↓
ViewModel
  ↓
Repository
  ↓
API, database, cache, or SDK
```

A feature with application policy or coordination may use:

```text
View
  ↓
ViewModel
  ↓
Use Case
  ↓
Repository or service
```

Use only the layers required by the problem.

### Boundaries before folders

Clean Architecture is primarily about dependency direction and ownership, not folder names.

Directories called `Presentation`, `Domain`, and `Data` do not prove that an architecture is clean.

Likewise, separate Swift packages are not required for every logical layer. Start with logical boundaries and introduce physical modules only when compile-time enforcement, ownership, visibility, reuse, or team structure justifies the additional complexity.

## Skill structure

```text
clean-architecture-ios/
├── SKILL.md
└── references/
    ├── use-cases-and-repositories.md
    ├── composition-root.md
    └── boundaries-and-models.md
```

### `SKILL.md`

The main skill file contains:

* activation and exclusion criteria;
* the concrete-first abstraction policy;
* the core architecture workflow;
* decision rules;
* common gotchas;
* output expectations;
* validation requirements;
* routing to the relevant references.

### `references/use-cases-and-repositories.md`

Read when deciding:

* whether a Use Case is needed;
* whether a Repository is needed;
* whether a Repository should have a protocol;
* whether a service or Data Source adds value;
* whether a layer is only forwarding calls;
* how to test concrete dependencies without protocol-heavy mocks.

### `references/composition-root.md`

Read when working with:

* application composition roots;
* application containers;
* feature factories;
* dependency lifetimes;
* authenticated session graphs;
* navigation composition;
* SwiftUI ownership;
* previews;
* test graph assembly;
* deterministic teardown.

### `references/boundaries-and-models.md`

Read when deciding:

* whether DTOs should remain separate from application models;
* whether persistence models can be used directly;
* where mapping belongs;
* when view data is useful;
* how to isolate external SDKs;
* whether to create a Swift module;
* how public feature entry points should be designed.

## Example prompts

```text
Review this iOS feature and tell me whether its Use Cases and Repositories add value.
```

```text
Design a composition root for this SwiftUI application without introducing protocols by default.
```

```text
Should this Repository be a protocol, or can it remain a concrete type?
```

```text
Review this dependency graph and identify thin pass-through layers.
```

```text
Should these DTO, SwiftData, domain, and view models remain separate?
```

```text
How should authenticated and unauthenticated dependency graphs be composed?
```

```text
Should this feature become a separate Swift package?
```

## Expected behavior

When designing architecture, the skill should:

1. inspect the existing dependency graph;
2. identify the user-facing operation and current constraints;
3. propose the smallest useful composition;
4. keep concrete dependencies concrete unless abstraction is justified;
5. explain every introduced protocol;
6. identify unnecessary Use Cases, Repositories, Data Sources, or mappers;
7. define the composition root and important dependency lifetimes;
8. preserve correct dependency direction;
9. include a minimal Swift example when useful;
10. discuss testing, migration cost, and trade-offs.

For focused questions, the response should stay focused rather than returning a complete architecture report.

## Non-goals

This skill does not enforce:

* a protocol for every dependency;
* a Use Case for every ViewModel action;
* a Repository for every endpoint;
* one model per layer;
* one mapper per model pair;
* one Swift package per architectural layer;
* a global dependency container;
* a service locator;
* a specific dependency-injection framework;
* a single architecture suitable for every project.

## Installation

Place the skill in a repository-scoped Agent Skills directory:

```text
.agents/
└── skills/
    └── clean-architecture-ios/
```

Or install it in the user-level skills directory:

```text
~/.agents/skills/clean-architecture-ios/
```

The final structure should look like this:

```text
.agents/skills/clean-architecture-ios/
├── SKILL.md
└── references/
    ├── use-cases-and-repositories.md
    ├── composition-root.md
    └── boundaries-and-models.md
```

## Design principles

* Prefer explicit initializer injection.
* Prefer concrete types by default.
* Prefer capabilities over generic abstractions.
* Prefer domain-oriented APIs over transport-oriented APIs.
* Prefer feature-level composition over one global screen factory.
* Prefer logical boundaries before physical modularization.
* Prefer application-owned models over leaked DTO or SDK types.
* Prefer stable semantic identity over generated presentation identity.
* Prefer focused test seams over a mock for every type.
* Prefer incremental correction over speculative redesign.
* Prefer the simplest composition that preserves the required boundaries.

## Validation

A proposed architecture should be rejected or revised when:

* concrete infrastructure is created inside a ViewModel or Use Case;
* protocols exist without a current explicit reason;
* a Use Case only mirrors a Repository method without boundary value;
* a Repository only renames an API client call;
* Data Sources add no mapping, policy, lifecycle, isolation, or error translation;
* DTO or SDK types leak across inappropriate boundaries;
* Domain imports transport or persistence types;
* dependency lifetimes are unclear;
* user-specific state survives session replacement unexpectedly;
* SwiftUI evaluation recreates identity-sensitive dependencies;
* physical modules add more cost than protection;
* the resulting graph is harder to explain than the original implementation.

## License

MIT
