---
name: clean-architecture-ios
description: Use this skill when designing, reviewing, or refactoring Clean Architecture composition in Swift or iOS, including dependency direction, composition roots, dependency injection, use cases, repositories, data sources, model boundaries, and feature composition. Use concrete types by default. Introduce protocols only when a current, explicit architectural, substitution, integration, or module-boundary requirement justifies them. Do not use this skill for generic folder organization or ordinary SwiftUI structure unless dependency boundaries are part of the task.
---

# Clean Architecture for Swift and iOS

## Purpose

Design pragmatic Clean Architecture composition for Swift and iOS applications.

Focus on clear responsibilities, explicit dependency construction, justified boundaries, and the simplest design that satisfies current requirements.

Do not reproduce textbook layers mechanically. Do not introduce protocols, use cases, repositories, data sources, mappers, factories, or Swift modules without a visible reason.

## When to use this skill

Use this skill when the task involves:

- composing an iOS feature or application;
- deciding how Views, ViewModels, Use Cases, Repositories, services, and Data Sources interact;
- defining Presentation, Application, Domain, Data, or Infrastructure boundaries;
- creating a composition root, application container, or feature factory;
- deciding whether a Use Case or Repository is useful;
- reviewing dependency injection and dependency direction;
- isolating networking, persistence, caches, Keychain, system APIs, or external SDKs;
- deciding whether DTOs, persistence models, domain models, and view data should be separate;
- structuring feature modules or Swift packages;
- removing protocol-heavy or layer-heavy boilerplate.

## When not to use this skill

Do not use this skill for:

- basic Swift or SwiftUI syntax;
- UI styling and layout;
- generic folder naming without dependency implications;
- backend architecture with no Swift or iOS context;
- performance analysis unless architecture composition is directly involved.

## Core principles

- Treat Clean Architecture primarily as a dependency rule, not as a required folder structure.
- Use concrete types by default.
- Introduce abstractions only when they solve a current and explicit problem.
- Build concrete dependencies in an explicit composition root.
- Keep application and domain policy independent from infrastructure when that boundary provides real value.
- Do not separate Application and Domain automatically when their responsibilities are not meaningfully different.
- Prefer logical boundaries before introducing physical Swift modules.
- Preserve the architecture already present unless changing it solves a concrete problem.
- For an existing codebase, prefer the smallest change that corrects dependency direction or removes accidental complexity.

## Composition levels

Do not force every feature through the same number of layers.

A simple feature may use:

```text
View
  ↓
ViewModel
  ↓
Concrete service
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
  ↓
Infrastructure
```

Use only the levels required by the problem.

## Default abstraction policy

### Prefer concrete types

Start with concrete structs, classes, actors, functions, values, and closures.

Dependency injection does not require protocols.

Prefer:

```swift
struct ProductRepository {
    let apiClient: ProductAPIClient

    func products() async throws -> [Product] {
        let response = try await apiClient.products()

        return response.items.map { dto in
            Product(
                id: dto.id,
                title: dto.title
            )
        }
    }
}
```

Avoid creating conventional pairs by default:

```swift
protocol ProductRepositoryProtocol { ... }
final class DefaultProductRepository: ProductRepositoryProtocol { ... }

protocol LoadProductsUseCaseProtocol { ... }
final class DefaultLoadProductsUseCase: LoadProductsUseCaseProtocol { ... }
```

Concrete dependencies can be injected through initializers, composed explicitly, and tested directly.

### Introduce protocols only for a visible need

Add a protocol when at least one of these conditions exists:

* multiple implementations already exist;
* a second implementation is planned and the shared capability is known from concrete requirements;
* an inner module must own a capability implemented by an outer module;
* distinct live, preview, offline, cached, platform, or test implementations are genuinely required and cannot be represented more simply through values or configuration;
* a third-party SDK or system API needs a controlled isolation boundary;
* the protocol defines a stable public or shared capability between independently evolving modules;
* a test seam cannot be achieved more simply with a concrete type, value, closure, configuration, or in-memory implementation.

Do not add a protocol only because:

* another implementation may exist someday;
* the code should appear testable;
* every dependency is expected to have a mock;
* dependency injection is being used;
* Clean Architecture is being applied;
* a naming convention expects `Protocol` and `Default`.

Prefer defining a justified protocol in the module that consumes and owns the required capability.

Place it elsewhere only when the protocol is itself a deliberate shared, integration, or public contract.

## Core workflow

### 1. Inspect the existing system

Before proposing a new composition, identify:

* current construction points;
* existing dependency direction;
* module boundaries;
* ownership and lifetimes;
* tests and test seams;
* external integrations;
* migration cost.

Do not redesign an existing feature from an idealized blank slate unless the user asks for a greenfield design.

### 2. Clarify the scenario

Identify:

* the user-facing operation;
* application and business rules;
* required data;
* external effects;
* ownership;
* expected lifetime;
* actual reasons for change.

Do not begin by creating folders, protocols, or layers.

### 3. Draw the dependency graph

Show the proposed graph before expanding the implementation.

For example:

```text
ProductListView
    ↓
ProductListViewModel
    ↓
LoadProducts
    ↓
ProductRepository
    ↓
ProductAPIClient
```

For every dependency, ask:

* Why is it needed?
* Can it remain concrete?
* What current requirement justifies an abstraction?
* Who creates it?
* How long should it live?
* Does the dependency point in the intended direction?

### 4. Separate policy from mechanisms

Keep application and business policy independent from concrete infrastructure where that separation provides value.

Inner logic should not construct:

* `URLSession` clients;
* SwiftData or Core Data containers;
* Keychain implementations;
* analytics SDKs;
* caches;
* file systems;
* external services.

Do not create separate domain models when existing models already have correct semantics and do not leak transport, storage, SDK, or UI concerns.

### 5. Decide whether a Use Case is needed

Create a Use Case when it:

* enforces application or business rules;
* validates an operation;
* coordinates multiple dependencies;
* defines ordering, rollback, compensation, or transaction behavior;
* represents an important named application action;
* is reused from multiple entry points;
* provides a deliberate application boundary.

Name Use Cases after user or system actions, not implementation details.

Prefer names such as:

```text
LoadProducts
AddProductToCart
SubmitOrder
RefreshSession
```

Do not create one Use Case for every Repository method automatically.

When an operation contains no policy, coordination, reuse, or useful boundary, a ViewModel may depend directly on a concrete Repository or service.

### 6. Decide whether a Repository is needed

Do not assume that data access requires a Repository.

A concrete client or service may be sufficient when it already exposes an application-oriented API and no additional data policy exists.

Create a Repository when it hides meaningful data-access policy, such as:

* remote and local coordination;
* caching and invalidation;
* synchronization;
* offline fallback;
* persistence;
* freshness decisions;
* conflict resolution;
* combining several data mechanisms.

DTO mapping may happen inside a Repository boundary, but mapping alone does not justify a Repository.

Do not create:

* one Repository per endpoint;
* one global Repository for unrelated domains;
* a generic CRUD Repository that removes domain language and invariants;
* a Repository that only renames a client call.

Prefer domain-oriented APIs over transport-oriented APIs.

### 7. Add Data Sources only when useful

Separate remote, local, cache, or persistence Data Sources when:

* a Repository coordinates several mechanisms;
* the mechanisms have distinct ownership or lifetimes;
* independent replacement or testing is valuable;
* the separation hides a meaningful infrastructure detail.

Do not add Data Source wrappers that only forward calls.

### 8. Define model boundaries deliberately

Separate models only when their responsibilities differ:

* DTO for transport;
* persistence model for storage;
* domain or application model for business meaning;
* view data for UI-specific representation.

Do not create identical model copies because every layer is expected to own one.

Map data where semantics change.

Do not create a dedicated mapper type when inline mapping is simple and local. Introduce a mapper when transformation is complex, reused, independently tested, or owned by a clear boundary.

### 9. Build the composition root

Create concrete dependencies in the application composition root or in delegated feature-level composition points that receive shared dependencies from it.

Suitable composition points include:

* application startup;
* an application container;
* a feature factory;
* a coordinator;
* a scene builder;
* a module assembly entry point.

Do not construct infrastructure dependencies inside Views, ViewModels, Use Cases, or domain entities.

Example:

```swift
@MainActor
struct ProductsFeatureFactory {
    let repository: ProductRepository

    func makeProductList() -> ProductListView {
        let loadProducts = LoadProducts(
            repository: repository
        )

        let viewModel = ProductListViewModel(
            loadProducts: loadProducts
        )

        return ProductListView(viewModel: viewModel)
    }
}
```

Create the Repository at the level that owns its intended lifetime.

### 10. Define dependency lifetimes

Classify dependencies as:

* application-scoped;
* authenticated-session-scoped;
* feature-scoped;
* screen-scoped;
* operation-scoped.

Do not make every dependency a singleton.

Do not recreate expensive shared infrastructure for every screen.

### 11. Review physical module boundaries

Use logical boundaries first.

Introduce SwiftPM targets or separate modules when compile-time enforcement, ownership, reuse, visibility control, build structure, or team boundaries justify the additional complexity.

Do not create one module per architectural type or layer by convention.

### 12. Remove accidental complexity

Before finalizing, ask:

* Can any protocol become a concrete type?
* Can any Use Case be removed?
* Does any Repository merely forward a call?
* Are Data Sources adding policy or only indirection?
* Are model types unnecessarily duplicated?
* Does any factory only wrap one initializer?
* Is every abstraction based on a current requirement?
* Is the proposed change smaller than a complete rewrite?
* Is the dependency graph easier to understand than the original code?

Prefer the simplest composition that preserves the required boundaries.

## Decision rules

* Prefer initializer injection for required dependencies.
* Use concrete types by default.
* Prefer values and closures for small injectable behaviors.
* Use protocols only for meaningful capability, substitution, integration, or module boundaries.
* Prefer protocol ownership by the consuming module.
* Use a Use Case for application policy or coordination, not automatic forwarding.
* Use a Repository for meaningful data-access policy, not every API call.
* Prefer feature-level composition over one global container that builds every screen.
* Prefer explicit construction over hidden dependency lookup.
* Prefer domain-oriented APIs over networking-oriented APIs.
* Prefer testing concrete behavior over creating a mock for every type.
* Prefer incremental correction over speculative redesign.

## Gotchas

* Do not treat `Presentation`, `Application`, `Domain`, and `Data` folders as proof of Clean Architecture.
* Do not create `Protocol` and `Default...` pairs by convention.
* Do not claim that dependency injection requires protocols.
* Do not create one Use Case per Repository method.
* Do not create one Repository per endpoint.
* Do not introduce generic CRUD repositories when domain capabilities differ.
* Do not place Use Cases inside Repository implementations.
* Do not construct infrastructure inside ViewModels or Use Cases.
* Do not introduce a service locator as a substitute for explicit composition.
* When reviewing legacy service-locator code, prefer an incremental migration toward explicit initializer injection.
* Do not duplicate models without semantic differences.
* Do not split every logical layer into a Swift package.
* Do not preserve an abstraction after its original reason disappears.

## References

Read these only when relevant:

* `references/use-cases-and-repositories.md` — read when deciding whether a Use Case, Repository, Repository protocol, service, or Data Source is needed, or when reviewing thin pass-through layers.
* `references/composition-root.md` — read when building application containers, feature factories, navigation composition, dependency lifetimes, SwiftUI previews, test composition, or session-scoped graphs.
* `references/boundaries-and-models.md` — read when separating DTOs, persistence models, domain models, view data, external SDKs, logical layers, or Swift module boundaries.

## Output expectations

Match the response depth to the request.

For a focused question, return the relevant decision, its rationale, and a minimal example when useful.

For a full architecture design or review, include:

1. The feature and user-facing scenario.
2. The proposed dependency direction.
3. A compact dependency graph.
4. Responsibilities of the main components.
5. Dependencies that should remain concrete.
6. Protocols introduced and the specific reason for each.
7. Use Cases that provide meaningful policy or coordination.
8. Layers or abstractions that should be omitted.
9. Composition-root placement.
10. Important dependency lifetimes.
11. A minimal Swift example.
12. Testing strategy, migration cost, and trade-offs.

When reviewing existing code, identify:

* speculative protocols;
* unnecessary Use Cases;
* pass-through Repositories or Data Sources;
* hidden dependency construction;
* incorrect dependency direction;
* infrastructure leaking into application logic;
* unnecessary model duplication;
* unclear ownership or lifetime;
* changes that are broader than necessary.

## Validation

Before presenting the final design, verify that:

* concrete implementations are created in composition;
* every abstraction has a current and explicit reason;
* removing each protocol was considered;
* every Use Case provides policy, coordination, reuse, or a deliberate boundary;
* Repository APIs use application or domain language;
* external DTOs and SDK types do not leak across inappropriate boundaries;
* dependency lifetimes are explicit where they matter;
* the design can be tested without mocking every type;
* the proposed change respects the existing codebase;
* the result is no more complex than the problem requires.
