# Use Cases, Repositories, Services, and Data Sources

Use this reference when deciding whether a Use Case, Repository, Repository protocol, service, or Data Source is needed, or when reviewing thin pass-through layers.

## Contents

- [Core model](#core-model)
- [Use Cases](#use-cases)
- [Repositories](#repositories)
- [Repository protocols](#repository-protocols)
- [Services](#services)
- [Data Sources](#data-sources)
- [Thin pass-through layers](#thin-pass-through-layers)
- [Testing without protocol-first design](#testing-without-protocol-first-design)
- [Review checklist](#review-checklist)

## Core model

Use Cases, Repositories, services, and Data Sources are optional architectural tools.

Do not introduce them because a Clean Architecture diagram contains them. Introduce each component only when it owns a distinct responsibility, protects a meaningful boundary, or makes an existing dependency easier to reason about.

Start with the simplest composition that satisfies the feature.

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

A Repository may coordinate several mechanisms:

```text
Repository
├── Remote Data Source
├── Local Data Source
└── Cache
```

Do not force every feature through the longest chain.

Choose `struct`, `final class`, or `actor` from ownership, identity, mutability, and isolation requirements. Do not use one type category mechanically for every architectural component.

## Use Cases

A Use Case represents a meaningful application operation.

It should answer:

> What user or system action is being performed?

Good names describe actions:

```text
LoadProducts
AddProductToCart
SubmitOrder
RefreshSession
ApplyPromoCode
SynchronizeFavorites
```

Avoid names based on implementation details:

```text
ExecuteRepository
HandleAPIResponse
PerformDataOperation
CallProductService
```

### Create a Use Case when it owns policy

A Use Case is useful when it:

* enforces application or business rules;
* validates an operation;
* coordinates multiple dependencies;
* defines ordering between operations;
* handles rollback or compensation;
* establishes a transaction boundary;
* combines several capabilities;
* represents an important product action;
* is reused from multiple entry points;
* provides a deliberate application boundary.

Example:

```swift
struct AddProductToCart {
    let productRepository: ProductRepository
    let cartRepository: CartRepository

    func callAsFunction(
        productID: Product.ID,
        quantity: Int
    ) async throws {
        guard quantity > 0 else {
            throw CartError.invalidQuantity
        }

        let product = try await productRepository.product(
            id: productID
        )

        guard product.isAvailable else {
            throw CartError.productUnavailable
        }

        try await cartRepository.add(
            productID: productID,
            quantity: quantity
        )
    }
}
```

This Use Case is justified because it owns validation, availability rules, and coordination.

The local availability check is illustrative. An authoritative backend must still validate server-owned constraints when the operation is submitted.

### Do not create a Use Case automatically

A Use Case may be unnecessary when it only forwards a call:

```swift
struct LoadProducts {
    let repository: ProductRepository

    func callAsFunction() async throws -> [Product] {
        try await repository.products()
    }
}
```

This is not inherently wrong. It may still provide value when:

* the application exposes a deliberate operation boundary;
* the operation is reused from several entry points;
* Presentation should depend on a narrow command or query API;
* future policy is already known from current requirements;
* the Use Case protects consumers from a broader Repository API.

Remove or avoid it when none of those reasons applies.

In a simple feature, this may be sufficient:

```swift
@MainActor
final class ProductListViewModel {
    private let repository: ProductRepository

    init(repository: ProductRepository) {
        self.repository = repository
    }
}
```

Do not create one Use Case per Repository method by convention.

### Use Case responsibilities

A Use Case should depend on the capabilities required to perform the operation.

It should not:

* construct Repositories;
* construct API clients;
* create database containers;
* import SwiftUI or UIKit for business decisions;
* expose transport DTOs;
* become a general utility container;
* own unrelated operations.

Prefer small operation-focused types over one large `ApplicationUseCases` object.

## Repositories

A Repository exposes data or state using application or domain language.

It hides meaningful policy related to obtaining, storing, combining, or synchronizing data.

Example:

```swift
struct ProductRepository {
    let remote: ProductRemoteDataSource
    let local: ProductLocalDataSource

    func products() async throws -> [Product] {
        do {
            let products = try await remote.products()
            try? await local.save(products)
            return products
        } catch {
            let cachedProducts = try await local.products()

            guard !cachedProducts.isEmpty else {
                throw error
            }

            return cachedProducts
        }
    }
}
```

This Repository is justified because it owns remote/local coordination and fallback policy.

The cache write in this example is best-effort. Production behavior should match the product's consistency, durability, and observability requirements.

The mapping boundary is intentionally omitted here. Decide in `boundaries-and-models.md` whether Data Sources return DTOs, persistence models, or application models.

### Create a Repository when it hides data policy

A Repository is useful when it owns:

* remote and local coordination;
* caching and invalidation;
* offline fallback;
* persistence policy;
* synchronization;
* freshness rules;
* conflict resolution;
* deduplication;
* merging several mechanisms;
* hiding a complex persistence or SDK integration.

DTO mapping may happen inside a Repository boundary, but mapping alone does not justify a Repository.

Simple local mapping does not by itself justify a separate layer or mapper type.

### A Repository is not required for all data access

A concrete client or service may be sufficient when:

* there is only one data mechanism;
* the API is already application-oriented;
* no cache, fallback, persistence, or synchronization policy exists;
* no meaningful dependency boundary needs inversion;
* the Repository would only rename another method.

Example of unnecessary indirection:

```swift
struct ProductRepository {
    let client: ProductAPIClient

    func products() async throws -> [ProductDTO] {
        try await client.products()
    }
}
```

If the Repository adds no policy, semantic mapping, lifecycle, or boundary value, inject the client directly or expose a more application-oriented service.

### Repository API design

Prefer application or domain language:

```swift
struct ProductRepository {
    func products(
        matching query: ProductQuery
    ) async throws -> ProductPage

    func product(
        id: Product.ID
    ) async throws -> Product
}
```

Avoid transport-oriented APIs leaking into consumers:

```swift
func performGetRequest(
    path: String,
    headers: [String: String],
    page: Int
) async throws -> ProductResponseDTO
```

Translate errors at a boundary only when the higher layer needs a more stable or application-relevant meaning.

Do not wrap every low-level error mechanically.

Do not create:

* one Repository per endpoint;
* one global Repository for unrelated domains;
* a generic CRUD Repository that erases domain language;
* a Repository that only wraps one client method;
* storage-specific APIs when the caller should not care about storage.

## Repository protocols

Repositories do not need protocols by default.

Start with a concrete type:

```swift
struct ProductRepository {
    let apiClient: ProductAPIClient
    let cache: ProductCache
}
```

Initializer injection, previews, and tests can work without a protocol.

### Introduce a Repository protocol when justified

A protocol is useful when:

* an inner module must define a capability implemented by an outer module;
* multiple concrete implementations already exist;
* a second implementation is planned from known requirements;
* independently evolving modules need a stable boundary;
* the protocol is a deliberate public or integration contract;
* substitution cannot be represented more simply with values, closures, or configuration;
* a system or SDK boundary requires module inversion or genuinely different implementations.

The presence of a third-party SDK alone does not require a protocol. A concrete adapter may be sufficient.

Example:

```swift
protocol ProductProviding: Sendable {
    func products(
        matching query: ProductQuery
    ) async throws -> ProductPage
}
```

Prefer capability-oriented names when they remain clear:

```text
ProductProviding
CartStoring
SessionRefreshing
AnalyticsTracking
```

Avoid mechanically appending `Protocol` when the name does not describe the capability.

Do not rename existing protocols only to remove a `Protocol` suffix unless the new name improves meaning or consistency.

Do not split one coherent capability into many tiny protocols unless consumers genuinely need different subsets.

### Protocol ownership

Prefer defining the protocol in the module that consumes and owns the required capability.

Example:

```text
ProductFeature
└── ProductProviding

ProductData
└── LiveProductRepository: ProductProviding
```

Place the protocol in a shared module only when it is intentionally a shared or public contract.

Do not move every protocol into a generic `Interfaces` module.

### Do not create protocols only for mocks

Before adding a protocol for testing, consider:

* a concrete in-memory implementation;
* a closure;
* a value containing a small set of operation closures;
* initializer configuration;
* a deterministic fake client;
* testing the concrete type directly.

A protocol is one possible test seam, not the default testability mechanism.

Do not replace one coherent capability protocol with a large bag of unrelated closures. Use closures for small, focused seams.

## Services

`Service` is a broad name. Use it only when the responsibility is clear.

A service may represent:

* an application capability;
* a stateless operation;
* an adapter around a system API;
* an integration with an external SDK;
* a technical capability shared by several features.

Examples:

```text
ImageLoadingService
NotificationSchedulingService
LocationAuthorizationService
PaymentAuthorizationService
```

Prefer a more specific name when possible:

```text
ImageLoader
NotificationScheduler
LocationAuthorizer
PaymentAuthorizer
SessionRefresher
```

### Service versus Use Case

Use a Use Case when the type represents an application action and owns policy.

Use a service when the type exposes a reusable capability or technical operation.

Example:

```text
SubmitOrder              → Use Case
PaymentAuthorizer        → service capability
OrderRepository          → data policy
PaymentSDKAdapter        → infrastructure adapter
```

### Service versus Repository

Use a Repository when the responsibility is primarily data access, persistence, synchronization, or data-source policy.

Use a service when the responsibility is an operation that is not naturally modeled as storage or retrieval.

Do not rename every Repository to `Service`, or every service to `Repository`. Choose the name from responsibility, not architectural fashion.

## Data Sources

A Data Source is a low-level access mechanism used by a higher-level component, typically a Repository or service.

Typical examples include:

```text
ProductRemoteDataSource
ProductLocalDataSource
ProductCache
KeychainSessionStore
SwiftDataProductStore
```

A Data Source may know about:

* endpoints;
* HTTP methods;
* DTOs;
* database queries;
* persistence entities;
* file paths;
* Keychain keys;
* SDK-specific types.

Consumers above the Data layer should normally not depend on those details.

### Create Data Sources when the separation is meaningful

Data Sources are useful when:

* a Repository coordinates remote and local mechanisms;
* storage mechanisms have different ownership or lifetimes;
* a mechanism needs independent replacement;
* infrastructure-specific details must be contained;
* the Repository would otherwise contain unrelated low-level code;
* independent testing of the mechanism provides real value.

### Avoid pass-through Data Sources

Avoid chains such as:

```text
Repository
  ↓
RemoteDataSource
  ↓
APIClient
```

when every layer exposes the same operation and none owns mapping, policy, lifecycle, isolation, or error translation.

A Repository may depend directly on a concrete API client when no separate Data Source responsibility exists.

## Thin pass-through layers

A thin layer is not automatically wrong.

Keep it when it provides at least one clear benefit:

* a stable application API;
* a real module boundary;
* lifecycle ownership;
* actor isolation;
* error translation;
* semantic mapping that changes meaning, validates data, isolates an external model, or is reused;
* authorization or validation;
* cross-cutting logging or observability policy consistently owned by that boundary;
* future variation already known from current requirements.

Remove it when it only:

* renames the same operation;
* forwards identical parameters and results;
* exists to satisfy a diagram;
* duplicates another abstraction;
* creates a mock target without improving design;
* hides where concrete work actually happens.

When reviewing an existing layer, ask:

1. What policy does this type own?
2. What change can happen behind this boundary?
3. Which consumer is protected by it?
4. What becomes harder if the type is removed?
5. Is the benefit greater than the indirection?

If these questions have no concrete answer, simplify the chain.

## Testing without protocol-first design

Concrete-first design can remain testable.

Example with a configurable concrete client:

```swift
struct ProductAPIClient {
    let fetchProducts: @Sendable () async throws -> [ProductDTO]

    func products() async throws -> [ProductDTO] {
        try await fetchProducts()
    }
}
```

Test configuration:

```swift
let client = ProductAPIClient(
    fetchProducts: {
        [
            ProductDTO(
                id: "1",
                title: "Keyboard"
            )
        ]
    }
)
```

Other useful seams include:

* in-memory stores;
* fixture-backed clients;
* temporary databases;
* injected clocks or identifier generators;
* deterministic concrete fakes.

Use a protocol when it improves the actual boundary, not merely because a test needs control over one value.

## Review checklist

Before introducing or preserving a layer, verify:

* it owns policy, mapping, lifecycle, isolation, or a meaningful boundary;
* it does more than forward equivalent inputs and outputs;
* the abstraction is justified by current requirements;
* a concrete type, value, closure, or configuration would not be simpler;
* its name describes an application action or capability;
* its removal would lose something meaningful;
* its ownership and lifetime are clear;
* the resulting composition remains easier to understand than the code it replaces.

For each component, confirm:

* a Use Case owns application policy, coordination, reuse, or a deliberate operation boundary;
* a Repository hides meaningful data-access policy;
* a Repository protocol exists only for a real substitution, integration, or module-boundary need;
* a service has a specific reusable responsibility;
* a Data Source isolates a real mechanism rather than adding pass-through indirection.

Prefer the simplest composition that preserves the required boundaries.

```
```
