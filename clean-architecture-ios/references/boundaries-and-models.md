# Boundaries, Models, and Swift Modules

Use this reference when separating DTOs, persistence models, domain models, view data, external SDKs, logical layers, or Swift module boundaries.

## Contents

- [Core model](#core-model)
- [Choosing model boundaries](#choosing-model-boundaries)
- [Transport DTOs](#transport-dtos)
- [Persistence models](#persistence-models)
- [Domain and application models](#domain-and-application-models)
- [View data](#view-data)
- [Mapping boundaries](#mapping-boundaries)
- [External SDK boundaries](#external-sdk-boundaries)
- [Logical and physical boundaries](#logical-and-physical-boundaries)
- [Swift module design](#swift-module-design)
- [Common mistakes](#common-mistakes)
- [Review checklist](#review-checklist)

## Core model

A boundary separates responsibilities that change for different reasons.

Typical representations may include:

```text
External response
    ↓
Transport DTO
    ↓
Domain or application model
    ↓
View data
````

Persistence may introduce another representation:

```text
Persistence model
    ↕
Domain or application model
```

This is not a mandatory pipeline.

Use separate models only when crossing the boundary changes one or more of these:

* semantics;
* invariants;
* ownership;
* lifecycle;
* mutability;
* identity;
* optionality;
* serialization format;
* framework coupling;
* visibility;
* error meaning;
* concurrency requirements.

Do not create one model per architectural layer by convention.

A single model may cross several logical layers when it already has the correct meaning and does not expose unwanted transport, persistence, SDK, or UI concerns.

## Choosing model boundaries

Before introducing another representation, ask:

1. Does the current type expose transport, persistence, SDK, or UI details?
2. Do both sides interpret the data in the same way?
3. Does one side require stronger invariants?
4. Can either side change independently?
5. Are ownership, lifecycle, identity, or mutability different?
6. Would sharing the type create an unwanted framework or module dependency?
7. Does the protected boundary justify the mapping cost?

Create a separate representation when those questions reveal a meaningful difference.

Keep one representation when:

* fields have the same semantics;
* optionality means the same thing;
* no external framework leaks through the type;
* ownership and lifecycle are compatible;
* independent evolution is unlikely;
* mapping would only copy identical values;
* the shared type is an intentional application-owned contract.

Prefer the fewest representations that preserve correct semantics.

## Transport DTOs

A DTO represents an external transport format.

It may contain:

* backend field names;
* transport-specific optionality;
* versioned payload fields;
* nested response envelopes;
* pagination metadata;
* string-backed statuses;
* transport identifiers;
* encoded timestamps;
* compatibility fields;
* fields unused by the application.

Example:

```swift
struct ProductDTO: Decodable {
    let id: String
    let displayName: String
    let priceAmount: Decimal?
    let currencyCode: String?
    let stockStatus: String
}
```

Keep DTOs near the networking boundary.

Do not expose a DTO directly to Presentation when:

* backend naming leaks into UI code;
* invalid field combinations are representable;
* missing values require application decisions;
* transport enums may contain unknown values;
* the payload may evolve independently;
* consumers would need to understand decoding rules.

A separate application model may be unnecessary when the transport type is:

* application-owned;
* intentionally shared;
* semantically correct for consumers;
* free from backend-specific naming and invalid states;
* stable as an explicit contract rather than merely coincidentally identical.

Do not use a server-controlled DTO as an application-wide model only because its fields currently match the UI.

### DTO decoding

Keep decoding concerns at the transport boundary:

```swift
struct ProductDTO: Decodable {
    enum CodingKeys: String, CodingKey {
        case id
        case displayName = "display_name"
        case stockStatus = "stock_status"
    }

    let id: String
    let displayName: String
    let stockStatus: String
}
```

Do not add `Codable` to internal models merely because the network uses JSON.

Use `Decodable` when only decoding is required and `Encodable` when only encoding is required.

## Persistence models

A persistence model represents storage requirements.

It may contain:

* framework annotations;
* relationships;
* delete rules;
* migration fields;
* storage identifiers;
* denormalized values;
* indexing properties;
* framework-managed reference identity;
* context-bound lifecycle.

Example:

```swift
@Model
final class StoredProduct {
    @Attribute(.unique)
    var id: String

    var title: String
    var updatedAt: Date

    init(
        id: String,
        title: String,
        updatedAt: Date
    ) {
        self.id = id
        self.title = title
        self.updatedAt = updatedAt
    }
}
```

A SwiftData or Core Data model may be used directly when:

* persistence is central to the feature;
* framework coupling is acceptable;
* no separate domain semantics are required;
* reference identity and mutation behavior are appropriate;
* all consumers belong to the persistence-aware boundary.

Create a separate application model when:

* persistence annotations would leak into inner modules;
* storage objects have context-bound lifecycle;
* the application requires immutable values;
* storage schema differs from product meaning;
* migration-only fields should remain hidden;
* the model must cross module or concurrency boundaries independently.

Do not pass context-bound persistence objects across actor or concurrency boundaries unless the persistence framework explicitly supports that usage.

Prefer stable identifiers or detached application values when crossing those boundaries.

Do not create copies of persistence models when the copies add no semantic, ownership, or isolation value.

## Domain and application models

A domain or application model represents meaning required by the product or use case.

It should use application language rather than transport or storage language.

Example:

```swift
struct Product: Identifiable, Sendable, Equatable {
    struct ID: RawRepresentable, Sendable, Hashable {
        let rawValue: String
    }

    let id: ID
    let title: String
    let price: Money?
    let availability: Availability
}
```

A domain or application model may:

* enforce valid construction;
* use value objects;
* express state with enums;
* remove transport-specific optionality;
* preserve business identity;
* expose behavior related to its invariants.

Do not force rich domain entities into every application.

For data-driven features, immutable application values may be sufficient.

Do not automatically create separate `ApplicationModel` and `DomainModel` types. Separate Application and Domain only when their responsibilities are meaningfully different.

### Invariants

Strengthen invariants only when the application can define valid behavior.

For example, convert:

```text
currencyCode: String?
priceAmount: Decimal?
```

into:

```swift
struct Money: Sendable, Equatable {
    let amount: Decimal
    let currency: Currency
}
```

when the application requires those values to exist together.

Do not invent defaults merely to make domain construction succeed.

For incomplete or invalid external data, choose deliberately among:

* rejecting the payload;
* preserving optionality;
* representing an unavailable value;
* returning a partial result;
* applying a documented product fallback.

## View data

View data represents presentation-specific state or formatting.

It may contain:

* localized strings;
* formatted prices and dates;
* button enablement;
* accessibility labels;
* display ordering;
* stable section or row identity;
* placeholders;
* UI-specific grouping;
* presentation state.

Example:

```swift
struct ProductRowViewData: Identifiable, Equatable {
    let id: Product.ID
    let title: String
    let priceText: String
    let availabilityText: String
    let canAddToCart: Bool
}
```

Create view data when it:

* removes repeated formatting from views;
* combines several application values;
* creates a clear rendering contract;
* keeps UI concerns out of domain models;
* represents presentation-only state.

Do not create view data when a View can consume a small application model directly without semantic or formatting changes.

Preserve semantic identity from the underlying model whenever possible.

Do not generate a new random identifier during each mapping pass, because unstable identity breaks diffing and view state.

Avoid using `ViewModel` for both:

* a state-owning observable presentation object;
* an immutable row or screen data value.

Prefer distinct names such as:

```text
ProductListViewModel
ProductRowViewData
```

## Mapping boundaries

Map where semantics, ownership, or representation changes.

Typical locations:

```text
Networking boundary:
DTO → application model

Persistence boundary:
stored model ↔ application model

Presentation boundary:
application model → view data
```

Keep mapping in the outer module that knows both representations.

When DTO and domain types live in separate modules:

```text
Data or Networking → Domain
```

Do not make the Domain module import transport types.

Place DTO-to-domain mapping in the Data or Networking target, not in the Domain target.

Prefer simple mapping near the transport boundary:

```swift
// Data or Networking module

extension ProductDTO {
    func toDomain() throws -> Product {
        guard let availability = Availability(
            rawValue: stockStatus
        ) else {
            throw ProductMappingError.invalidAvailability
        }

        let price: Money?

        switch (priceAmount, currencyCode) {
        case let (.some(amount), .some(code)):
            price = try Money(
                amount: amount,
                currencyCode: code
            )

        case (nil, nil):
            price = nil

        default:
            throw ProductMappingError.incompletePrice
        }

        return Product(
            id: Product.ID(rawValue: id),
            title: displayName,
            price: price,
            availability: availability
        )
    }
}
```

This keeps the transport dependency outside Domain and makes missing or invalid values explicit.

Mapping-specific errors normally belong to the outer mapping boundary.

Convert them into a stable application error only when a consumer needs that distinction.

Use a dedicated mapper when transformation is:

* complex;
* reused;
* configurable;
* independently tested;
* dependent on policy;
* owned by a clear boundary.

Do not create a mapper type for every one-to-one property copy.

### Mapping failures

Distinguish failure meaning:

* decoding failure: the external representation is unreadable;
* mapping failure: decoded data cannot form the required application model;
* validation failure: a requested operation violates an application rule.

Translate an error only when the higher layer needs a more stable or application-relevant meaning.

Do not wrap every error mechanically.

## External SDK boundaries

Do not let third-party SDK types spread through application code without evaluating the coupling.

A concrete adapter is often sufficient:

```swift
struct AnalyticsTracker {
    let sdk: ExternalAnalyticsSDK

    func track(_ event: AnalyticsEvent) {
        sdk.send(
            name: event.name,
            parameters: event.parameters
        )
    }
}
```

An adapter may protect the application from:

* SDK-specific event types;
* callback APIs;
* global singletons;
* initialization details;
* vendor naming;
* unstable error types;
* platform availability;
* replacement cost.

The presence of an SDK does not automatically justify a protocol.

Introduce a protocol only when a current substitution, integration, ownership, or module-boundary requirement needs one.

Prefer application-owned input and output types at the adapter boundary.

Do not expose SDK models merely to avoid a small conversion.

Choose the adapter's type and isolation from the SDK's ownership and concurrency behavior.

A `struct` wrapper does not make a mutable SDK value-semantic or `Sendable`.

## Logical and physical boundaries

A logical boundary is expressed through:

* responsibilities;
* files and namespaces;
* access control;
* dependency direction;
* explicit ownership.

A physical boundary is enforced through a separate Swift target, package, framework, or module.

Start with logical boundaries when they are sufficient.

Introduce a physical module when it provides a concrete benefit:

* compile-time dependency enforcement;
* controlled API visibility;
* independent ownership;
* reuse across products;
* isolated testing;
* team boundaries;
* replaceable implementations;
* build organization.

Do not create a package for every layer, model category, or feature type.

Physical modularization adds costs:

* public API design;
* manifest and target maintenance;
* dependency graph complexity;
* additional imports;
* build-system and cross-module compilation trade-offs;
* migration work;
* visibility decisions.

The benefit should exceed those costs.

## Swift module design

Keep module APIs small and intentional.

Prefer:

```text
ProductFeature
├── public feature entry point
├── public route inputs
└── internal Views, ViewModels, and helpers
```

Avoid making every model and initializer `public` merely so application composition can reach internal details.

Expose one intentional assembly entry point when useful:

```swift
public struct ProductsFeature {
    @MainActor
    public static func make(
        dependencies: Dependencies
    ) -> some View {
        // Internal SwiftUI assembly.
    }
}
```

Return `some View` only when SwiftUI is intentionally part of the feature's public boundary.

This is appropriate when:

* the feature is explicitly SwiftUI-based;
* consumers are expected to receive a ready-to-render SwiftUI screen;
* supporting UIKit or a UI-agnostic entry point is not a requirement;
* keeping SwiftUI internal provides no meaningful benefit.

Otherwise, expose a platform-appropriate entry point or keep UI assembly in a presentation-specific module.

For example, a module may expose:

* a ViewModel or state object;
* a coordinator or flow entry point;
* route values;
* a UIKit view controller factory;
* an application capability independent from UI.

A feature `Dependencies` value should:

* contain explicit fields;
* expose only the feature's intentional external contract;
* avoid low-level infrastructure when a narrower capability is sufficient;
* remain small enough to understand as a feature boundary.

Do not introduce a `Dependencies` value merely to avoid writing one or two explicit parameters.

Prefer:

```swift
ProductsFeature.make(
    repository: repository,
    imageLoader: imageLoader
)
```

when the dependency list is short and stable.

Use a feature `Dependencies` value when it forms a meaningful, coherent feature contract.

Do not turn it into a global or type-erased container.

### Dependency direction between modules

A common modular composition is:

```text
AppComposition
├── ProductFeature
├── ProductData
└── connects ProductData to the capability required by ProductFeature
```

When inversion is justified:

```text
ProductFeature owns ProductProviding
ProductData implements ProductProviding
AppComposition connects them
```

Do not add protocol-based inversion when a concrete dependency in one module is simpler and no boundary requires it.

### Shared models

Do not create a global `Models` module containing unrelated types.

A model belongs in a shared module only when:

* several consumers intentionally share the same semantics;
* ownership is clear;
* the model forms a stable contract;
* the module does not pull in unwanted frameworks.

Prefer domain- or capability-focused ownership over convenience-based sharing.

## Common mistakes

* Creating DTO, persistence, domain, and view models with identical fields by convention.
* Passing DTOs throughout the application because decoding already works.
* Making Domain depend on Data through DTO-aware initializers.
* Adding persistence annotations to otherwise framework-independent models without evaluating coupling.
* Passing context-bound persistence objects across unsupported concurrency boundaries.
* Putting localized UI strings into domain entities.
* Generating random view identity during mapping.
* Creating a mapper type for every trivial conversion.
* Inventing defaults that hide invalid external data.
* Exposing third-party SDK types across feature boundaries.
* Adding a protocol merely because an SDK adapter exists.
* Creating packages without a dependency-enforcement or ownership need.
* Making every type `public` to work around poor module composition.
* Using `some View` as a public feature contract without intentionally accepting SwiftUI coupling.
* Creating a `Dependencies` bag for every feature regardless of size.
* Building a shared module that becomes a dependency magnet.
* Treating physical modules as proof of Clean Architecture.

## Review checklist

Before adding or preserving a boundary, verify:

* the two sides differ in semantics, ownership, lifecycle, identity, or framework coupling;
* the additional model protects a real concern;
* mapping remains in the outer module that knows both representations;
* Domain does not depend on transport or persistence types;
* mapping-specific errors remain at the appropriate outer boundary;
* simple mapping remains local;
* invalid and incomplete data are handled explicitly;
* transport and persistence details do not leak unintentionally;
* context-bound storage models do not cross unsupported concurrency boundaries;
* view identity remains stable;
* UI formatting stays outside domain models unless it belongs to product semantics;
* external SDK types are contained where isolation matters;
* protocols exist only for current explicit needs;
* logical boundaries were considered before new Swift modules;
* module APIs expose only intentional contracts;
* `some View` is public only when SwiftUI is an intentional module boundary;
* feature dependency bags are justified by a meaningful contract;
* shared models have clear ownership and genuinely shared semantics;
* the final design contains no more models, mappers, adapters, or modules than the problem requires.
