# Dependency Injection in SwiftUI

SwiftUI has real, compile-time dependency injection already: `@Environment` with an `EnvironmentKey`. It is hierarchical (inject at any point, override at any descendant), preview-friendly (override a single value without building a whole test app), and it works because the consuming view is a `DynamicProperty`-context struct.

You do not need a DI container. You do not need a service locator. You do not need a protocol for every dependency.

## Three DI patterns, in order of preference

### 1. `@Environment` + `EnvironmentKey` (cross-cutting, shared)

Use for: network client, auth session, analytics, logger, date provider, feature flags — anything many views in a subtree need.

```swift
// 1. Define the shape as a value (capability struct) or protocol.
struct Analytics {
    var track: (_ event: String, _ props: [String: Any]) -> Void

    static let noop = Analytics(track: { _, _ in })
}

// 2. Environment key with a safe default.
private struct AnalyticsKey: EnvironmentKey {
    static let defaultValue: Analytics = .noop
}

extension EnvironmentValues {
    var analytics: Analytics {
        get { self[AnalyticsKey.self] }
        set { self[AnalyticsKey.self] = newValue }
    }
}

// 3. Inject at the composition root.
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            RootView().environment(\.analytics, .live)
        }
    }
}

// 4. Consume wherever needed.
struct CheckoutButton: View {
    @Environment(\.analytics) private var analytics
    var body: some View {
        Button("Buy") {
            analytics.track("buy_tapped", [:])
        }
    }
}

// 5. Previews override trivially.
#Preview {
    CheckoutButton()
        .environment(\.analytics, Analytics(track: { e, _ in print("preview:", e) }))
}
```

The default value is important — it lets the view work in any preview or test without scaffolding, and it documents the safe fallback.

### 2. Initializer injection (single-screen or local)

Use for: dependencies scoped to one screen, or when you want the dependency to be explicit in the API.

```swift
struct ProductDetailView: View {
    let productID: ProductID
    let loadProduct: (ProductID) async throws -> Product

    @State private var product: Product?

    var body: some View {
        Group { if let p = product { ProductBody(product: p) } else { ProgressView() } }
            .task { product = try? await loadProduct(productID) }
    }
}

// Real call site
ProductDetailView(productID: id, loadProduct: liveAPI.fetchProduct)

// Preview
ProductDetailView(productID: .sample, loadProduct: { _ in .preview })
```

The dependency is just a closure — a value. No protocol, no class, no mock framework.

### 3. Custom `DynamicProperty` wrapper (when you need graph integration)

Use for: a dependency that must observe external state, persist something, or manage resources — where a simple `Environment` value is not enough. See `custom-dynamic-properties.md`.

## Do NOT do this

### ❌ Property-wrapped dependencies inside a class

```swift
// BROKEN
final class MyViewModel: ObservableObject {
    @Environment(\.analytics) var analytics   // silently does nothing
    @AppStorage("name") var name = ""          // does not persist
}
```

`@Environment`, `@AppStorage`, `@Query`, `@FocusState`, `@SceneStorage`, `@FetchRequest` are all `DynamicProperty`s. They resolve only inside a `View`, `App`, `Scene`, `ViewModifier`, or another `DynamicProperty`. Inside a class they are inert. This is the single most common bug in MVVM-in-SwiftUI code and it is not a SwiftUI limitation — it is a direct consequence of putting state where it does not belong.

### ❌ Service locator / singleton

```swift
// BROKEN in principle — untestable, no override points
final class Services {
    static let shared = Services()
    let analytics = Analytics.live
    let api = API.live
}
```

Compile-time injection is strictly better: override at any level of the tree, preview-friendly, no shared mutable state.

### ❌ Protocol per dependency by default

```swift
// Overkill for most cases
protocol AnalyticsServicing { func track(_ event: String, props: [String: Any]) }
final class LiveAnalytics: AnalyticsServicing { ... }
final class MockAnalytics: AnalyticsServicing { ... }
```

A struct of closures gives you the same substitutability with less code and better composition:

```swift
struct Analytics {
    var track: (_ event: String, _ props: [String: Any]) -> Void
}

extension Analytics {
    static let live = Analytics { event, props in /* real */ }
    static let noop = Analytics { _, _ in }
    static func recording(into sink: Recorder) -> Analytics { ... }  // for tests
}
```

Reserve protocols for:
- Objective-C interop (where closure-structs do not bridge).
- Genuine polymorphism with multiple live implementations that share behavior.
- API surfaces exported from a library where callers need to provide their own impl and a struct-of-closures would be awkward.

## Composition root

Every app has a composition root — usually `@main struct MyApp` — where all the real dependencies are wired up once. Keep it flat and explicit.

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(\.analytics, .live)
                .environment(\.api, .live(baseURL: .production))
                .environment(\.featureFlags, .live)
        }
    }
}
```

For complex wiring, group into a single `AppEnvironment` struct and inject it in one call — but do not build a DI framework.

## Preview scaffolding

A well-structured SwiftUI codebase gets previews *for free* because every dependency is either:

- A value with a safe default (`Environment` with `defaultValue`).
- A closure passed in as a parameter.
- A `@Binding` supplied from the preview scope.

If a preview requires setting up a class graph, that is a smell — the view is coupled to an implementation instead of an abstraction.
