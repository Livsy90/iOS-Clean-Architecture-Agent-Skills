# Composition Root, Dependency Lifetimes, and Feature Assembly

Use this reference when building application containers, feature factories, navigation composition, dependency lifetimes, SwiftUI previews, test composition, or session-scoped dependency graphs.

## Contents

- [Core model](#core-model)
- [Composition root](#composition-root)
- [Feature factories](#feature-factories)
- [Dependency lifetimes](#dependency-lifetimes)
- [Session-scoped graphs](#session-scoped-graphs)
- [Navigation composition](#navigation-composition)
- [SwiftUI composition](#swiftui-composition)
- [Preview and test composition](#preview-and-test-composition)
- [Common mistakes](#common-mistakes)
- [Review checklist](#review-checklist)

## Core model

Composition is the process of creating concrete dependencies and connecting them into an executable object graph.

A composition root is the outermost place where the application chooses:

- concrete implementations;
- dependency configuration;
- ownership and lifetimes;
- environment-specific behavior;
- feature assembly;
- navigation entry points.

Business and presentation types should receive dependencies rather than construct them.

Prefer:

```text
Composition
  ├── creates infrastructure
  ├── creates Repository or service
  ├── creates Use Case when needed
  ├── creates ViewModel
  └── creates View or flow entry point
````

Avoid:

```text
ViewModel
  └── creates Repository
        └── creates API client
```

Dependency injection does not require protocols, a framework, or a global container. Initializer injection and ordinary Swift construction are sufficient for many applications.

Prefer immutable dependency storage. Choose `struct`, `final class`, or `actor` from ownership, identity, mutability, and isolation requirements.

Do not make the entire dependency graph an actor merely because some dependency owns mutable state. Prefer a dedicated actor for that state when possible.

## Composition root

The application composition root usually lives near:

* the application entry point;
* `AppDelegate` or scene setup;
* a root coordinator;
* a dedicated application assembly;
* an object created by the SwiftUI `App`.

Example:

```swift
@main
struct ShopApp: App {
    private let composition = AppComposition.live(
        configuration: .production
    )

    var body: some Scene {
        WindowGroup {
            composition.makeRootView()
        }
    }
}
```

```swift
struct AppComposition {
    let productRepository: ProductRepository
    let cartRepository: CartRepository

    static func live(
        configuration: AppConfiguration
    ) -> AppComposition {
        let apiClient = APIClient(
            baseURL: configuration.apiBaseURL
        )

        let database = AppDatabase(
            configuration: configuration.database
        )

        return AppComposition(
            productRepository: ProductRepository(
                apiClient: apiClient,
                database: database
            ),
            cartRepository: CartRepository(
                apiClient: apiClient,
                database: database
            )
        )
    }

    @MainActor
    func makeRootView() -> some View {
        RootView(
            productsFactory: ProductsFeatureFactory(
                repository: productRepository
            ),
            cartFactory: CartFeatureFactory(
                repository: cartRepository
            )
        )
    }
}
```

Do not isolate the entire composition graph to `MainActor` by default.

Isolate:

* UI construction;
* UI-owned mutable state;
* dependencies that explicitly require main-actor access.

Keep non-UI infrastructure off `MainActor` unless its API or ownership requires otherwise.

### Composition root responsibilities

The composition root may:

* create application-scoped infrastructure;
* choose live implementations;
* configure shared dependencies;
* create or replace a session graph;
* delegate feature-specific assembly;
* provide root navigation composition.

It should not:

* contain business rules;
* become global mutable application state;
* expose arbitrary lookup by type;
* own every screen ViewModel;
* initialize every feature eagerly without a reason.

### Eager versus lazy construction

Choose eager or lazy construction deliberately.

Create lightweight, launch-critical dependencies eagerly.

Defer expensive work such as:

* database warmup;
* SDK initialization;
* large caches;
* feature graphs;
* background synchronization;
* expensive adapters;

until required, unless the first useful screen depends on them.

A long lifetime does not imply eager construction.

Keep environment configuration explicit. Do not scatter production, debug, preview, and test decisions across feature factories.

## Feature factories

A feature factory assembles dependencies owned by one feature or flow.

It receives shared dependencies from an outer composition point and creates feature-scoped or screen-scoped objects.

Example:

```swift
@MainActor
struct ProductsFeatureFactory {
    let repository: ProductRepository
    let imageLoader: ImageLoader

    func makeProductList(
        onSelect: @escaping (Product.ID) -> Void
    ) -> ProductListView {
        let viewModel = ProductListViewModel(
            repository: repository,
            onSelect: onSelect
        )

        return ProductListView(
            viewModel: viewModel,
            imageLoader: imageLoader
        )
    }

    func makeProductDetails(
        id: Product.ID
    ) -> ProductDetailsView {
        let loadProduct = LoadProduct(
            repository: repository
        )

        let viewModel = ProductDetailsViewModel(
            productID: id,
            loadProduct: loadProduct
        )

        return ProductDetailsView(
            viewModel: viewModel,
            imageLoader: imageLoader
        )
    }
}
```

A feature factory is useful when it:

* builds several related screens;
* controls feature-scoped dependencies;
* centralizes flow or navigation construction;
* hides feature internals;
* provides preview or test assembly;
* makes implementation or lifetime decisions.

Do not introduce a factory that only mirrors one initializer and controls no meaningful decision.

Pass factories or coordinators to navigation composition points, not arbitrary leaf Views.

Leaf Views should receive only the state and dependencies they require.

## Dependency lifetimes

Choose lifetime from ownership and behavior, not convenience.

| Lifetime           | Typical examples                                      |
| ------------------ | ----------------------------------------------------- |
| Application-scoped | API transport, database container, image pipeline     |
| Session-scoped     | authenticated state, account repositories, user cache |
| Feature-scoped     | checkout state, editor draft, flow coordinator        |
| Screen-scoped      | ViewModel, screen state, pagination state             |
| Operation-scoped   | request, transaction context, lightweight Use Case    |

Key rules:

* Do not make every dependency application-scoped.
* Do not recreate expensive shared infrastructure per screen.
* Do not retain screen ViewModels for the entire application.
* Release feature-scoped state when the flow ends.
* Treat Use Cases as lightweight values unless they explicitly own long-lived state.

The application composition root should own or delegate ownership of long-lived dependencies.

Navigation or the view hierarchy should normally own screen-scoped dependencies.

## Session-scoped graphs

Authenticated applications often need dependencies whose state belongs to the current user or account.

Keep them separate from application-scoped infrastructure:

```text
Application graph
├── transport
├── database infrastructure
├── logging
└── Session graph
    ├── authenticated client
    ├── account repositories
    ├── user cache
    └── user features
```

Example:

```swift
struct SessionComposition {
    let userID: User.ID
    let profileRepository: ProfileRepository
    let orderRepository: OrderRepository
    let cartRepository: CartRepository
}
```

Create the session graph after authentication.

Release or replace it on:

* logout;
* account switching;
* credential invalidation;
* session expiration requiring a clean restart.

Do not keep user-specific mutable state in an application singleton after logout.

When replacing a session graph, review:

* active tasks;
* `AsyncSequence` consumers;
* notifications and observers;
* subscriptions;
* WebSocket or SDK sessions;
* cached credentials;
* database contexts;
* feature ViewModels;
* navigation state.

If a graph owns tasks, observers, streams, or external sessions, define who cancels or invalidates them.

Do not rely only on incidental deallocation when deterministic teardown is required.

## Navigation composition

Navigation determines when screens and their dependencies are created.

A coordinator or router may:

* own navigation state;
* interpret routes;
* request screens from feature factories;
* retain feature-scoped composition;
* release flow graphs when navigation ends.

Example:

```swift
@MainActor
final class ProductsCoordinator {
    private let factory: ProductsFeatureFactory

    init(factory: ProductsFeatureFactory) {
        self.factory = factory
    }

    func makeList() -> ProductListView {
        factory.makeProductList { [weak self] productID in
            self?.showProduct(id: productID)
        }
    }

    private func showProduct(id: Product.ID) {
        // Update navigation state.
    }
}
```

The parent flow should own the coordinator for the lifetime of that navigation flow.

Prefer passing route inputs explicitly:

```swift
factory.makeProductDetails(id: productID)
```

Do not make Views resolve dependencies from a global container when a route changes.

Navigation callbacks, route values, and coordinators do not require protocols by default.

Introduce an abstraction only for a real substitution, integration, or module-boundary need.

## SwiftUI composition

SwiftUI does not remove the need for explicit composition.

Create identity-sensitive and long-lived dependencies outside transient view evaluation.

Avoid:

```swift
var body: some View {
    ProductListView(
        viewModel: ProductListViewModel(
            repository: ProductRepository(
                apiClient: APIClient()
            )
        )
    )
}
```

Repeated evaluation must not recreate stateful dependencies unexpectedly.

Compose the screen before passing objects into the rendered hierarchy:

```swift
@MainActor
@Observable
final class ProductListViewModel {
    private let repository: ProductRepository

    init(repository: ProductRepository) {
        self.repository = repository
    }
}
```

```swift
@MainActor
struct ProductsScene: View {
    @State private var viewModel: ProductListViewModel

    init(repository: ProductRepository) {
        _viewModel = State(
            initialValue: ProductListViewModel(
                repository: repository
            )
        )
    }

    var body: some View {
        ProductListView(viewModel: viewModel)
    }
}
```

Use the property wrapper appropriate to the observation model and deployment target.

For `ObservableObject`, use the ownership mechanism appropriate to that model, such as `@StateObject`.

The architectural requirement is stable ownership, not a specific wrapper.

### Environment injection

Use the SwiftUI environment for intentional subtree context, such as:

* presentation context;
* theme;
* locale-related formatting;
* navigation context;
* a narrowly scoped capability intentionally shared by a subtree.

Do not use the environment merely because many screens need the same dependency.

Required business and data dependencies should usually remain explicit through initializer injection.

Do not turn the environment into a service locator.

## Preview and test composition

### SwiftUI previews

Build previews from the same feature entry point when practical, but provide preview-specific concrete dependencies.

A preview does not require protocols by default.

```swift
extension ProductsFeatureFactory {
    static func preview() -> ProductsFeatureFactory {
        ProductsFeatureFactory(
            repository: ProductRepository.preview(
                products: Product.samples
            ),
            imageLoader: ImageLoader.preview()
        )
    }
}
```

```swift
#Preview {
    ProductsFeatureFactory
        .preview()
        .makeProductList(onSelect: { _ in })
}
```

Preview dependencies should be:

* deterministic;
* fast;
* independent from production credentials;
* free from unintended persistent writes;
* configurable for loading, content, empty, and error states.

Do not let previews silently use live networking or production databases.

### Test composition

Tests should assemble only the graph required by the scenario.

```swift
@MainActor
func makeSubject(
    products: [Product] = Product.samples
) -> ProductListViewModel {
    let repository = ProductRepository.inMemory(
        products: products
    )

    return ProductListViewModel(
        repository: repository
    )
}
```

A test composition may replace:

* persistence with in-memory storage;
* networking with fixture-backed clients;
* clocks and identifiers with deterministic values;
* external SDK adapters with controlled implementations;
* production configuration with test configuration.

Test composition should verify important graph behavior, including:

* object lifetime;
* shared versus isolated state;
* session replacement;
* feature teardown;
* concrete implementation selection.

Do not create a mock for every type merely because the graph is assembled in a test.

Integration tests may use most of the live graph while replacing only unsafe or nondeterministic edges.

## Common mistakes

### Hidden construction

Avoid:

```swift
final class ProductListViewModel {
    private let repository = ProductRepository.live()
}
```

The consumer should not choose its own infrastructure implementation or lifetime.

### Service locator

Do not introduce global lookup as a substitute for explicit composition:

```swift
let repository: ProductRepository =
    Dependencies.shared.resolve()
```

For legacy service-locator code, migrate incrementally by moving required dependencies into initializers at feature boundaries.

### God container

Do not pass the entire application container into every feature:

```swift
ProductListView(container: appContainer)
```

Pass only the dependencies required by the feature or screen.

### Lifetime mismatch

Avoid:

* creating a database per screen;
* retaining screen state for the whole application;
* sharing user state across sessions;
* recreating caches that should persist;
* keeping flow state after the flow ends.

### Eager initialization

Do not create every SDK, repository, cache, and feature graph during launch merely because the composition root knows about them.

### Environment as registry

Do not use SwiftUI environment values as a general dependency registry.

### Main-actor over-isolation

Do not mark the entire composition graph `@MainActor` by default.

Keep UI construction and UI-owned state on `MainActor`; keep independent infrastructure appropriately isolated.

## Review checklist

Before finalizing composition, verify:

* concrete implementations are selected in an explicit outer composition point;
* consumers do not construct required infrastructure;
* shared dependencies have deliberate owners;
* application, session, feature, screen, and operation lifetimes are not mixed accidentally;
* expensive dependencies are created eagerly only when justified;
* session and feature graphs have deterministic teardown when required;
* feature factories make real composition decisions;
* navigation receives route inputs explicitly;
* SwiftUI evaluation does not recreate identity-sensitive dependencies;
* required dependencies are explicit rather than hidden in global lookup or environment;
* previews use deterministic non-production dependencies;
* tests assemble focused graphs without mocking every type;
* protocols exist only for current substitution, integration, or module-boundary needs;
* the composition layer contains no business logic;
* the final graph is no more complex than the application requires.
